# 🏗️ Homelab & Infraestructura On-Premise: Proxmox VE, Redes, Docker, IaC y SecOps

Este repositorio documenta el diseño, armado desde cero y administración de mi servidor en casa (Homelab). El objetivo del proyecto es llevar a la práctica real los conceptos de administración de sistemas Linux, redes, contenedores, automatización (Infraestructura como Código) y seguridad defensiva, operando bajo estándares de producción. Asi mismo, si alguien quiere replicar una arquitectura parecida o tiene alguna sugerencia, es mas que bienvenida a realizarla!

En este mismo repositorio estan tanto las **guías técnicas paso a paso** (con el porqué de cada configuración y comandos de auditoría) como los **archivos de código desplegables** (`docker-compose.yml`, playbooks de Ansible y plantillas de aprovisionamiento) que corren en el servidor.

---

## 🖥️ Especificaciones del Servidor y Almacenamiento en 3 Niveles

Para evitar cuellos de botella de entrada/salida (I/O) y separar los servicios críticos de la casa del entorno de pruebas, el almacenamiento físico se dividió en tres niveles independientes dentro de **Proxmox VE**:

* **CPU:** Intel Core i3-12100 (4 Cores / 8 Threads, con gráficos integrados Intel UHD 730).
* **Motherboard y Fuente:** MSI H610M + Fuente 600W.
* **Memoria RAM:** 32 GB DDR4 (2666 MHz).
* **Red:** Gigabit Ethernet en puente (`vmbr0`) + Punto de Acceso corporativo **Ubiquiti UniFi UAP-AC-LR**.

| Nivel | Unidad Física | Formato / Tipo en Proxmox | Uso Asignado en el Servidor |
| :--- | :--- | :--- | :--- |
| **Nivel 1 (Producción)** | M.2 NVMe 512 GB | `ext4` (`local`) + `LVM-thin` (`local-lvm`) | Sistema base Proxmox VE, contenedores LXC y VMs de servicios críticos 24/7 (UniFi, Docker, DNS, Monitoreo). |
| **Nivel 2 (Laboratorio)** | SSD SATA 500 GB | `ext4` Directory (`LABS-SSD`) | Imágenes ISO, plantillas Cloud-Init y máquinas virtuales de prueba (Ansible, Terraform, Kubernetes K3s) sin desgastar el NVMe principal. |
| **Nivel 3 (Datos y Backups)** | 3x HDD 1 TB (7200 RPM) | Pool **ZFS RAIDZ1** (`HDD-raidz1` / `hdd-backups`) | ~2 TB con tolerancia al fallo físico de 1 disco y compresión `lz4`. Aloja archivos pesados y los backups automáticos nocturnos (`vzdump` + `zstd`). |

---

## 🗺️ Roadmap del Proyecto (De Cero a Cloud / DevOps)

El proyecto avanza en **6 etapas progresivas**. Cada etapa se da por cerrada únicamente cuando el servicio está funcionando en el servidor, asegurado y documentado en este repositorio.

| Etapa | Qué se construye en el servidor | Dónde se aloja | Habilidades Técnicas Aplicadas | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **1. Base, Red y Backups** | Proxmox VE + Almacenamiento 3 Niveles (ZFS RAIDZ1) + Controlador UniFi LXC + Backups automáticos | NVMe (`local-lvm`) y ZFS (`hdd-backups`) | Ajuste de BIOS para servidores, hipervisor KVM/LXC, administración ZFS/LVM, radiofrecuencia Wi-Fi y retención de copias (`vzdump` / `prune-backups`). | ✅ Completado |
| **2. Contenedores y GitOps** | VM Debian + Docker + Compose + Portainer + **Pi-hole & Unbound** | NVMe (`local-lvm`) | Aislamiento de contenedores en VM dedicada, redes Docker, DNS recursivo propio (sin depender de DNS externos) y despliegue continuo desde GitHub. | 🔄 En progreso |
| **3. Red Zero Trust** | **Cloudflare Tunnel (`cloudflared`)** + Proxy Inverso + Dashboard Web | VM Docker en NVMe (`local-lvm`) | Publicación segura de servicios hacia un dominio propio con certificados SSL/TLS sin abrir puertos en el router del ISP (evitando exposición de IP y NAT). | ⏳ Pendiente |
| **4. Observabilidad** | **Prometheus + Grafana + Uptime Kuma** (o Zabbix) | NVMe (`local-lvm`) / `LABS-SSD` | Recolección de métricas en tiempo real (CPU, RAM, salud del pool ZFS, tráfico de red) y alertas automatizadas ante caídas de servicios. | ⏳ Pendiente |
| **5. Automatización e IaC** | Plantilla **Cloud-Init** + **Terraform / OpenTofu** + **Ansible** | SSD SATA (`LABS-SSD`) | Creación automatizada de VMs por API y configuración desatendida con hardening Linux (`nftables`, SSH seguro, usuarios sin privilegios). | ⏳ Pendiente |
| **6. Orquestación y SecOps** | Clúster **Kubernetes (K3s)** y/o SIEM **Wazuh** | SSD SATA (`LABS-SSD`) | Orquestación de microservicios (Pods, Deployments, Ingress) y monitoreo de seguridad / auditoría de logs centralizada. | ⏳ Pendiente |

---

## 📂 Estructura del Repositorio e Índice de Documentación

A medida que se completan las etapas del Roadmap, el repositorio se organiza separando las bitácoras de implementación de los archivos de configuración y despliegue:

```text
├── 01-proxmox-base/          # Documentación de BIOS, Proxmox VE, ZFS RAIDZ1 y Backups
├── 02-unifi-controller-lxc/  # Despliegue de controlador UniFi en LXC y ajuste RF del AP
├── docker-stacks/            # Archivos docker-compose.yml (Pi-hole, Unbound, Cloudflare, etc.)
├── iac-automatizacion/       # Playbooks de Ansible, scripts Bash y código Terraform (Etapa 5)
└── README.md                 # Portada general y Roadmap del proyecto
```

### 📘 Etapas Documentadas hasta el momento:
* **[Etapa 01.A — Despliegue de Proxmox VE, Almacenamiento en 3 Niveles y Backups Automatizados]:** Preparación de BIOS/UEFI, particionado LVM-thin en NVMe, montaje de SSD secundario, creación de pool ZFS RAIDZ1 por ID físico de disco y programación de respaldos en caliente (`snapshot` + `zstd`).
* **Etapa 01.B — Controlador UniFi Network en Contenedor LXC y Radiofrecuencia:** Aprovisionamiento de contenedor Debian 12 sin privilegios, recuperación de AP Ubiquiti 3 UAP-AC-LR desde modo TFTP, adopción en Capa 2 y ajuste de canales y potencia para cobertura de 250 m².

---

## Checklist de Avance Detallado

- [x] **Etapa 1:** Configuración de BIOS MSI H610M (VT-x, VT-d, AHCI, Power On tras corte eléctrico).
- [x] **Etapa 1:** Instalación de Proxmox VE en NVMe (`local-lvm`), alta de `LABS-SSD` y creación de pool ZFS `HDD-raidz1`.
- [x] **Etapa 1:** Creación de contenedor LXC Debian 12, instalación de UniFi Network Server y adopción de AP UAP-AC-LR.
- [x] **Etapa 1:** Programación de backups nocturnos (`vzdump`) con rotación GFS (`prune-backups`) y copia base protegida.
- [ ] **Etapa 2:** Creación y hardening de VM Debian (`docker-prod`) con Docker Engine y Portainer.
- [ ] **Etapa 2:** Despliegue de stack `Pi-hole + Unbound` desde este repositorio mediante GitOps.
- [ ] **Etapa 3:** Configuración de dominio propio y túnel cifrado saliente con Cloudflare Zero Trust.
- [ ] **Etapa 4:** Despliegue de stack de monitoreo y alertas de estado.
- [ ] **Etapa 5:** Aprovisionamiento automatizado de laboratorio con Cloud-Init, Terraform y Ansible.
- [ ] **Etapa 6:** Despliegue de clúster K3s y auditoría de seguridad con Wazuh.

---
## 🤝 Contacto

Siéntete libre de utilizar estos apuntes y Cheat Sheets como material de consulta rápida para tu día a día en la terminal.

🔗 **Conectemos en LinkedIn:** https://www.linkedin.com/in/lucca-martinenghi-it/


>
> "Talk is cheap. Show me the code." — Linus Torvalds
>
<img width="220" height="220" alt="tux" src="https://github.com/user-attachments/assets/800b49d8-1684-4aa4-a4ac-a9525755afa4" />

