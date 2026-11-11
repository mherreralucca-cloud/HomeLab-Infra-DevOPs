# Despliegue de UniFi Network Server en Contenedor LXC y Ajuste de frecuencia (UAP-AC-LR)

En esta etapa del proyecto necesitaba poner y administrar un punto de acceso **Ubiquiti UniFi UAP-AC-LR5** para cubrir el Wi-Fi de mi casa. Como los equipos de la línea UniFi no tienen una interfaz web propia para configurarlos desde una IP, requieren un controlador central (**UniFi Network Application**) que los adopte, les puse la configuración y recolecte las métricas de red.

En lugar de instalar ese controlador en mi PC personal (que se apaga y deja sin monitoreo ni portal a la red), decidí alojarlo directamente en mi servidor **Proxmox VE** para que corra las 24 horas consumiendo el mínimo posible de recursos.

---

## 1. Decisiones de Arquitectura: ¿Por qué un Contenedor LXC sin privilegios?

A la hora de elegir dónde instalar el controlador dentro de Proxmox tenía tres caminos: crear una Máquina Virtual completa, meterlo en Docker o usar un contenedor de sistema **LXC (Linux Containers)**. Opté por **LXC directo sobre Proxmox** por tres motivos:
 
### A. Eficiencia de Kernel compartido (`namespaces` y `cgroups`)
Levantar una Máquina Virtual (KVM) solo para correr el controlador implica emular hardware e iniciar un kernel de Linux entero desde cero. Con LXC aplico virtualización a nivel de sistema operativo: el contenedor comparte el mismo kernel del host Proxmox, pero vive aislado gracias a dos funciones nativas del kernel de Linux:
* **Namespaces (Espacios de nombres):** Aíslan lo que el contenedor puede ver. Si abro una terminal adentro del contenedor, tiene su propia tabla de procesos (donde su proceso `init` es el `PID 1`), su propio sistema de archivos montado y su propia interfaz de red (`eth0`), sin poder ver ni tocar nada de lo que corre en Proxmox o en otros contenedores.
* **Control Groups (`cgroups` v2):** Son los que ponen el techo de consumo. Al asignarle 2 núcleos y 2 GB de RAM desde Proxmox, el kernel mete al contenedor en un `cgroup` que impide que un pico de consumo de Java o MongoDB deje sin memoria al resto del servidor.

### B. Seguridad del modo *Unprivileged Container*
En Linux, correr servicios como `root` siempre tiene el riesgo de que una vulnerabilidad permita escalar privilegios. Al crear el contenedor LXC tildando la opción **Unprivileged container**, Proxmox hace un mapeo de usuarios (*UID mapping*): el usuario `root` adentro del contenedor (`UID 0`) en realidad corre en el servidor Proxmox como un usuario común sin privilegios (`UID 100000`). Si alguien lograra vulnerar la aplicación de UniFi y escapar del contenedor, caería en el host sin permisos de administrador.

### C. Por qué descarté Docker y el nuevo "UniFi OS Server"
* **El problema de la Capa 2 en Docker:** El controlador UniFi descubre los puntos de acceso mediante tramas de difusión (*broadcast*) en Capa 2. Si metía el controlador detrás de la red NAT por defecto de Docker (`docker0`), el mapeo de puertos suele bloquear el descubrimiento automático y obliga a adoptar los equipos a mano por SSH. En cambio, un contenedor LXC conectado al puente `vmbr0` tiene su propia dirección MAC e IP directa en la LAN, igual que un equipo físico.
* **UniFi Network Server (`:8443`) vs. UniFi OS Server (`:11443`):** Al entrar al panel aparece el aviso para actualizar a *UniFi OS Server*. Decidí mantener la versión pura (**UniFi Network Server**) porque *UniFi OS* agrega una capa extra (pensada para consolas Cloud Key o Dream Machine) que levanta contenedores internos con Podman (*rootless* con `slirp4netns`), pide entre 4 GB de RAM y más de 30 GB de disco, y se lleva mal con los contenedores LXC. Para administrar mis puntos de acceso, la versión independiente en el puerto `8443` tiene exactamente las mismas funciones de red consumiendo menos de la mitad de recursos.

---

## 2. Dimensionamiento de Recursos y Configuración de Red del Contenedor

Desde el almacenamiento del nodo descargué la plantilla oficial **`debian-12-standard`** (`CT Templates` → `Templates`) y creé el contenedor con esta configuración:

* **Hostname:** `unifi-controller` (Contenedor sin privilegios).
* **Disco (`rootfs`):** `16 GiB` alojados en **`local-lvm`** (sobre el SSD M.2 NVMe).
  * *Por qué en el NVMe y no en los HDD:* El controlador de UniFi guarda toda la telemetría de clientes, señales y tráfico en una base de datos **MongoDB** que hace escrituras aleatorias pequeñas todo el tiempo. En el NVMe no hay partes móviles y el acceso es instantáneo; en los discos mecánicos eso metería latencia y ruido constante.
* **CPU y Memoria:** `2 Cores`, `2048 MiB` de RAM y `512 MiB` de Swap.
  * *Por qué 2 GB de RAM:* UniFi levanta un servidor Java (JVM) junto con el motor de MongoDB. Darle 2 GB asegura que ambos procesos trabajen cómodos en la RAM física. Si le daba 512 MB, el sistema iba a entrar en *thrashing* (intercambiando memoria con el disco swap todo el tiempo), subiendo el uso de CPU y colgando el panel web.

### Diseño de Red (`vmbr0`, CIDR `/24` y Gateway)
Configuré la placa de red virtual del contenedor (`net0`) con parámetros estáticos para que los puntos de acceso siempre lo encuentren en el mismo lugar:

* **Bridge `vmbr0`:** Funciona como un switch virtual en Capa 2 que une la interfaz del contenedor con la placa Ethernet física del servidor enchufada al router principal.
* **IPv4 Estática (`192.168.1.101/24`):** A diferencia de un celular o una notebook que pueden pedir IP por DHCP, un servidor de infraestructura tiene que tener IP fija.
* **Default Gateway (`192.168.1.1`):** Es la IP interna del módem-router de Movistar. Cuando el contenedor tiene que salir a Internet (por ejemplo, para bajar paquetes con `apt`), le manda el tráfico a esta puerta de enlace, que se encarga de hacer **NAT (*Network Address Translation*)** para salir por mi única IP pública de fibra.
* **DNS (`1.1.1.1`):** Resuelve los nombres de dominio (como los repositorios de Debian o Ubiquiti) a direcciones IP de forma provisoria hasta poner en marcha mi propio Pi-hole + Unbound.

---

## 3. Instalación del Servicio y Puertos Habilitados

Adentro de la consola del contenedor (`root@unifi-controller`), actualicé el sistema base y utilicé el script de mantenimiento de Glenn R. (estándar en la comunidad de Ubiquiti porque resuelve automáticamente las dependencias exactas de Java y las versiones compatibles de MongoDB en Debian 12):

```bash
apt update && apt upgrade -y
apt install -y curl wget ca-certificates gnupg
wget [https://get.glennr.nl/unifi/install/install_latest/unifi-latest.sh](https://get.glennr.nl/unifi/install/install_latest/unifi-latest.sh)
bash unifi-latest.sh
```

* **Por qué instalé `ca-certificates` y `gnupg` antes:** Son necesarios para que Debian pueda validar los certificados SSL/TLS al descargar por HTTPS y verificar las firmas criptográficas (GPG) de los repositorios oficiales de MongoDB y Ubiquiti.

Una vez instalado, el demonio `unifi` queda escuchando en varios puertos de transporte (TCP y UDP) dentro de la IP `192.168.1.101`:

| Puerto / Protocolo | Uso dentro de la Arquitectura UniFi |
| :--- | :--- |
| **`8443/TCP`** | Interfaz web de administración cifrada por HTTPS (`https://192.168.1.101:8443`). |
| **`8080/TCP`** | Canal `inform`: es el puerto al que se conectan los puntos de acceso para reportar su estado y recibir cambios de configuración. |
| **`10001/UDP`** | Descubrimiento de dispositivos en Capa 2 dentro de la red local (usa UDP por velocidad y baja sobrecarga). |

---

## 4. Recuperación Física del UAP-AC-LR (Modo TFTP) y Adopción en Capa 2

Un Access Point (AP) no rutea tráfico ni hace NAT: trabaja como un **puente (*bridge*) transparente** que une el medio inalámbrico con el cable Ethernet para que toda la casa opere sobre la misma red local.

### Conexión PoE y salida del Modo de Recuperación TFTP
El UAP-AC-LR se alimenta por **PoE (*Power over Ethernet*)**: del módem-router sale un cable de red al puerto **`LAN`** del inyector PoE, y del puerto **`PoE`** del inyector sale el cable que lleva datos y corriente eléctrica juntos hasta el AP en el techo.

Como el equipo tenía una configuración vieja, intenté resetearlo de fábrica, pero al mantener apretado el botón físico `Reset` más tiempo del debido, el anillo LED empezó a **parpadear en secuencia blanco, azul y apagado cada 1 segundo**:
* **Por qué pasó esto:** El firmware de arranque (*bootloader*) de Ubiquiti tiene dos tiempos distintos en el botón `Reset`:
  1. **De 5 a 8 segundos (con el equipo prendido):** Borra la configuración de usuario y lo deja de fábrica (**luz blanca fija**).
  2. **Más de 15 segundos (o apretado al enchufar):** El AP asume que el firmware se corrompió y entra en **Modo de Recuperación TFTP (*Trivial File Transfer Protocol*)** (parpadeo blanco/azul/apagado). En ese estado no levanta el sistema operativo ni aparece en la red porque queda esperando que le carguen un firmware nuevo por cable.
* **Cómo lo destrabé:** Desenchufé el cable del puerto PoE durante 10 segundos y lo volví a conectar sin tocar nada para forzar un arranque normal. Una vez que inició, presioné el botón `Reset` con un clip durante **6 segundos exactos** hasta que parpadeó y se apagó. Al terminar de reiniciar quedó con la **luz blanca fija**, listo para adoptar.

### Qué hace el controlador por detrás al darle a "Adopt"
1. El AP en estado de fábrica (luz blanca) pide una IP al DHCP de la red.
2. Manda paquetes *broadcast* por la LAN buscando un controlador UniFi.
3. El contenedor LXC (`192.168.1.101`) lo escucha a través del puente `vmbr0` y lo lista en la pestaña **UniFi Devices**.
4. Al hacer clic en **Adopt**, el controlador entra por **SSH** al AP usando el usuario/clave de fábrica (`ubnt` / `ubnt`), le inyecta una clave criptográfica nueva atada a mi servidor y le graba la orden de reportarse siempre a `http://192.168.1.101:8080/inform`. Cuando termina de aprovisionar, el LED del AP pasa a **azul fijo** (*Online*).

---

## 5. Física de Radiofrecuencia: Ajuste para Terreno de 250 m² (100 m² Techados)

La distribución física de mi casa tiene una particularidad: el terreno total es de **250 m²**, con **100 m² cubiertos**. El módem de Movistar (junto con el servidor Proxmox y mi puesto de trabajo) está en una esquina de la casa, mientras que el **UAP-AC-LR** está instalado en el centro del techo, a unos 6 metros del módem.

Para lograr la máxima velocidad adentro de la casa (aprovechando los **500 Mbps** de fibra óptica) y que al mismo tiempo el AP llegue hasta el fondo del patio sin cortarse, configuré los radios aplicando estos criterios de radiofrecuencia:

### A. SSID, BSSID y Band Steering
* **SSID y WPA2:** Configuré el mismo nombre de red (**SSID**) y contraseña **WPA2** para que los dispositivos reconozcan las antenas como parte de la misma red. Internamente, el celular distingue cada antena física por su dirección MAC de radio (**BSSID**), lo que le permite saltar de una a otra (*roaming*).
* **Diferencia física entre 2.4 GHz y 5 GHz:**
  * **2.4 GHz (Onda de ~12.5 cm):** Al tener una longitud de onda más larga, atraviesa paredes de ladrillo con mucha menos pérdida de energía. Es la banda encargada de dar alcance hasta el fondo de los 250 m².
  * **5 GHz (Onda de ~6 cm):** Es una onda más corta que permite pasar mucho más caudal de datos (ideal para los 500 Mbps en los 100 m² techados), pero se frena rápido contra las paredes.
* **Band Steering:** Unifiqué ambas bandas bajo el mismo SSID y activé *Band Steering*. Como el 2.4 GHz llega con más fuerza, los teléfonos tienden a engancharse ahí primero; esta función detecta si el equipo soporta 5 GHz y lo empuja a la banda rápida cuando está cerca del AP.

### B. Ancho de Canal (*Channel Width*): Por qué `80 MHz` en 5 GHz y `20 MHz` en 2.4 GHz
El ancho de canal es el tamaño de la "tubería" de frecuencias por donde viajan los datos:

1. **En 5 GHz fijé `80 MHz` (Velocidad pura):**
   Un canal estándar de Wi-Fi mide 20 MHz, lo que en Wi-Fi 5 (802.11ac) rinde apenas unos 80-100 Mbps reales. Al poner **80 MHz**, el AP hace *Channel Bonding*: une 4 canales contiguos de 20 MHz (`20 x 4 = 80 MHz`) llevando el enlace teórico a 867 Mbps. Es lo que me permite pasar los **500 Mbps reales** de la fibra por aire adentro de la casa.
2. **En 2.4 GHz fijé `20 MHz` (Alcance y estabilidad):**
   Duplicar el ancho de canal tiene un costo físico directo: por cada duplicación (de 20 a 40 MHz, o de 40 a 80 MHz), la antena receptora abre una ventana más grande y **capta el doble de ruido de fondo (+3 dB de piso de ruido por cada salto)**, además de repartir la potencia de transmisión en un espectro más ancho. Sumado a eso, la banda de 2.4 GHz es muy angosta y está llena de interferencias de vecinos y Bluetooth. Dejarla clavada en **20 MHz** concentra toda la energía de la antena *Long Range* en un canal angosto y limpio para que atraviese las paredes hacia el patio.

### C. Elección de Canales Fijos (`1, 6, 11` y `36, 44`)
* **Evitar solapamiento en 2.4 GHz:** Aunque existen los canales del 1 al 13, están separados por solo 5 MHz entre sí. Como cada canal ocupa 20 MHz de ancho, los canales vecinos se pisan y meten ruido. Los **únicos tres canales que no se tocan entre sí son el `1`, el `6` y el `11`**. Fijé el AP en uno libre (distinto al del módem) para evitar interferencia co-canal.
* **Evitar canales DFS en 5 GHz:** En 5 GHz los canales no se pisan, pero el bloque intermedio (del `52` al `144`) corresponde a canales **DFS (*Dynamic Frequency Selection*)**, usados por radares meteorológicos y de aviación. Si el AP queda en *Auto* y cae en un canal DFS, ante cualquier interferencia parecida a un radar corta el Wi-Fi por varios minutos. Por eso fijé el canal **`36` / `44`** (banda baja UNII-1 sin DFS).

### D. Ajuste de Potencia y *Minimum RSSI* (El problema de los 250 m²)
El **RSSI** mide con qué fuerza escucha el AP al celular en **dBm** (valores negativos porque es una fracción de milivatio: `-50 dBm` es excelente al lado del AP, `-75 dBm` empieza a ser flojo y `-85 dBm` está al límite de cortarse).

Como los celulares sufren del efecto *Sticky Client* (se quedan aferrados a una antena aunque tengas otra más cerca hasta que la señal muere del todo), configuré el **Minimum RSSI** de forma asimétrica según el plano de mi casa:
* **En 5 GHz → Activado en `-75 dBm` (Potencia `High`):** Mientras me muevo por los 100 m² techados navego en 5 GHz a máxima velocidad. Apenas salgo al patio y la pared hace caer la señal de 5 GHz por debajo de `-75 dBm`, el AP suelta al teléfono en milisegundos para obligarlo a pasarse a 2.4 GHz.
* **En 2.4 GHz → Desactivado (Potencia `High`):** Al principio habíamos probado poner un límite de `-78 dBm` también en 2.4 GHz, pero como el módem de Movistar está encerrado en una esquina de la casa, el AP central es el único que llega al fondo del terreno de 250 m². Desactivar el *Minimum RSSI* en 2.4 GHz evita que el AP te expulse cuando estás en el fondo del patio con `-82 dBm`.

> **Nota sobre la convivencia con el módem del ISP:** Como entre la esquina del módem de Movistar y el UAP-AC-LR en el centro del techo hay apenas 5 a 7 metros, si el Wi-Fi del módem queda prendido los dispositivos que arrancan cerca del escritorio se quedan pegados al módem y no saltan al AP. La solución definitiva para limpiar el espectro en una superficie techada de 100 m² es apagar las placas Wi-Fi del módem entrando a `http://192.168.1.1` (con la clave de administración de la etiqueta del equipo) y dejar que el UAP-AC-LR central maneje el 100% del aire.

---

## 6. Cheatsheet: Administración del Contenedor y Diagnóstico UniFi

| Comando / Acción | Dónde se corre | Para qué sirve |
| :--- | :--- | :--- |
| `pct list` | Nodo Proxmox | Lista los contenedores LXC y verifica que el ID `100` (`unifi-controller`) esté en estado `running`. |
| `pct enter 100` | Nodo Proxmox | Entra directo a la terminal del contenedor sin necesitar SSH. |
| `systemctl status unifi` | Contenedor LXC | Revisa si el demonio del controlador UniFi y su instancia de Java están corriendo. |
| `systemctl status mongodb` | Contenedor LXC | Verifica el estado del motor de base de datos del controlador. |
| `ss -tulnp \| grep -E '8443\|8080\|10001'` | Contenedor LXC | Comprueba que los puertos de la interfaz web (`8443`), adopción (`8080`) y descubrimiento (`10001`) estén abiertos. |
| **LED Blanco Fijo** | Físico en UAP-AC-LR | El AP está en valores de fábrica y esperando ser adoptado por el controlador. |
| **LED Azul Fijo** | Físico en UAP-AC-LR | El AP ya fue adoptado, tiene configuración inyectada y está operativo. |
| **LED Blanco / Azul / Apagado** | Físico en UAP-AC-LR | Modo de recuperación TFTP (se sale desenchufando el cable PoE 10 segundos). |
