# Detalles Técnicos - GPU Passthrough

## 🔍 Análisis de Hardware

### GPU NVIDIA Configurada
- **IDs PCI**: `10de:11c0`, `10de:0e0b`
- **Bus PCI**: `0000:03:00.0` y `0000:03:00.1`
- **Fabricante**: NVIDIA Corporation
- **Configuración**: GPU principal + función de audio

### Configuración IOMMU
```bash
# Parámetros GRUB utilizados
iommu=1                    # Habilita IOMMU
intel_iommu=on             # Habilita IOMMU específico de Intel
rd.driver.pre=vfio-pci     # Precarga driver VFIO
video=efifb:off            # Deshabilita framebuffer EFI
vfio_pci.ids=10de:11c0,10de:0e0b  # IDs específicos de GPU
```

## 📋 Configuración de Módulos del Kernel

### Módulos Cargados en Boot
```bash
# /etc/modules
vfio                    # Framework VFIO base
vfio_iommu_type1        # Soporte IOMMU tipo 1
vfio_pci ids=10de:11c0,10de:0e0b  # Driver VFIO para GPU específica
```

### Dependencias de Módulos
```bash
# /etc/modprobe.d/nvidiagpu.conf
softdep nvidia pre: vfio vfio_pci  # Carga VFIO antes que NVIDIA
```

### Configuración VFIO
```bash
# /etc/modprobe.d/vfio_pci.conf
options vfio_pci ids=10de:11c0,10de:0e0b  # IDs de GPU
options vfio-pci disable_vga=1            # Deshabilita VGA
```

## 🖥️ Configuración de la Máquina Virtual

### Especificaciones Técnicas
- **Arquitectura**: x86_64
- **Tipo de máquina**: pc-q35-4.2 (Q35 chipset moderno)
- **Firmware**: OVMF (UEFI para VMs)
- **Memoria**: 8GB (8388608 KiB)
- **vCPUs**: 8 (4 cores × 2 threads)
- **Topología**: 1 socket, 4 cores, 2 threads

### Optimizaciones de CPU
```xml
<cpu mode="host-model" check="partial">
  <topology sockets="1" cores="4" threads="2"/>
</cpu>
```

### Características de Hyper-V
```xml
<hyperv>
  <relaxed state="on"/>
  <vapic state="on"/>
  <spinlocks state="on" retries="8191"/>
  <vendor_id state="on" value="whatever"/>
</hyperv>
```

### Configuración de GPU Passthrough
```xml
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x03" slot="0x00" function="0x1"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
</hostdev>
```

## 🔊 Configuración de Audio

### Parámetros QEMU para Audio
```bash
QEMU_AUDIO_DRV=alsa
QEMU_ALSA_DAC_BUFFER_SIZE=512
QEMU_ALSA_DAC_PERIOD_SIZE=170
QEMU_PA_SAMPLES=1024
QEMU_AUDIO_TIMER_PERIOD=99
QEMU_PA_SERVER=/run/user/1000/pulse/native
```

### Configuración de Audio en VM
```xml
<sound model="ac97">
  <address type="pci" domain="0x0000" bus="0x03" slot="0x01" function="0x0"/>
</sound>
```

## 🌐 Configuración de Red

### Interfaces de Red Configuradas
1. **Red NAT**: Para conectividad básica
2. **Bridge directo**: Para conectividad de alto rendimiento

```xml
<interface type="network">
  <mac address="52:54:00:7a:90:b9"/>
  <source network="default"/>
  <model type="e1000e"/>
</interface>
<interface type="direct">
  <mac address="52:54:00:14:c8:8c"/>
  <source dev="usb0" mode="bridge"/>
  <model type="e1000e"/>
</interface>
```

## 🔧 Scripts y Automatización

### Script de Desconexión de GPU
```bash
#!/bin/sh
# detachCard2.sh
virsh nodedev-detach pci_0000_03_00_0  # GPU principal
virsh nodedev-detach pci_0000_03_00_1  # Función de audio
```

### Servicio Systemd
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

## 🚀 Optimizaciones de Rendimiento

### Configuraciones QEMU
- **Emulador**: `/usr/bin/qemu-system-x86_64`
- **Tipo de disco**: qcow2 (formato eficiente)
- **Controladores**: PCIe modernos con múltiples puertos
- **Memoria balloon**: VirtIO para gestión dinámica de memoria

### Configuraciones de Gráficos
```xml
<graphics type="spice" autoport="yes">
  <listen type="address"/>
  <image compression="off"/>
  <gl enable="no" rendernode="/dev/dri/by-path/pci-0000:02:00.0-render"/>
</graphics>
```

## 🔍 Diagnóstico y Troubleshooting

### Comandos de Verificación
```bash
# Verificar dispositivos PCI
lspci -nn -D | grep NVIDIA

# Verificar grupos IOMMU
find /sys/kernel/iommu_groups/ -type l

# Verificar estado VFIO
lsmod | grep vfio

# Verificar dispositivos libvirt
virsh nodedev-list --tree

# Verificar logs del servicio
journalctl -u lend-card.service
```

### Problemas Comunes

1. **Error 43 en Windows**: GPU detectada como virtual
   - Solución: Usar `kvm.hidden=on` en configuración

2. **GPU no se desconecta**: Problema con script de detach
   - Verificar permisos y rutas del script

3. **Audio no funciona**: Problema con passthrough de audio
   - Verificar configuración ALSA y PulseAudio

4. **Rendimiento bajo**: Configuración subóptima
   - Verificar CPU pinning y optimizaciones Hyper-V

## 📊 Métricas de Rendimiento

### Benchmarks Esperados
- **GPU Performance**: 95-98% del rendimiento nativo
- **Latencia**: <5ms adicional
- **CPU Overhead**: <5% en juegos GPU-intensivos
- **Memory Overhead**: ~100MB adicional para VFIO

### Monitoreo
```bash
# Monitorear uso de GPU en host
nvidia-smi

# Monitorear uso de CPU
htop

# Monitorear memoria
free -h

# Monitorear red
iftop
```

## 🔒 Consideraciones de Seguridad

1. **Aislamiento**: GPU completamente aislada en VM
2. **Permisos**: Scripts ejecutados con privilegios mínimos
3. **Logs**: Actividad registrada en systemd journal
4. **Backup**: Configuraciones críticas respaldadas

## 📚 Referencias Técnicas

- [Documentación VFIO](https://www.kernel.org/doc/html/latest/driver-api/vfio.html)
- [Guía KVM GPU Passthrough](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
- [Documentación QEMU](https://qemu.readthedocs.io/)
- [Libvirt XML Schema](https://libvirt.org/formatdomain.html) 