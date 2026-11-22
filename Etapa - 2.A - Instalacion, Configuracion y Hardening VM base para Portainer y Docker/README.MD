# Etapa 2.A — Aprovisionamiento y Hardening de la VM Base (`docker-prod`)

En esta primera parte de la Etapa 2 armé desde cero la máquina virtual que va a alojar todo el motor de contenedores del servidor (**VM 200 — `docker-prod`**) corriendo **Debian 13** sin entorno gráfico sobre Proxmox VE. La idea acá es dejar una base limpia, reproducible y segura antes de instalar Docker y Portainer: configurar bien el hardware virtualizado, asignar IP estática, activar el agente de comunicación con Proxmox, cerrar el acceso SSH para usar únicamente llaves Ed25519, levantar el firewall `ufw` en modo *Default Deny* y sacar un snapshot de resguardo.

> **Nota de lectura:** Arranco directo con la arquitectura y el paso a paso de cómo lo implementé. Si querés entender a fondo el porqué de cada decisión (KVM vs. QEMU, paravirtualización `VirtIO`, funcionamiento de `LVM-thin` con `Discard`, congelamiento de disco con `fsfreeze` y cómo opera la criptografía `Ed25519`), te dejé todo el **Marco Teórico desarrollado al final del documento**.

---

## Arquitectura del Entorno

Así quedó planteada la máquina virtual dentro del nodo Proxmox y la red local antes de desplegar los contenedores:

```text
Nodo Proxmox VE (pve — Intel Core i3-12100)
├── Bridge de red: vmbr0 (LAN 192.168.1.0/24 | Gateway: 192.168.1.1)
└── VM 200: docker-prod (Debian 13 minimal — sin GUI)
    ├── CPU / RAM: 2-4 vCPUs (Type: host) | 6144 MiB RAM fija
    ├── Almacenamiento: 40 GiB en local-lvm (M.2 NVMe | VirtIO SCSI single + Discard)
    ├── Interfaz de red: ens18 (VirtIO) → IP estática 192.168.1.102/24
    ├── Integración host: qemu-guest-agent activo
    ├── Hardening base: SSH solo llave Ed25519 (usuario lucca) + ufw Default Deny
    └── Resguardo: Snapshot base-debian-hardened
```

---

## Implementación Práctica Paso a Paso

### Paso 1: Creación de la VM 200 (`docker-prod`) en Proxmox VE

1. **Descarga de la ISO mínima de Debian:**
   Desde la interfaz web de Proxmox entré al almacenamiento `local` → **ISO Images** → **Download from URL** y descargué la imagen oficial *netinst* de Debian 13 (`debian-13.9.0-amd64-netinst.iso`) para hacer una instalación limpia por red.

2. **Configuración del hardware virtual (`Create VM`):**
   Armé la máquina virtual cuidando de usar controladores paravirtualizados en lugar de hardware emulado estándar:
   * **General:** `VM ID: 200`, `Name: docker-prod` y la casilla **Start at boot** activada (así arranca sola apenas vuelva la luz y levante el servidor físico).
   * **OS:** Seleccioné la ISO de Debian 13 recién descargada.
   * **System:** Controladora **VirtIO SCSI single** y casilla **Qemu Agent** tildada.
   * **Disks:** Bus `SCSI 0`, almacenamiento **`local-lvm`** (NVMe), tamaño `40 GiB` y casilla **Discard** activada para habilitar TRIM sobre el *thin pool*.
   * **CPU:** `2 Cores` (o `4 Cores`) con **Type: `host`**, para exponerle a la VM el conjunto completo de instrucciones nativas del Intel Core i3-12100 sin traducción intermedia.
   * **Memory:** `6144 MiB` (6 GB de RAM) fijos.
   * **Network:** Bridge **`vmbr0`**, modelo **VirtIO (paravirtualized)**.

3. **Instalación del sistema operativo (Consola de la VM):**
   * Configuré región (`Argentina`), contraseña de `root` y mi usuario estándar sin privilegios (`lucca`).
   * **Particionado:** Elegí *Guiado - utilizar todo el disco* en una única partición `ext4`. No activé LVM adentro de Debian porque el disco virtual ya corre sobre el volumen `LVM-thin` de Proxmox y sumar una segunda capa de LVM no aporta nada en este caso.
   * **Selección de programas (`tasksel`):** Destildé el entorno de escritorio (`Debian desktop environment` / `GNOME`) para no gastar RAM ni CPU en interfaz gráfica. Dejé únicamente **`SSH server`** y **`standard system utilities`**.

---

### Paso 2: Paquetes Base, Utilidades de Red y Agente QEMU

Una vez instalada y reiniciada la VM, entré por la consola de Proxmox como `root` para actualizar el sistema e instalar las herramientas base de administración:

* `sudo`: permite administrar el equipo desde el usuario estándar `lucca` sin tener que usar `root`.
* `ufw`: capa amigable para gestionar el firewall del kernel (`nftables`/`iptables`).
* `ca-certificates`, `curl` y `gnupg`: necesarios para bajar paquetes por HTTPS y validar firmas criptográficas GPG.
* `bind9-dnsutils`: suma herramientas de diagnóstico DNS (`dig` y `nslookup`) que no vienen en la instalación mínima de Debian.
* `qemu-guest-agent`: demonio encargado de comunicar la VM con Proxmox VE.

```bash
apt update && apt full-upgrade -y
apt install -y vim sudo ufw curl ca-certificates gnupg git qemu-guest-agent bind9-dnsutils

# Sumar el usuario estándar al grupo sudo
usermod -aG sudo lucca
```

Después inicié y verifiqué el estado del agente de QEMU:

```bash
systemctl start qemu-guest-agent
systemctl status qemu-guest-agent
```

*(En la sección de troubleshooting más abajo dejé anotado por qué no se usa `systemctl enable` en Debian con este servicio).*

---

### Paso 3: Configuración de IP Estática (`192.168.1.102/24`)

Como esta VM va a alojar el servidor DNS de la casa (Pi-hole + Unbound), no puede depender de una IP dinámica por DHCP que cambie después de un reinicio. Antes de tocar nada hice un respaldo de `/etc/network/interfaces` y lo edité:

```bash
cp /etc/network/interfaces /etc/network/interfaces.bak
vim /etc/network/interfaces
```

Dejé configurada la interfaz `ens18` con la IP fija `192.168.1.102/24` apuntando al módem de Movistar (`192.168.1.1`):

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug ens18
iface ens18 inet static
    address 192.168.1.102/24
    gateway 192.168.1.1
    dns-nameservers 1.1.1.1 8.8.8.8
```

Limpié la dirección DHCP previa de la placa, reinicié el servicio de red y verifiqué con `ip -c a`:

```bash
ip addr flush dev ens18 && systemctl restart networking
ip -c a
```

---

### Paso 4: Llave SSH (`Ed25519`) y Hardening de `sshd`

1. **Generación y copia de la llave desde mi PC de trabajo (Arch Linux/Omarchy):**
   Desde la terminal local generé un par de llaves criptográficas `Ed25519` y envié la llave pública hacia la VM (siempre sin `sudo` para usar la llave del usuario local y no la de `/root/`):

   ```bash
   ssh-keygen -t ed25519
   ssh-copy-id -i ~/.ssh/id_ed25519.pub lucca@192.168.1.102
   ```

2. **Verificación y endurecimiento del servidor SSH (`sshd_config`):**
   Primero comprobé que `ssh lucca@192.168.1.102` entrara directo sin pedirme contraseña. Recién con eso confirmado respaldé y edité la configuración de SSH dentro de la VM:

   ```bash
   sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
   sudo vim /etc/ssh/sshd_config
   ```

   Configuré estas tres cosas:

   ```text
   PermitRootLogin no
   PasswordAuthentication no
   PubkeyAuthentication yes
   MaxAuthTries 3 (opcional)
   ```

   Antes de reiniciar el servicio, validé la sintaxis con `sshd -t` para evitar un bloqueo de acceso (*SSH lockout*):

   ```bash
   sudo sshd -t && sudo systemctl restart sshd
   ```

---

### Paso 5: Firewall (`ufw`) en *Default Deny* y Snapshot de Resguardo

Configuré el firewall local bloqueando todo el tráfico entrante por defecto y abriendo únicamente los puertos que necesita esta VM, siempre restringidos a mi subred local (`192.168.1.0/24`):

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp comment 'SSH Admin LAN'
sudo ufw allow from 192.168.1.0/24 to any port 53 proto tcp comment 'Pi-hole DNS TCP'
sudo ufw allow from 192.168.1.0/24 to any port 53 proto udp comment 'Pi-hole DNS UDP'
sudo ufw allow from 192.168.1.0/24 to any port 8080 proto tcp comment 'Pi-hole Web UI'
sudo ufw allow from 192.168.1.0/24 to any port 9443 proto tcp comment 'Portainer HTTPS UI'

sudo ufw enable
sudo ufw status numbered
```

**Snapshot de seguridad en Proxmox:**
Con el sistema base actualizado, asegurado y sin todavía tocar Docker, entré a la web de Proxmox (`VM 200` → **Snapshots** → **Take Snapshot**) y saqué la instantánea **`base-debian-hardened`**. Si en algún momento rompo algo en las siguientes etapas, puedo volver a este estado limpio en un par de segundos.

---

## Errores Comunes y Troubleshooting

Durante la puesta en marcha aparecieron dos comportamientos puntuales que vale la pena dejar documentados:

* **Aviso de `systemd` al habilitar `qemu-guest-agent`:**
  Al correr `systemctl enable qemu-guest-agent` en Debian, `systemd` avisa que la unidad no tiene sección `[Install]`. No es un error: en Debian este servicio es una unidad estática que `udev` dispara automáticamente apenas detecta el puerto virtual `/dev/virtio-ports/org.qemu.guest_agent.0` creado por Proxmox. Solo hace falta iniciarlo con `systemctl start qemu-guest-agent`. Si tildaste la casilla *Qemu Agent* en Proxmox con la VM ya encendida, tenés que apagarla por completo y hacer un arranque en frío (un `reboot` interno no alcanza para que QEMU genere el dispositivo virtual).

* **Error `Bad key types '+ssh-rsa,ssh-dss'` al enviar la llave desde Arch Linux:**
  Al ejecutar `ssh-copy-id`, el cliente OpenSSH de mi PC abortó porque el archivo local `~/.ssh/config` arrastraba la directiva del algoritmo viejo `ssh-dss` (DSA), cuyo soporte fue eliminado en versiones recientes de OpenSSH por inseguro. Lo resolví respaldando mi configuración local y limpiando el algoritmo obsoleto antes de volver a correr `ssh-copy-id`:

  ```bash
  cp ~/.ssh/config ~/.ssh/config.bak
  sed -i 's/,ssh-dss//g' ~/.ssh/config
  ssh-copy-id -i ~/.ssh/id_ed25519.pub lucca@192.168.1.102
  ```

---

## Perspectiva de Ciberseguridad (Por qué apliqué este Hardening)

1. **Bloqueo de `root` y contraseñas en `sshd_config`:**
   * **El riesgo:** Cualquier atacante o bot automatizado sabe que en todo servidor Linux existe el usuario `root` (`UID 0`). Dejar `PermitRootLogin` y `PasswordAuthentication` activos expone el equipo a ataques de fuerza bruta ante cualquier dispositivo comprometido dentro de la red.
   * **La mitigación:** Deshabilitando `PermitRootLogin no` y `PasswordAuthentication no`, para entrar al servidor es obligatorio tener el archivo físico de la llave privada `Ed25519` del usuario `lucca` y recién adentro elevar privilegios mediante `sudo`.

2. **Firewall `ufw` con política *Default Deny* en la cadena `INPUT`:**
   * **El riesgo:** Por defecto, las cadenas del firewall del kernel (`INPUT`, `FORWARD`, `OUTPUT`) arrancan en política `ACCEPT`, dejando pasar tráfico hacia cualquier puerto que un programa abra por accidente.
   * **La mitigación:** Se cierra el 100% de las conexiones entrantes (`default deny incoming` / política `DROP` en `INPUT`) y se habilitan solamente los puertos indispensables (`22`, `53`, `8080`, `9443`), restringiendo el origen de forma explícita a la LAN `192.168.1.0/24`.

---

## Marco Teórico: Virtualización KVM/QEMU, Almacenamiento y Criptografía SSH

Acá dejo el sustento técnico de las decisiones que tomé al crear y configurar esta máquina virtual.

### 1. ¿Qué diferencia hay entre QEMU y KVM en Proxmox VE?
Cuando levantamos una Máquina Virtual en Proxmox, trabajan juntas dos piezas con roles distintos:
* **QEMU (*Quick Emulator*):** Corre en el espacio de usuario del servidor Proxmox (como un proceso `root` independiente por cada VM encendida) y se encarga de emular el hardware periférico: la placa madre, el firmware BIOS (SeaBIOS), la controladora de discos y la placa de red. Para el Debian que corre adentro, todos esos dispositivos emulados se ven como fierro físico real.
* **KVM (*Kernel-based Virtual Machine*):** Es un módulo nativo del kernel de Linux. Si QEMU tuviera que traducir por software cada instrucción del procesador, la VM sería lentísima. Gracias a que en la Etapa 1 activé **Intel VT-x** en la BIOS de la placa MSI, QEMU se apoya en KVM para ejecutar el código de la VM directo sobre los núcleos físicos del Intel Core i3-12100, logrando un rendimiento casi igual al de una PC física.

### 2. Paravirtualización (`VirtIO`) vs. Hardware Emulado Común
En los parámetros de disco y red elegí **VirtIO** en lugar de `SATA` o `Intel E1000`:
* Si seleccionás `SATA` o `E1000`, QEMU tiene que gastar ciclos de CPU simulando una placa física vieja, y el sistema invitado pierde tiempo traduciendo instrucciones para ese hardware ficticio.
* **VirtIO** aplica **paravirtualización**: el kernel de Debian sabe que está corriendo dentro de una máquina virtual y carga drivers específicos (`virtio_net` y `virtio_scsi`) para saltearse la emulación y comunicarse directo con el hipervisor Proxmox con mayor velocidad y menor consumo de CPU.

### 3. Almacenamiento `local-lvm` (`LVM-thin`), `Discard` (TRIM) y `QEMU Guest Agent` (`fsfreeze`)
* **Aprovisionamiento fino (*Thin Provisioning*):** El almacenamiento `local-lvm` en el M.2 NVMe es un *thin pool* de LVM. En lugar de reservar los 40 GiB físicos de golpe al crear la VM, `LVM-thin` asigna bloques reales del NVMe solo a medida que Debian escribe datos.
* **Recuperación de espacio con `Discard` (TRIM):** Cuando adentro de la VM borrás un archivo o eliminás una imagen de Docker de 2 GB, el sistema de archivos `ext4` de Debian marca ese espacio como libre en su tabla interna, pero no le avisa al disco subyacente. Sin esa notificación, el volumen en el NVMe crecería hasta ocupar los 40 GiB físicos aunque adentro solo haya 5 GB usados. Al tildar **Discard**, cada borrado envía una instrucción `TRIM` (`SCSI UNMAP`) a través de `VirtIO SCSI` para que `local-lvm` libere realmente esos bloques en el NVMe.
* **El rol del `qemu-guest-agent` y el congelamiento de disco (`fsfreeze`):** El kernel de Linux no escribe los datos al disco en el instante exacto en que un programa los guarda; primero los retiene en un búfer en la RAM para ganar velocidad y los vuelca al disco segundos después. Si Proxmox saca un backup o un snapshot justo en el medio, una base de datos de la VM puede quedar corrupta. El paquete `qemu-guest-agent` escucha a Proxmox por un canal serie virtual (`/dev/virtio-ports/org.qemu.guest_agent.0`): un instante antes de hacer el backup, Proxmox manda la orden `guest-fsfreeze-freeze` para que el agente fuerce al kernel de Debian a volcar toda la RAM al disco (`sync`) y congele las escrituras durante una fracción de segundo, asegurando una copia íntegra (además de reportar la IP de la VM en el panel de Proxmox).

### 4. Llaves Criptográficas SSH (`Ed25519`) vs. Contraseñas
Al iniciar sesión por SSH con contraseña, tu clave viaja (dentro del túnel cifrado) al servidor para validarse, dejando la puerta abierta a ataques de fuerza bruta. Con **criptografía asimétrica** (`ssh-keygen -t ed25519`) el modelo cambia por completo:
* **Llave Privada (`id_ed25519`):** Es secreta y nunca sale de tu PC de trabajo (funciona como el molde físico de la llave).
* **Llave Pública (`id_ed25519.pub`):** Actúa como el "cerrojo" que se copia al servidor dentro de `~/.ssh/authorized_keys`. Aunque alguien lea tu llave pública, es matemáticamente inviable deducir la privada a partir de ella.
* **Cómo ocurre la autenticación:** Al correr `ssh lucca@192.168.1.102`, el servidor usa tu llave pública para enviarle un desafío matemático a tu PC. Tu PC lo resuelve firmándolo con la llave privada sin exponerla jamás por la red; si la firma cierra, te deja entrar. Elegí el algoritmo de curva elíptica **`Ed25519`** porque genera llaves cortas, muy rápidas y mucho más seguras frente a ataques modernos que las viejas RSA de 2048 bits o el algoritmo roto DSA (`ssh-dss`).
