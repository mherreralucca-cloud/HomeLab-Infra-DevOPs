# Documentación Técnica: Despliegue de Servidor Proxmox VE y Almacenamiento en 3 Niveles

## 1. Especificaciones del Hardware y Arquitectura de Almacenamiento

* **Procesador:** Intel Core i3-12100 (4 núcleos / 8 hilos).
* **Placa Madre y Fuente:** MSI H610M con fuente de alimentación de 600W.
* **Memoria RAM:** 32 GB DDR4.
* **Diseño de Almacenamiento en 3 Niveles:**
  * **Nivel 1 (Sistema Operativo y Servicios Críticos):** SSD M.2 NVMe de 512 GB (`/dev/nvme0n1`) destinado a Proxmox VE, contenedores LXC y máquinas virtuales principales.
  * **Nivel 2 (Laboratorios e Imágenes ISO):** SSD SATA de 500 GB (`/dev/sdc`) montado como directorio dedicado (`LABS-SSD`).
  * **Nivel 3 (Almacenamiento Masivo y Backups):** 3 discos duros mecánicos WD de 1 TB a 7200 RPM (`WDC_WD10EZEX`) configurados en un arreglo **ZFS RAIDZ1** (`HDD-raidz1`), dejando ~2 TB utilizables y tolerancia a fallos de 1 disco.

---

## 2. Fase 1: Preparación del Firmware UEFI/BIOS (MSI H610M)

Antes de instalar el hipervisor, se entró al modo avanzado de la BIOS (`Supr` + `F7`) para dejar configurados los parámetros base de servidor:

1. **Virtualización en CPU (`OC` → `CPU Features`):**
   * **Intel Virtualization Tech (VT-x):** `Enabled`, necesario para correr máquinas virtuales KVM de forma nativa sobre el procesador.
   * **Intel VT-d Tech (IOMMU):** `Enabled`, para habilitar el mapeo directo de dispositivos físicos y aislamiento de entrada/salida.
2. **Controladora de discos y periféricos (`Settings` → `Advanced` → `Integrated Peripherals`):**
   * **SATA Mode:** `AHCI` (con Intel VMD desactivado) para que el kernel de Linux vea cada disco físico por separado y sin intermediarios antes de armar el ZFS.
   * **HD Audio Controller:** `Disabled`, ya que la placa de sonido integrada no tiene uso en el servidor y apagarla libera recursos e interrupciones del bus.
3. **Gestión de energía (`Settings` → `Advanced` → `Power Management Setup`):**
   * **Restore after AC Power Loss:** `Power On`, para que el equipo vuelva a arrancar solo apenas regrese la luz después de un corte.
4. **Modo de arranque (`Settings` → `Boot` / `Security`):**
   * **Boot Mode Select:** `UEFI`.
   * **Secure Boot:** `Disabled`, evitando bloqueos al cargar el kernel de Proxmox o los módulos de ZFS.

---

## 3. Fase 2: Instalación de Proxmox VE en el M.2 NVMe (Nivel 1)

1. **Armado del USB booteable:** Se descargó la ISO oficial de Proxmox VE y se grabó en un pendrive usando BalenaEtcher / Rufus en **modo DD**.
2. **Elección del disco del sistema (`Target Harddisk`):** Se inició el instalador gráfico (**Install Proxmox VE Graphical**), se aceptó la licencia y se seleccionó únicamente el **M.2 NVMe de 512 GB** con el sistema de archivos por defecto (**`ext4`**).
3. **Particionado y volúmenes LVM creados por el instalador:** El instalador armó tres particiones en el NVMe (1 MB para *BIOS Boot*, 1 GB para la partición *EFI* y el resto para LVM) dentro del grupo de volúmenes `pve`:
   * **`root` (`local` en `/var/lib/vz`):** Formateado en `ext4` para guardar el sistema base, plantillas de contenedores e imágenes ISO.
   * **`swap`:** Partición de memoria de intercambio.
   * **`data` (`local-lvm`):** Volumen *LVM-thin* (aprovisionamiento fino) donde se guardan los discos de las VMs y contenedores LXC, permitiendo sacar *snapshots* al instante sin duplicar el espacio ocupado.
4. **Configuración de red del nodo:** Se definió la ubicación (`Argentina`), la contraseña del usuario `root` y una IP estática sobre el puente de red **`vmbr0`**, que funciona como un switch virtual conectando las VMs y contenedores con la placa de red física enchufada al módem de Movistar.

---

## 4. Fase 3: Post-Instalación, Repositorios y Auditoría del Host

1. **Ingreso a la interfaz web:** Desde el navegador de otra PC en la red se entró a `https://IP-PROXMOX:8006` usando el usuario `root` (autenticación PAM).

[image]

2. **Cambio de repositorios APT:** Como Proxmox trae por defecto los repositorios pagos (`enterprise.proxmox.com`), se deshabilitaron desde `pve` → `Updates` → `Repositories` (tanto el de PVE como el de Ceph) y se agregó el repositorio **`No-Subscription`** para poder bajar actualizaciones y parches sin errores.

[image]

3. **Actualización y chequeo por consola (`>_ Shell`):**
```bash
# Actualizar todos los paquetes del sistema y el kernel de Proxmox
apt update && apt full-upgrade -y

# Comprobar que IOMMU (Intel VT-d) levantó bien en el kernel
dmesg | grep -e DMAR -e IOMMU

# Revisar que el sistema reconozca los 5 discos físicos (1 NVMe, 1 SSD SATA y 3 HDD SATA)
lsblk
```

---

## 5. Fase 4: Configuración de Almacenamiento (Niveles 2 y 3)

### Nivel 2: SSD SATA de 500 GB para Laboratorios (`LABS-SSD`)
Desde `pve` → `Disks` → `Directory`, se formateó el SSD Kingston (`/dev/sdc`) en **`ext4`** con el nombre **`LABS-SSD`** para alojar ISOs y máquinas virtuales de prueba sin gastar espacio ni escrituras en el NVMe principal. Estos fueron los comandos que corrió Proxmox por detrás según el log de la tarea:

```bash
# Crear la tabla de particiones GPT en /dev/sdc
/sbin/sgdisk -n1 -t1:8300 /dev/sdc

# Darle formato ext4 a la partición /dev/sdc1
/sbin/mkfs -t ext4 /dev/sdc1

# Leer el UUID del disco y dejar creado el servicio de montaje en /mnt/pve/LABS-SSD
/sbin/blkid /dev/sdc1 -o export
```

### Nivel 3: 3x HDD de 1 TB en ZFS RAIDZ1 (`HDD-raidz1`) y Backups (`hdd-backups`)

1. **Armado del pool ZFS:** En `pve` → `Disks` → `ZFS`, se agruparon los tres discos Western Digital de 1 TB bajo **RAIDZ1** (1 disco de paridad), activando la compresión **`lz4`** y `ashift=12` (para sectores físicos de 4K). Proxmox armó el arreglo referenciando el ID físico de cada disco (`/dev/disk/by-id/`) para que no falle si algún día cambian de puerto SATA en la placa madre:

```bash
/sbin/zpool create -o ashift=12 HDD-raidz1 raidz \
  /dev/disk/by-id/ata-WDC_WD10EZEX-08WN4A0_WD-WCC6Y4SEYD2A \
  /dev/disk/by-id/ata-WDC_WD10EZEX-08WN4A0_WD-WCC6Y4XFY97C \
  /dev/disk/by-id/ata-WDC_WD10EZEX-08WN4A0_WD-WCC6Y7EEE5NK

/sbin/zfs set compression=lz4 HDD-raidz1
systemctl enable zfs-import@HDD\x2draidz1.service
```

2. **Dataset separado para copias de seguridad (`hdd-backups`):** Como Proxmox solo deja usar los pools ZFS directos para discos virtuales, se abrió la terminal del nodo para crear un dataset interno dedicado a backups y luego se lo cargó en `Datacenter` → `Storage` → `Add` → `Directory` apuntando a `/HDD-raidz1/backups` con la opción **`Backup`** (`vzdump`) activada:

```bash
zfs create HDD-raidz1/backups
```

[image]
[image]
[image]

---

## 6. Fase 5: Copias de Seguridad Automatizadas y Política de Retención (vzdump)

Para tener resguardo de los contenedores LXC y las VMs frente a cualquier rotura del NVMe o cambio de configuración que salga mal, se dejó programado el motor de backups de Proxmox apuntando a `hdd-backups` (`/HDD-raidz1/backups`).

### 1. Cómo trabaja por detrás: `pvescheduler`, Modo `snapshot` y Compresión `zstd`
* **Demonio `pvescheduler`:** Todo lo que se programa en el menú de backups se escribe en `/etc/pve/jobs.cfg`. El servicio `pvescheduler` lee ese archivo y lanza `vzdump` a la hora indicada sin tener que tocar `crontab` a mano.
* **Respaldo en caliente (`mode: snapshot`):** En vez de usar `stop` (que apaga el contenedor y te deja sin servicio mientras copia) o `suspend`, se configuró en modo `snapshot`. Aprovechando que los contenedores viven en `local-lvm` (`LVM-thin`), Proxmox saca una foto de solo lectura del volumen en un segundo y copia los bloques hacia los discos mecánicos mientras el contenedor (como el controlador UniFi) sigue andando sin cortes.
* **Compresión `zstd`:** Usa varios hilos del i3-12100 a la vez para comprimir rápido sin clavar el procesador, armando archivos `.tar.zst` (para LXC) y `.vma.zst` (para VMs).

### 2. Regla programada en `Datacenter → Backup`
Se armó la tarea automática con esta configuración:
1. **General:**
   * **Storage:** `hdd-backups`.
   * **Schedule:** `03:30` (todos los días a las 3:30 AM para no frenar la red ni los discos mientras se usa la PC).
   * **Selection mode:** `All` (así cualquier VM o contenedor nuevo que se cree más adelante entra solo en el backup sin tener que editar la tarea).
   * **Compression:** `ZSTD (fast and good)`.
   * **Mode:** `Snapshot`.
2. **Retention (Limpieza automática con `prune-backups`):**
   Para que los backups diarios no llenen los 2 TB del RAID, se destildó *Keep all backups* y se dejó este esquema de rotación:
   * `keep-last = 3` (guarda las últimas 3 copias).
   * `keep-daily = 7` (guarda 1 por día de la última semana).
   * `keep-weekly = 4` (guarda 1 por semana del último mes).
   * `keep-monthly = 2` (guarda 1 por mes de los últimos 2 meses).
3. **Note Template:**
   * Se dejó la etiqueta `{{guestname}}` para que en la lista de archivos aparezca el nombre del contenedor o VM (ej. `unifi-controller`) y no solo el número de ID.

### 3. Prueba manual, bloqueo de copia base y chequeo por terminal
1. **Ejecución con `Run now` y marca `Protected`:** Se seleccionó la tarea recién creada y se corrió a mano con **Run now** para comprobar que escribiera bien en el ZFS. Cuando terminó con `TASK OK`, se entró a `hdd-backups` → `Backups`, se eligió el archivo creado y se le activó **`Change Protection`**. Eso le agrega un archivo `.protected` al lado para que la rotación automática nunca borre esa primera copia limpia.
2. **Comandos de verificación en la consola del nodo (`>_ Shell`):**

```bash
# Ver cómo quedó guardada la tarea programada
cat /etc/pve/jobs.cfg

# Revisar que estén creados el backup (.tar.zst), el log (.log), la nota (.notes) y el bloqueo (.protected)
ls -lh /HDD-raidz1/backups/dump/

# Simular qué copias borraría la regla de retención sin tocar ningún archivo (--dry-run)
pvesm prune-backups hdd-backups --dry-run 1
```

---

## 7. Cheatsheet, Backups y Restauración de ProxmoxVE

[image]

| Comando / Ruta en Consola | Para qué sirve |
| :--- | :--- |
| `/etc/pve/jobs.cfg` | Archivo donde Proxmox guarda las tareas programadas de backup y sus horarios. |
| `/etc/pve/storage.cfg` | Archivo donde quedan declarados los discos y directorios dados de alta (`LABS-SSD`, `HDD-raidz1`, `hdd-backups`). |
| `systemctl status pvescheduler` | Revisa si el servicio que dispara los backups automáticos está corriendo. |
| `ls -lh /HDD-raidz1/backups/dump/` | Lista los archivos físicos de backup en el RAID, sus logs y si tienen activado el `.protected`. |
| `vzdump 100 --storage hdd-backups --mode snapshot --compress zstd` | Lanza a mano desde la terminal un backup en caliente del contenedor o VM `100`. |
| `pvesm prune-backups hdd-backups --dry-run 1` | Muestra en pantalla qué backups se guardan (`keep`) y cuáles se borrarían según la retención configurada. |
| `pct restore 100 /HDD-raidz1/backups/dump/vzdump-lxc-100-....tar.zst --storage local-lvm` | Restaura un contenedor LXC desde el backup mandándolo de vuelta al NVMe (`local-lvm`). |
| `qmrestore /HDD-raidz1/backups/dump/vzdump-qemu-101-....vma.zst 101 --storage local-lvm` | Restaura una máquina virtual desde el backup hacia el NVMe (`local-lvm`). |
