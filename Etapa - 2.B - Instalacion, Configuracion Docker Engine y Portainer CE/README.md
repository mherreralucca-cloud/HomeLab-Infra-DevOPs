# Etapa 2.B — Instalación de Docker Engine y Despliegue de Portainer CE

Con la máquina virtual base (`docker-prod` — `192.168.1.102`) ya endurecida y con el snapshot `base-debian-hardened` guardado en Proxmox, en esta segunda parte instalé el motor de contenedores oficial **Docker Engine** junto con el plugin de **Docker Compose**, configuré la rotación automática de logs para cuidar el disco NVMe y levanté **Portainer CE** como interfaz web de gestión (y motor GitOps para conectar más adelante este repositorio de GitHub).

> **Nota de lectura:** Arranco directo con la arquitectura y el paso a paso de cómo dejé todo funcionando. Si querés entender a fondo qué pasa por debajo (cómo valida firmas la llave GPG en APT, la cadena `dockerd → containerd → runc`, el aislamiento con `namespaces`/`cgroups`/`overlay2` y cómo opera la red virtual `docker0`), dejé todo el **Marco Teórico desarrollado al final del documento**.

---

## Arquitectura del Entorno

Así quedó la capa de contenedores desplegada dentro de la VM `docker-prod` al terminar esta etapa:

```text
VM 200: docker-prod (Debian 12 — IP 192.168.1.102/24)
├── Repositorio oficial APT: /etc/apt/sources.list.d/docker.list
│   └── Firma aislada (signed-by): /etc/apt/keyrings/docker.asc
├── Motor Docker (dockerd → containerd → runc)
│   ├── Política global: /etc/docker/daemon.json (rotación json-file: máx. 3 x 10 MB)
│   ├── Socket de control: /var/run/docker.sock (acceso grupo docker: lucca)
│   ├── Almacenamiento: /var/lib/docker/overlay2/ (capas) + /volumes/ (persistencia)
│   └── Switch virtual por defecto: docker0 (172.17.0.1/16)
└── Contenedor activo: portainer (portainer/portainer-ce:latest)
    ├── Puerto expuesto: 9443/TCP (HTTPS) → [https://192.168.1.102:9443](https://192.168.1.102:9443)
    ├── Bind Mount: /var/run/docker.sock:/var/run/docker.sock
    └── Volumen persistente: portainer_data → /data
```

---

## Implementación Práctica Paso a Paso

### Paso 1: Registro del Repositorio Oficial de Docker y Llave GPG

En lugar de instalar versiones viejas desde los repositorios genéricos de Debian, configuré el repositorio oficial de Docker validado con su propia llave GPG aislada en `/etc/apt/keyrings/`.

1. **Descargar la llave pública GPG oficial:**
   Creé el directorio `/etc/apt/keyrings` con permisos `0755`, bajé la clave de Docker con `curl` y le di permisos de lectura (`a+r`) para que `apt` pueda comprobar las firmas:

   ```bash
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL [https://download.docker.com/linux/debian/gpg](https://download.docker.com/linux/debian/gpg) -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc
   ```

2. **Añadir la fuente de paquetes a APT:**
   Este comando detecta automáticamente la arquitectura del procesador (`amd64`) y el nombre en clave de mi versión de Debian desde `/etc/os-release` (`bookworm`), y escribe el archivo `/etc/apt/sources.list.d/docker.list` atado estrictamente a la llave recién descargada mediante `signed-by`:

   ```bash
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] [https://download.docker.com/linux/debian](https://download.docker.com/linux/debian) \
     $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

3. **Instalar Docker Engine y el plugin de Docker Compose:**
   Actualicé el índice de paquetes e instalé el motor (`docker-ce`), el cliente de consola (`docker-ce-cli`), el runtime (`containerd.io`), el constructor de imágenes (`docker-buildx-plugin`) y el orquestador declarativo (`docker-compose-plugin`):

   ```bash
   sudo apt update
   sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

---

### Paso 2: Rotación de Logs (`daemon.json`) y Permisos de Usuario

Antes de levantar el primer contenedor, configuré el límite de logs del demonio para evitar que los registros en JSON crezcan sin control sobre los 40 GiB del disco NVMe:

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
```

Luego habilité el servicio para que arranque con el sistema, lo reinicié para tomar la configuración del `daemon.json` y sumé mi usuario (`lucca`) al grupo `docker` para ejecutar comandos sin anteponer `sudo` cada vez (usando `newgrp docker` para aplicar el grupo en la sesión SSH actual sin desconectarme):

```bash
sudo systemctl enable --now docker
sudo systemctl restart docker
sudo usermod -aG docker $USER
newgrp docker
```

**Auditoría de la instalación y verificación de `docker0`:**

```bash
docker version
docker compose version
docker info --format 'Logging Driver: {{.LoggingDriver}}'
ip -c addr show docker0
```

Con `ip -c addr show docker0` confirmé que el demonio creó correctamente el puente virtual con la IP `172.17.0.1/16`.

---

### Paso 3: Despliegue de Portainer CE (Interfaz HTTPS y Motor GitOps)

Para tener control visual de los contenedores y desplegar futuros stacks leyendo directamente los archivos `docker-compose.yml` desde este repositorio de GitHub, levanté **Portainer Community Edition**.

1. **Crear el volumen persistente:**
   Como el sistema de archivos interno de un contenedor es efímero (`overlay2`), creé el volumen administrado `portainer_data` (alojado físicamente en `/var/lib/docker/volumes/portainer_data/_data`) para que las credenciales, ajustes y vínculos con GitHub persistan ante reinicios o actualizaciones:

   ```bash
   docker volume create portainer_data
   ```

2. **Lanzar el contenedor de Portainer:**

   ```bash
   docker run -d \
     -p 9443:9443 \
     --name portainer \
     --restart=always \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v portainer_data:/data \
     portainer/portainer-ce:latest
   ```

   Desglose de los parámetros utilizados:
   * `-d` (*Detached*): corre el contenedor en segundo plano y devuelve el control de la terminal.
   * `-p 9443:9443`: publica el puerto TCP `9443` de la VM (`192.168.1.102`) apuntando al puerto `9443` interno de Portainer (interfaz web HTTPS). Omití a propósito el puerto `8000` porque solo se utiliza para túneles de agentes Edge remotos que acá no necesito, reduciendo la superficie expuesta.
   * `--name portainer`: asigna un nombre fijo y fácil de identificar al contenedor.
   * `--restart=always`: instruye a `dockerd` para que vuelva a levantar Portainer automáticamente si el proceso falla o cuando la VM se reinicia.
   * `-v /var/run/docker.sock:/var/run/docker.sock`: *Bind Mount* que expone el socket Unix del motor Docker de la VM adentro del contenedor, permitiendo que Portainer gestione contenedores, redes y stacks.
   * `-v portainer_data:/data`: monta el volumen persistente dentro de `/data`, donde Portainer guarda su base de datos.
   * `portainer/portainer-ce:latest`: imagen oficial descargada desde Docker Hub (`docker.io`).

---

### Paso 4: Verificación por Consola e Inicialización Web

Comprobé en la terminal que el contenedor estuviera corriendo, el volumen montado, el puerto `9443` escuchando y el puente `docker0` ya activo:

```bash
docker ps
docker volume ls
sudo ss -tulpn | grep 9443
ip -c addr show docker0
```

Por último, ingresé desde el navegador a **`https://192.168.1.102:9443`**, acepté la advertencia del certificado TLS autofirmado y configuré la cuenta de administrador local.

---

## Errores Comunes y Troubleshooting

* **La interfaz `docker0` figura en `state DOWN` con `NO-CARRIER` tras instalar Docker:**
  Al tirar `ip -c addr show docker0` apenas instalado Docker (antes de lanzar Portainer), el puente aparece con `<NO-CARRIER,BROADCAST,MULTICAST,UP> ... state DOWN`. No es una falla: `docker0` funciona igual que un switch físico al que todavía no le enchufaste ningún cable. Apenas levantás el primer contenedor sobre la red por defecto (en este caso Portainer), Docker conecta un extremo virtual (`veth`) al puente y `docker0` pasa automáticamente a `state UP`.

* **Bloqueo del asistente inicial de Portainer por timeout (5 minutos):**
  Por seguridad, si pasan más de 5 minutos desde que arranca el contenedor `portainer` sin que nadie cree el usuario administrador en `https://192.168.1.102:9443`, Portainer bloquea el formulario inicial. Si pasa eso, simplemente se reinicia el contenedor para resetear el temporizador y se recarga el navegador:

  ```bash
  docker restart portainer
  ```

---

## Perspectiva de Ciberseguridad (Por qué apliqué estos controles)

1. **Prevención de Denegación de Servicio por disco lleno (`/etc/docker/daemon.json`):**
   A diferencia de los demonios tradicionales de Linux que rotan sus logs con `logrotate` o `journald`, Docker captura toda la salida (`stdout`/`stderr`) de cada contenedor en archivos JSON sin tope de tamaño por defecto. Un servicio con mucho tráfico (como Pi-hole recibiendo miles de consultas diarias) podría llenar los 40 GiB de la VM y tirar todo el servidor. Fijar `"max-size": "10m"` y `"max-file": "3"` pone un techo de 30 MB por contenedor.

2. **Privilegios críticos del socket `/var/run/docker.sock` y del grupo `docker`:**
   El daemon `dockerd` corre como `root`. Tanto cualquier usuario incluido en el grupo `docker` como cualquier contenedor que tenga montado `/var/run/docker.sock` (como Portainer) tiene control total sobre el motor de contenedores de la VM. Por eso el acceso a Portainer queda restringido por HTTPS en el puerto `9443` solo dentro de la red local y protegido además por el hardening SSH de la Etapa 2.A.

3. **Evasión de `ufw` por reglas NAT de Docker:**
   Cuando se publica un puerto con `-p`, Docker inserta reglas `DNAT` en la cadena `PREROUTING` del kernel y envía el tráfico a `FORWARD` antes de que llegue a la cadena `INPUT` que filtra `ufw`. Por esa razón solo publico los puertos necesarios en cada contenedor y evito exponer servicios internos que no necesiten acceso desde la LAN.

---

## Hoja de Trucos (Cheat Sheet) — VM Base, Docker y Portainer

| Categoría | Comando / Ruta | Qué hace y para qué sirve |
| :--- | :--- | :--- |
| **Agente Proxmox** | `systemctl status qemu-guest-agent` | Verifica que el demonio que reporta la IP a Proxmox y congela el disco (`fsfreeze`) en los backups esté activo. |
| **Red Debian** | `/etc/network/interfaces` | Archivo donde se define la IP estática `192.168.1.102/24` sobre la placa VirtIO `ens18`. |
| **Seguridad SSH** | `sudo sshd -t && sudo systemctl restart sshd` | Valida que `/etc/ssh/sshd_config` no tenga errores de sintaxis antes de reiniciar el servicio SSH. |
| **Firewall Host** | `sudo ufw status numbered` | Muestra las reglas activas del firewall local numeradas por orden de evaluación. |
| **Criptografía APT** | `/etc/apt/keyrings/docker.asc` | Llave pública GPG oficial usada por `apt` para verificar la firma digital de los paquetes de Docker. |
| **Config Docker** | `/etc/docker/daemon.json` | Archivo de configuración del demonio `dockerd` (usado para limitar el tamaño y rotación de logs JSON). |
| **Socket Docker** | `/var/run/docker.sock` | Canal de comunicación (socket Unix) por el cual la CLI y Portainer le dan órdenes al demonio `dockerd`. |
| **Capas y Datos** | `/var/lib/docker/overlay2/` y `/var/lib/docker/volumes/` | Directorios donde Docker guarda las capas de las imágenes (`overlay2`) y los volúmenes persistentes (`portainer_data`). |
| **Gestión Contenedores** | `docker ps` / `docker ps -a` | Lista los contenedores activos (`ps`) o absolutamente todos incluyendo los detenidos (`ps -a`). |

---

## Marco Teórico: Criptografía GPG, Motor Docker y Redes de Contenedores

Acá dejo el detalle técnico de todo lo que sucede por detrás cuando instalamos y operamos Docker en Linux.

### 1. ¿Qué es una Llave GPG y por qué la descargamos para instalar Docker?
Cuando corremos `apt install` en Debian, el sistema baja paquetes `.deb` desde servidores remotos. Si alguien interceptara la conexión o suplantara un servidor (*Man-in-the-Middle*), podría inyectar un paquete adulterado que instalaríamos con permisos de `root`. Para evitarlo, Debian usa **SecureApt con GPG (*GNU Privacy Guard*)**:
* **Firma digital en origen:** Al compilar una nueva versión de `docker-ce`, el equipo de Docker firma el índice del repositorio (`Release`) usando su **Llave Privada GPG**.
* **El archivo `/etc/apt/keyrings/docker.asc`:** Es la **Llave Pública GPG oficial de Docker**. Al guardarla en `/etc/apt/keyrings/` y referenciarla con `signed-by=/etc/apt/keyrings/docker.asc` en `/etc/apt/sources.list.d/docker.list`, obligamos a `apt` a cumplir dos reglas:
  1. Verificar matemáticamente con `docker.asc` que la firma del paquete sea legítima y no se haya alterado ni un solo bit en tránsito.
  2. **Aislamiento de confianza (`signed-by`):** A diferencia del método viejo `apt-key` (que agregaba llaves a un llavero global), esta clave solo sirve para validar paquetes del repositorio de Docker y de ningún otro origen del sistema.

### 2. La arquitectura interna: `docker-ce-cli` → `dockerd` → `containerd` → `runc`
Docker no es un binario monolítico, sino una cadena modular donde cada pieza cumple un rol específico:
* **`docker-ce-cli` (Cliente):** Es el comando `docker` que escribimos en consola. No ejecuta contenedores por sí mismo; traduce nuestras órdenes a llamadas API y las envía a través del socket `/var/run/docker.sock`.
* **`docker-ce` / `dockerd` (Demonio servidor):** Corre en segundo plano como `root`. Gestiona imágenes, redes virtuales (`docker0`), volúmenes y escucha las peticiones tanto de la CLI como de Portainer.
* **`containerd.io` y `runc` (Runtime de bajo nivel):** Cuando `dockerd` necesita arrancar un contenedor, delega el ciclo de vida a `containerd`. Este invoca a `runc`, una herramienta liviana que habla directo con el kernel de Debian para pedirle que cree el entorno aislado y lance el proceso del contenedor.

### 3. Cómo funciona un contenedor por dentro (`namespaces`, `cgroups` y `overlay2`)
Un contenedor Docker **no levanta un kernel propio** ni emula hardware como una VM; es un proceso Linux corriendo sobre el kernel de la VM Debian, pero confinado mediante tres mecanismos nativos del kernel:
1. **Namespaces (Aislamiento de visibilidad):** Hacen que el proceso crea estar solo en su propia máquina:
   * *Process table (`pid`):* La aplicación dentro del contenedor (ej. Portainer) se ve como `PID 1` y no puede ver los procesos del host.
   * *Network (`netns`):* Tiene su propia placa de red virtual (`eth0`) y su propia dirección IP privada.
   * *Mount (`mnt`):* Ve su propio árbol de directorios raíz (`/`) y no puede acceder al sistema de archivos de la VM salvo en las rutas montadas explícitamente con `-v`.
2. **Control Groups (`cgroups`):** Mecanismo del kernel que mide y limita el consumo de RAM, CPU y cantidad de procesos de cada contenedor para evitar que uno solo sature la VM.
3. **Sistema de archivos por capas (`overlay2`):** Las imágenes descargadas se almacenan en `/var/lib/docker/overlay2/` divididas en **capas inmutables de solo lectura (`lowerdir`)**. Al iniciar un contenedor, Docker monta encima una capa delgada de lectura/escritura (`upperdir`). Si borrás el contenedor, esa capa superior se descarta y la imagen base permanece intacta. Por eso, los datos que deben sobrevivir a actualizaciones o recreaciones se guardan afuera usando **Volúmenes (`-v`)**.

### 4. Qué es la interfaz `docker0` (`172.17.0.1/16`) y cómo rutea el tráfico
Al instalarse, `dockerd` crea un switch virtual interno en Capa 2 llamado **`docker0`** con la IP **`172.17.0.1/16`** y agrega la ruta correspondiente en la tabla de enrutamiento del kernel:
* **Pares virtuales (`veth`):** Cada contenedor conectado a la red bridge por defecto recibe un "cable virtual" de dos puntas: un extremo (`veth<id>`) queda conectado a `docker0` en la VM, y el otro (`eth0`) entra al *network namespace* del contenedor con una IP como `172.17.0.2`.
* **Salida a Internet mediante NAT:** Para que los contenedores alcancen Internet, Docker habilita el reenvío de paquetes (`net.ipv4.ip_forward = 1`) y aplica **NAT (*IP Masquerading*)**, traduciendo el tráfico saliente con la IP `192.168.1.102` de la VM.
* **Publicación de puertos (`-p`):** Como la subred `172.17.0.0/16` es interna de la VM, el parámetro `-p 9443:9443` configura el reenvío del tráfico entrante que llega al puerto `9443` de la VM hacia el puerto `9443` del contenedor Portainer.
