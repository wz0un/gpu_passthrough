# GPU Passthrough Setup for KVM

Este proyecto contiene la configuración completa para implementar **GPU Passthrough** en KVM, permitiendo ejecutar Windows 10 en una máquina virtual con rendimiento nativo de GPU para gaming.

## 🎮 ¿Qué es GPU Passthrough?

GPU Passthrough es una técnica de virtualización que permite asignar una GPU física directamente a una máquina virtual, evitando la capa de virtualización del gráfico. Esto resulta en rendimiento prácticamente nativo, ideal para gaming en máquinas virtuales.

## 🖥️ Hardware Configurado

- **GPU**: NVIDIA (IDs: `10de:11c0`, `10de:0e0b`)
- **CPU**: Intel con soporte IOMMU
- **Sistema**: Linux con KVM/QEMU
- **VM**: Windows 10

## 📁 Estructura del Proyecto

```
gpu_passthrough/
├── README.md                    # Esta documentación
├── detachCard2.sh              # Script para desconectar GPU del host
├── lend-card.service           # Servicio systemd para automatizar el proceso
├── win10.xml                   # Configuración de la VM Windows 10
├── nvidiagpu.conf              # Configuración de módulos NVIDIA
├── vfio_pci.conf               # Configuración VFIO para GPU
├── etc-modules                 # Módulos del kernel a cargar
├── grub                        # Configuración GRUB con parámetros IOMMU
├── modules                     # Configuración de initramfs
└── history commands in bash.txt # Comandos de instalación
```

## 🚀 Instalación y Configuración

### 1. Verificar Compatibilidad

```bash
# Verificar soporte IOMMU
lspci -nn -D | grep NVIDIA
dmesg | grep -i iommu
```

### 2. Configurar GRUB

Editar `/etc/default/grub` y agregar los parámetros IOMMU:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=1 intel_iommu=on rd.driver.pre=vfio-pci video=efifb:off vfio_pci.ids=10de:11c0,10de:0e0b"
```

### 3. Configurar Módulos del Kernel

**Archivo: `/etc/modules`**
```
vfio
vfio_iommu_type1
vfio_pci ids=10de:11c0,10de:0e0b
```

**Archivo: `/etc/modprobe.d/nvidiagpu.conf`**
```
softdep nvidia pre: vfio vfio_pci
```

**Archivo: `/etc/modprobe.d/vfio_pci.conf`**
```
options vfio_pci ids=10de:11c0,10de:0e0b
options vfio-pci disable_vga=1
```

### 4. Configurar Initramfs

**Archivo: `/etc/initramfs-tools/modules`**
```
softdep nvidia pre: vfio vfio_pci
vfio
vfio_iommu_type1
vfio_virqfd
options vfio_pci ids=10de:11c0,10de:0e0b
vfio_pci ids=10de:11c0,10de:0e0b
vfio_pci
nvidia
```

### 5. Script de Desconexión de GPU

**Archivo: `/usr/local/sbin/detachCard2.sh`**
```bash
#!/bin/sh
virsh nodedev-detach pci_0000_03_00_0 
virsh nodedev-detach pci_0000_03_00_1
```

### 6. Servicio Systemd

**Archivo: `/etc/systemd/system/lend-card.service`**
```ini
[Unit]
Description=Drop one video card so that libvirt can use it for GPU-passthrough.
After=libvirt-bin.service
Before=display-manager.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/detachCard2.sh

[Install]
WantedBy=multi-user.target
```

### 7. Aplicar Configuración

```bash
# Hacer ejecutable el script
sudo chmod +x /usr/local/sbin/detachCard2.sh

# Habilitar el servicio
sudo systemctl enable lend-card

# Actualizar GRUB e initramfs
sudo grub-mkconfig
sudo update-grub
sudo update-initramfs -k all -c

# Reiniciar
sudo reboot
```

## 🎯 Configuración de la Máquina Virtual

El archivo `win10.xml` contiene la configuración completa de la VM con:

- **8GB RAM** asignada
- **8 vCPUs** (4 cores, 2 threads)
- **GPU passthrough** configurado
- **UEFI/OVMF** para boot moderno
- **Optimizaciones Hyper-V** habilitadas
- **Audio ALSA** configurado

### Características Destacadas:

- **Hostdev PCI**: Asignación directa de GPU
- **QEMU optimizations**: Configuraciones para mejor rendimiento
- **Audio passthrough**: Audio directo desde host
- **Network bridging**: Conectividad de red optimizada

## 🔧 Comandos Útiles

```bash
# Verificar dispositivos PCI
lspci -nn -D | grep NVIDIA

# Verificar estado IOMMU
dmesg | grep -i iommu

# Listar dispositivos libvirt
virsh nodedev-list

# Verificar estado del servicio
systemctl status lend-card

# Iniciar/detener VM
virsh start win10
virsh shutdown win10
```

## ⚠️ Consideraciones Importantes

1. **Compatibilidad**: Verifica que tu hardware soporte IOMMU
2. **GPU dedicada**: Necesitas una GPU secundaria para el host
3. **Drivers**: Instala drivers NVIDIA en la VM
4. **Backup**: Haz backup antes de modificar configuraciones críticas
5. **IDs específicos**: Los IDs de GPU (`10de:11c0`, `10de:0e0b`) son específicos para esta configuración

## 🎮 Gaming Performance

Con esta configuración obtienes:
- ✅ Rendimiento GPU nativo
- ✅ Soporte completo de DirectX/Vulkan
- ✅ Latencia mínima
- ✅ Compatibilidad con la mayoría de juegos

## 📝 Notas del Autor

Esta configuración fue desarrollada basándose en guías públicas de internet, adaptadas específicamente para una GPU NVIDIA con IDs `10de:11c0` y `10de:0e0b`. Los archivos están optimizados para un sistema Intel con soporte IOMMU.

## 🤝 Contribuciones

Si encuentras problemas o mejoras, no dudes en abrir un issue o pull request.

---

**⚠️ Disclaimer**: Esta configuración es específica para el hardware del autor. Adapta los IDs de GPU y configuraciones según tu hardware específico.