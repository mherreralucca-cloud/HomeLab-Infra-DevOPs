# Etapa 2.C — Arquitectura DNS Recursiva Aislada (Unbound + Pi-hole) y Despliegue GitOps con Docker y Portainer

En esta tercera parte de la Etapa 2 implementé la infraestructura DNS de mi red local dentro de la VM `docker-prod` (`192.168.1.102`). El foco del despliegue está en el diseño de red, el aislamiento de contenedores, la validación criptográfica **DNSSEC** con mi propio resolvedor recursivo (**Unbound**) y la automatización del ciclo de vida mediante un flujo **GitOps** (Docker Compose versionado en este repositorio y desplegado desde **Portainer** con secretos fuera de Git).

> **Nota de lectura:** Arranco directo con la topología de red, el despliegue paso a paso, las medidas de hardening y las pruebas de validación en consola. Si querés profundizar en los fundamentos (por qué DNS requiere TCP y UDP en el puerto 53 con `edns-buffer-size: 1232`, resolución recursiva frente a *forwarders*, validación DNSSEC con `root.key` y funcionamiento interno de IPAM y `healthcheck`), te dejé todo el **Marco Teórico desarrollado al final del documento**.

---

## Topología y Aislamiento de Red (`dns_net`)

En lugar de colgar los servicios sobre el puente por defecto `docker0` (`172.17.0.0/16`) —donde las IPs internas cambian según el orden de arranque—, declaré una red bridge dedicada (**`dns_net`**) con gestión **IPAM** en la subred **`172.23.0.0/24`**. Esto crea un switch virtual independiente en el kernel (`br-<id>` con gateway `172.23.0.1/24`) con IPs internas estáticas: **`172.23.0.2`** para Unbound y **`172.23.0.3`** para Pi-hole.

Un punto central de esta arquitectura es que **Unbound no tiene bloque `ports:` en el Compose**. No expone ningún socket hacia la VM `192.168.1.102` ni hacia la LAN: queda invisible para cualquier dispositivo de la casa y únicamente recibe tráfico desde Pi-hole por el enlace privado `172.23.0.3 → 172.23.0.2:5335`.

```text
Repositorio GitHub (refs/heads/main)
└── docker-stacks/pihole-unbound/docker-compose.yml ──► Consumido por Portainer CE (:9443)
                                                               │ (inyecta PIHOLE_PASSWORD)
                                                               ▼
VM 200: docker-prod (IP 192.168.1.102/24 — Firewall ufw LAN 192.168.1.0/24)
├── Persistencia en NVMe (/opt/stacks/dns-stack/)
│   ├── unbound/unbound.conf ──► Montaje de archivo único en solo lectura (:ro)
│   ├── etc-pihole/          ──► Persistencia de pihole-FTL.db y pihole.toml
│   └── etc-dnsmasq.d/       ──► Configuración adicional de dnsmasq
└── Red Bridge Dedicada: dns_net (172.23.0.0/24 — IPAM estático | br-<id>: 172.23.0.1)
    │
    ├── [pihole — 172.23.0.3] ──► Publica 53/TCP, 53/UDP y 8080:80/TCP en el host
    │        │
    │        │ Reenvío interno exclusivo (172.23.0.2#5335)
    │        ▼
    └── [unbound — 172.23.0.2:5335] ──► SIN puertos publicados hacia el host
             │
             └──► Salida directa a Internet (Resolución recursiva + DNSSEC):
                  Root Servers (".") ──► TLD (.com / .ar) ──► Servidor Autoritativo
```

```text
[PC / Celular en LAN (192.168.1.x)]
       │
       │ 1. Consulta DNS (Puerto 53 UDP/TCP hacia 192.168.1.102)
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ Red Privada Docker: dns_net (172.23.0.0/24)                             │
│                                                                         │
│  [Contenedor Pi-hole: 172.23.0.3 (Puerto 53)]                           │
│     ├── ¿Está en las listas de bloqueo? ──► Devuelve 0.0.0.0 (Fin)      │
│     ├── ¿Está guardado en su caché RAM? ──► Devuelve la IP en 0 ms (Fin)│
│     │                                                                   │
│     │ 2. Si no lo conoce, se lo pide a Unbound por la red interna       │
│     ▼    (hacia 172.23.0.2 en el puerto 5335)                           │
│  [Contenedor Unbound: 172.23.0.2 (Puerto 5335 - Sin puertos al host)]   │
└─────┼───────────────────────────────────────────────────────────────────┘
      │
      │ 3. Unbound sale directo a Internet y recorre el árbol jerárquico:
      ├──► Paso A: Consulta a los Root Servers (".") ──► "¿Quién maneja .ar?"
      ├──► Paso B: Consulta al servidor TLD ".ar" ─────► "¿Quién maneja .com.ar?"
      └──► Paso C: Consulta al servidor autoritativo ──► Obtiene la IP real,
                   valida DNSSEC y entrega la respuesta a Pi-hole.

```

La capa de entrada expone el puerto **53 tanto en UDP como en TCP** y fija **`edns-buffer-size: 1232`** en Unbound: el 99% de las consultas viajan por UDP sin la latencia del saludo de tres vías, el tope de 1232 bytes evita la fragmentación IP sobre enlaces Ethernet con MTU 1500, y `53/TCP` garantiza el reintento automático cuando una respuesta con firmas DNSSEC extensas supera ese umbral (`TC=1`).

---

## Implementación Paso a Paso

### Paso 0: Snapshot Preventivo en Proxmox VE

Antes de abrir el puerto 53 y modificar la infraestructura DNS, saqué un snapshot en Proxmox (`VM 200: docker-prod` → **Snapshots** → **Take Snapshot**) con el nombre **`pre-dns-stack`** para conservar intacto el estado limpio de Debian + Docker + Portainer.

---

### Paso 1: Pre-chequeo de Puertos y Configuración Endurecida de Unbound

Entré por SSH a `docker-prod` (`ssh lucca@192.168.1.102`) y confirmé primero que ningún servicio del sistema (como `systemd-resolved`) estuviera ocupando el puerto 53:

```bash
sudo ss -tulpn | grep :53
```

Con la salida vacía confirmada, creé los directorios de persistencia bajo `/opt/stacks/dns-stack/` y redacté `/opt/stacks/dns-stack/unbound/unbound.conf`:

```bash
sudo mkdir -p /opt/stacks/dns-stack/unbound
sudo mkdir -p /opt/stacks/dns-stack/etc-pihole
sudo mkdir -p /opt/stacks/dns-stack/etc-dnsmasq.d

sudo vim /opt/stacks/dns-stack/unbound/unbound.conf
```

```yaml
server:
    interface: 0.0.0.0
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no
    prefer-ip6: no

    # Rutas y privilegios dentro del contenedor mvance/unbound
    username: "_unbound"
    directory: "/opt/unbound/etc/unbound"
    chroot: ""
    auto-trust-anchor-file: "/opt/unbound/etc/unbound/var/root.key"

    # Hardening DNSSEC, privacidad y prevención de fragmentación
    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1232
    prefetch: yes
    num-threads: 2

    # Control de Acceso: solo consultas desde redes privadas (Docker/LAN) y localhost
    access-control: 127.0.0.1/32 allow
    access-control: 172.16.0.0/12 allow
    access-control: 192.168.1.0/24 allow
    access-control: 0.0.0.0/0 refuse

    # Protección contra DNS Rebinding (impide que dominios externos resuelvan IPs de tu LAN)
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8

    verbosity: 0
```

Por último fijé permisos `644` (`root` escribe, `_unbound` dentro del contenedor solo lee) y verifiqué que figurara como archivo regular (`-rw-r--r--`):

```bash
sudo chmod 644 /opt/stacks/dns-stack/unbound/unbound.conf
ls -la /opt/stacks/dns-stack/unbound/unbound.conf
```

---

### Paso 2: Declaración GitOps (`docker-compose.yml`) e Inyección de Secretos

Para evitar el *ClickOps* y mantener la infraestructura versionada sin exponer credenciales en un repositorio público, subí el archivo declarativo en `docker-stacks/pihole-unbound/docker-compose.yml`:

```yaml
name: dns-stack

services:
  unbound:
    image: mvance/unbound:latest
    container_name: unbound
    restart: unless-stopped
    hostname: unbound
    volumes:
      - /opt/stacks/dns-stack/unbound/unbound.conf:/opt/unbound/etc/unbound/unbound.conf:ro
    networks:
      dns_net:
        ipv4_address: 172.23.0.2
    healthcheck:
      test: ["CMD", "drill", "@127.0.0.1", "-p", "5335", "cloudflare.com"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
    security_opt:
      - no-new-privileges:true

  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    hostname: pihole-prod
    depends_on:
      unbound:
        condition: service_healthy
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"
    environment:
      TZ: 'America/Argentina/Mendoza'
      FTLCONF_webserver_api_password: '${PIHOLE_PASSWORD}'
      FTLCONF_dns_upstreams: '172.23.0.2#5335'
      FTLCONF_dns_listeningMode: 'all'
      FTLCONF_misc_etc_dnsmasq_d: 'true'
    volumes:
      - /opt/stacks/dns-stack/etc-pihole:/etc/pihole
      - /opt/stacks/dns-stack/etc-dnsmasq.d:/etc/dnsmasq.d
    cap_add:
      - NET_ADMIN
    networks:
      dns_net:
        ipv4_address: 172.23.0.3
    security_opt:
      - no-new-privileges:true

networks:
  dns_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.23.0.0/24
```

**Flujo de despliegue en Portainer (`https://192.168.1.102:9443`):**
1. En **local → Stacks → + Add stack**, seleccioné el método **`Repository`** apuntando a este repositorio público (`refs/heads/main`) con ruta de despliegue `docker-stacks/pihole-unbound/docker-compose.yml`.
2. En la sección **Environment variables**, cargué **`PIHOLE_PASSWORD`** antes de presionar **Deploy the stack**. Portainer guarda el secreto en su volumen local `portainer_data`, lo inyecta en memoria al sustituir `${PIHOLE_PASSWORD}` y nunca toca Git.
3. **Orquestación de arranque (`healthcheck` + `service_healthy`):** en lugar de usar un `depends_on` simple (que solo verifica que el proceso arrancó), definí un `healthcheck` con `drill` contra `127.0.0.1:5335` (con 15 s de `start_period` para inicializar `root.key`). Docker Compose retiene el arranque de `pihole` hasta que `unbound` reporta estado `healthy`.
4. **Reserva de puertos para la Etapa 3:** mapeé la interfaz web de Pi-hole al puerto `8080` (`8080:80/tcp`) para mantener libres los puertos `80` y `443` de la VM de cara al Proxy Inverso.

---

### Auditoría y chequeo en Consola (`ss`, `ip`, `dig` y `drill`)

Una vez activo el stack, validé el aislamiento de red, el rendimiento de caché y la validación criptográfica DNSSEC directamente desde la terminal SSH de `docker-prod`:

```bash
# 1. Verificar ambos contenedores en estado Up (healthy) y el switch virtual dns_net (172.23.0.1/24)
docker ps
ip -c a | grep -A 3 "172.23.0.1"

# 2. Validar exposición de puertos en el host (:53 y :8080 presentes; :5335 ausente)
sudo ss -tulpn | grep -E ':53|:8080|:5335'
```

La salida de `ss -tulpn` confirmó únicamente `:53` (TCP/UDP) y `:8080` (TCP) tomados por `docker-proxy`, sin rastro del puerto `:5335`, demostrando que Unbound quedó aislado dentro de `dns_net`.

Luego corrí las pruebas de resolución recursiva, caché RAM y DNSSEC:

```bash
# Prueba 1: Resolución recursiva en frío (Raíz -> TLD -> Autoritativo)
dig @127.0.0.1 clarin.com

# Prueba 2: Misma consulta segundos después (validación de caché RAM)
dig @127.0.0.1 clarin.com

# Prueba 3: Validación DNSSEC positiva auditando el bit "ad" directo en Unbound
docker exec unbound drill @127.0.0.1 -p 5335 sigok.verteiltesysteme.net | grep -E "rcode|flags"

# Prueba 4: Validación DNSSEC negativa (dominio con firma criptográfica rota a propósito)
dig @127.0.0.1 sigfail.verteiltesysteme.net
```

Resultados registrados en la auditoría:
* **Resolución en frío vs. caché (`clarin.com`):** la primera consulta devolvió `status: NOERROR`, `udp: 1232` (validando `edns-buffer-size`) y un `Query time` de **`368 msec`** tras recorrer el árbol completo. Al repetir la consulta segundos después, el TTL bajó y el tiempo cayó a **`0 msec`**, comprobando la respuesta directa desde RAM.
* **DNSSEC positivo (`sigok.verteiltesysteme.net`):** `drill` directo sobre Unbound devolvió `rcode: NOERROR` junto con el bit **`ad`** (*Authenticated Data*), confirmando la verificación de la cadena de firmas contra `root.key`.
* **DNSSEC negativo (`sigfail.verteiltesysteme.net`):** devolvió **`status: SERVFAIL`** y `ANSWER: 0`, certificando el descarte automático de respuestas con firmas falsificadas o corruptas.

---

## Capa Frontal: Pi-hole como DNS Sinkhole

En esta topología, **Pi-hole v6** actúa únicamente como el punto de entrada (*DNS Sinkhole*) de la LAN en el puerto 53: recibe las peticiones de los clientes de la red local, intercepta dominios de publicidad y telemetría devolviendo `0.0.0.0`, sirve desde su caché RAM las resoluciones recientes y delega el 100% de las consultas legítimas a Unbound por el enlace interno `172.23.0.2#5335` (configurado con variables nativas `FTLCONF_*`). Con el servidor ya en producción, el panel web (`http://192.168.1.102:8080/admin`) confirma los clientes activos en red y el porcentaje de consultas filtradas en tiempo real:

![Dashboard operativo de Pi-hole mostrando clientes activos y consultas filtradas](./img/pihole-dashboard.png)

---

## Troubleshooting Real

Durante el despliegue y las pruebas surgieron dos comportamientos concretos que dejé resueltos en consola:

* **Recarga de variables de entorno en Pi-hole v6 (`PIHOLE_PASSWORD`):** desplegar sin la variable cargada hace que Pi-hole v6 persista `pwhash = ""` en `/opt/stacks/dns-stack/etc-pihole/pihole.toml` dejando el panel sin contraseña. En Docker, hacer `Restart` no aplica variables de entorno nuevas: es obligatorio **recrear** el contenedor desde Portainer (`Update the stack` con *redeploy*) o definir el hash directamente con `docker exec -it pihole pihole setpassword` y validarlo con `grep "pwhash" /opt/stacks/dns-stack/etc-pihole/pihole.toml`.
* **Limpieza del bit `ad` entre Pi-hole y Unbound:** al ejecutar `dig @127.0.0.1 sigok.verteiltesysteme.net` pasando por Pi-hole (con su casilla propia de DNSSEC desactivada para no duplicar el trabajo de Unbound), Pi-hole responde `NOERROR` pero remueve el flag `ad` de la cabecera. La auditoría real del bit `ad` se debe hacer consultando directamente a Unbound (`docker exec unbound drill @127.0.0.1 -p 5335 sigok.verteiltesysteme.net`).

---

## Hoja de Trucos (Cheat Sheet)

| Categoría | Archivo / Comando / Directiva | Para qué sirve |
| :--- | :--- | :--- |
| **Persistencia Host** | `/opt/stacks/dns-stack/unbound/unbound.conf` | Configuración endurecida de Unbound montada como archivo único en `:ro`. |
| **GitOps Repo** | `docker-stacks/pihole-unbound/docker-compose.yml` | Definición declarativa del stack leída por Portainer desde `refs/heads/main`. |
| **Auditoría Puertos** | `sudo ss -tulpn \| grep -E ':53\|:8080\|:5335'` | Confirma `:53` y `:8080` activos en el host y `:5335` aislado dentro de `dns_net`. |
| **Auditoría Red** | `docker network inspect dns-stack_dns_net` | Inspecciona la subred `172.23.0.0/24` y las IPs IPAM `.2` y `.3`. |
| **Prueba Caché** | `dig @127.0.0.1 clarin.com` | Contrasta resolución recursiva en frío frente a caché RAM (`0 msec`). |
| **Prueba DNSSEC (+)** | `docker exec unbound drill @127.0.0.1 -p 5335 sigok.verteiltesysteme.net \| grep -E "rcode\|flags"` | Valida `NOERROR` + flag `ad` (*Authenticated Data*) directamente en Unbound. |
| **Prueba DNSSEC (-)** | `dig @127.0.0.1 sigfail.verteiltesysteme.net` | Comprueba el bloqueo de firmas corruptas devolviendo `status: SERVFAIL`. |
| **Hardening Unbound** | `private-address` + `access-control` + `auto-trust-anchor-file` | Mitiga *DNS Rebinding*, cierra *Open Resolver* y habilita DNSSEC. |
| **Hardening Docker** | `cap_add: [NET_ADMIN]` + `no-new-privileges:true` | Mínimo privilegio en kernel sin recurrir a `privileged: true`. |

---

## Marco Teórico: Protocolo DNS, Recursividad, DNSSEC y Orquestación GitOps

### 1. El protocolo DNS en Capas 4 y 7 (UDP/TCP 53 y `EDNS0`)
El protocolo **DNS** trabaja en la **Capa 7 (Aplicación)** traduciendo nombres de dominio a direcciones IP de Capa 3, pero necesita ambos protocolos de Capa 4 sobre el puerto 53:
* **UDP:** resuelve las consultas estándar en un único intercambio de paquetes sin la sobrecarga del *Three-Way Handshake* de TCP (`SYN` → `SYN-ACK` → `ACK`).
* **TCP y `edns-buffer-size: 1232`:** el límite histórico de DNS sobre UDP era de 512 bytes, insuficiente para transportar firmas criptográficas DNSSEC. La extensión **EDNS0** eleva ese tope a **1232 bytes** (valor recomendado por *DNS Flag Day* para evitar fragmentación IP en redes con MTU de 1500). Si una respuesta supera los 1232 bytes, el servidor responde con el bit de truncamiento (`TC=1`) y el cliente reintenta por **53/TCP**.

### 2. Reenviador (*Forwarder*) vs. Resolvedor Recursivo Propio (`Unbound`)
Un filtro DNS por sí solo actúa como **Forwarder**: ante cualquier dominio no bloqueado le reenvía la consulta a un DNS comercial (`8.8.8.8`, `1.1.1.1` o el del ISP), concentrando todo el historial de navegación en un tercero. Con **Unbound**, el servidor ejecuta la **resolución recursiva en escalera** sin intermediarios:
1. **Servidores Raíz (*Root Servers* `.`):** usando *Root Hints* (las IPs fijas de los 13 grupos raíz), Unbound consulta qué servidores administran la terminación del dominio (`.com`, `.ar`).
2. **Servidores TLD (*Top Level Domain*):** devuelven qué servidor de nombres es autoritativo para ese dominio.
3. **Servidor Autoritativo:** entrega la IP real, que Unbound guarda en caché RAM y renueva proactivamente antes de expirar gracias a `prefetch: yes`.

### 3. Validación Criptográfica DNSSEC (`RRSIG` y `root.key`)
Como el DNS original carece de autenticación, un atacante en tránsito puede inyectar respuestas UDP falsas (**Envenenamiento de Caché / *DNS Spoofing***). **DNSSEC** introduce firmas digitales (`RRSIG`) en cada nivel del árbol DNS. Unbound parte desde su llave pública raíz local (**`root.key`** / *Trust Anchor*) y valida matemáticamente toda la cadena hasta el registro final; ante cualquier firma alterada o faltante, descarta la respuesta devolviendo **`SERVFAIL`**.

### 4. Redes con IPAM, `healthcheck` y Despliegue GitOps en Docker
* **Aislamiento con IPAM:** declarar una red bridge propia (`dns_net: 172.23.0.0/24`) en `docker-compose.yml` crea un switch virtual dedicado en el kernel (`br-<id>`), habilita IPs estáticas internas y permite dejar servicios de backend (Unbound) sin ningún puerto publicado en el host.
* **Dependencias reales de arranque:** `depends_on` con `condition: service_healthy` acoplado a un `healthcheck` activo asegura que un contenedor dependiente (`pihole`) no inicie hasta que el backend (`unbound`) realmente responda consultas en su puerto interno.
* **Separación de código y secretos:** en el flujo GitOps, el archivo `docker-compose.yml` versionado en GitHub describe el estado deseado con marcadores (`${PIHOLE_PASSWORD}`), mientras que Portainer inyecta los valores sensibles en memoria desde su almacenamiento local al ordenar el despliegue a `dockerd` mediante `/var/run/docker.sock`.

---
## Enfoque en Ciberseguridad
## Hardening Aplicado y Validación Criptográfica

### Controles Defensivos Implementados

* **Mitigación de *DNS Rebinding* (`private-address`):** declarar los rangos RFC 1918 (`192.168.0.0/16`, `172.16.0.0/12`, `10.0.0.0/8`) y `169.254.0.0/16` en `unbound.conf` obliga a descartar cualquier respuesta pública externa que intente resolver hacia una IP privada de mi LAN.
* **Cierre de *Open Resolver* y *Fingerprinting*:** las ACLs (`access-control`) aceptan únicamente consultas desde `127.0.0.1/32` (healthcheck), redes Docker (`172.16.0.0/12`) y mi LAN (`192.168.1.0/24`), rechazando el resto (`0.0.0.0/0 refuse`). Además, `hide-identity: yes` y `hide-version: yes` ocultan la versión del motor, y `harden-glue: yes` + `harden-dnssec-stripped: yes` bloquean inyecciones de registros fuera de zona y degradaciones de firmas.
* **Montaje único en `:ro` preservando `var/root.key`:** monto únicamente `/opt/stacks/dns-stack/unbound/unbound.conf` como solo lectura (`:ro`) en vez de bloquear toda la carpeta `/opt/unbound/etc/unbound/`. Así el archivo de configuración queda inmutable desde adentro del contenedor, mientras la subcarpeta interna `var/` sigue escribible para que el script de inicio genere y actualice el `root.key`.
* **Mínimo privilegio en el kernel:** descarté `privileged: true` y le asigné a Pi-hole únicamente `cap_add: [NET_ADMIN]` para operar los sockets DNS. En ambos contenedores activé `security_opt: [no-new-privileges:true]` para impedir escaladas mediante binarios `SUID`/`SGID`, y Unbound corre bajo el usuario sin privilegios `_unbound`.
