# Guía de Instalación - GPU Passthrough

## 📋 Prerrequisitos

### Hardware Requerido
- **CPU**: Intel con soporte IOMMU (VT-d)
- **GPU**: NVIDIA (en este caso IDs: `10de:11c0`, `10de:0e0b`)
- **GPU secundaria**: Para el sistema host (integrada o dedicada)
- **RAM**: Mínimo 16GB (8GB para VM + 8GB para host)
- **Almacenamiento**: SSD recomendado para mejor rendimiento

### Software Requerido
- **Sistema Operativo**: Linux (Ubuntu/Debian recomendado)
- **KVM/QEMU**: Sistema de virtualización
- **Libvirt**: Gestión de máquinas virtuales
- **OVMF**: Firmware UEFI para VMs

## 🔧 Instalación Paso a Paso

### Paso 1: Verificar Compatibilidad

```bash
# Verificar soporte de virtualización
egrep -c '(vmx|svm)' /proc/cpuinfo

# Verificar soporte IOMMU
dmesg | grep -i iommu

# Verificar GPU NVIDIA
lspci -nn -D | grep NVIDIA
```

**Resultado esperado**: Debe mostrar al menos 1 para virtualización y mensajes de IOMMU.

### Paso 2: Instalar Dependencias

```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar KVM y herramientas
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils

# Instalar herramientas adicionales
sudo apt install -y virt-manager ovmf

# Agregar usuario al grupo libvirt
sudo usermod -a -G libvirt $USER
sudo usermod -a -G kvm $USER
```

### Paso 3: Configurar GRUB

```bash
# Editar configuración GRUB
sudo nano /etc/default/grub
```

**Agregar/modificar la línea**:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=1 intel_iommu=on rd.driver.pre=vfio-pci video=efifb:off vfio_pci.ids=10de:11c0,10de:0e0b"
```

### Paso 4: Configurar Módulos del Kernel

```bash
# Editar /etc/modules
sudo nano /etc/modules
```

**Agregar**:
```
vfio
vfio_iommu_type1
vfio_pci ids=10de:11c0,10de:0e0b
```

### Paso 5: Configurar VFIO

```bash
# Crear archivo de configuración NVIDIA
sudo nano /etc/modprobe.d/nvidiagpu.conf
```

**Contenido**:
```
softdep nvidia pre: vfio vfio_pci
```

```bash
# Crear archivo de configuración VFIO
sudo nano /etc/modprobe.d/vfio_pci.conf
```

**Contenido**:
```
options vfio_pci ids=10de:11c0,10de:0e0b
options vfio-pci disable_vga=1
```

### Paso 6: Configurar Initramfs

```bash
# Editar configuración de initramfs
sudo nano /etc/initramfs-tools/modules
```

**Agregar**:
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

### Paso 7: Crear Script de Desconexión

```bash
# Crear directorio si no existe
sudo mkdir -p /usr/local/sbin

# Crear script
sudo nano /usr/local/sbin/detachCard2.sh
```

**Contenido**:
```bash
#!/bin/sh
virsh nodedev-detach pci_0000_03_00_0 
virsh nodedev-detach pci_0000_03_00_1
```

```bash
# Hacer ejecutable
sudo chmod +x /usr/local/sbin/detachCard2.sh
```

### Paso 8: Crear Servicio Systemd

```bash
# Crear archivo de servicio
sudo nano /etc/systemd/system/lend-card.service
```

**Contenido**:
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

```bash
# Habilitar servicio
sudo systemctl enable lend-card
```

### Paso 9: Aplicar Configuración

```bash
# Actualizar GRUB
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Actualizar initramfs
sudo update-initramfs -k all -c

# Reiniciar sistema
sudo reboot
```

## 🖥️ Configuración de la Máquina Virtual

### Paso 10: Crear VM Windows 10

```bash
# Usar virt-manager o crear VM desde línea de comandos
virt-install \
  --name win10 \
  --memory 8192 \
  --vcpus 8 \
  --disk path=/path/to/win10.qcow2,size=50 \
  --os-variant win10 \
  --network network=default \
  --graphics spice \
  --video qxl \
  --boot uefi
```

### Paso 11: Configurar GPU Passthrough

1. **Editar configuración de VM**:
```bash
virsh edit win10
```

2. **Agregar configuración de GPU** (ver `win10.xml` para ejemplo completo):
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

## ✅ Verificación de Instalación

### Verificar Configuración

```bash
# Verificar módulos VFIO cargados
lsmod | grep vfio

# Verificar dispositivos PCI
lspci -nn -D | grep NVIDIA

# Verificar grupos IOMMU
find /sys/kernel/iommu_groups/ -type l

# Verificar estado del servicio
systemctl status lend-card
```

### Verificar VM

```bash
# Listar VMs
virsh list --all

# Verificar configuración de VM
virsh dumpxml win10 | grep -A 10 -B 10 hostdev
```

## 🎮 Configuración de Windows 10

### Instalación de Drivers

1. **Instalar drivers NVIDIA** en Windows 10
2. **Configurar audio** si es necesario
3. **Optimizar para gaming**:
   - Deshabilitar Windows Game Mode
   - Configurar plan de energía de alto rendimiento
   - Deshabilitar actualizaciones automáticas

### Optimizaciones Adicionales

```bash
# En Windows, ejecutar como administrador:
# Deshabilitar servicios innecesarios
# Configurar prioridad de procesos
# Optimizar configuración de red
```

## 🔧 Troubleshooting

### Problemas Comunes

1. **Error 43 en Windows**:
   - Agregar `<kvm><hidden state="on"/></kvm>` en configuración VM

2. **GPU no se desconecta**:
   - Verificar permisos del script
   - Verificar que el servicio esté habilitado

3. **Audio no funciona**:
   - Verificar configuración ALSA
   - Configurar passthrough de audio

4. **Rendimiento bajo**:
   - Verificar configuración de CPU
   - Optimizar configuración de memoria

### Comandos de Diagnóstico

```bash
# Ver logs del servicio
journalctl -u lend-card.service

# Ver logs de libvirt
journalctl -u libvirtd

# Verificar dispositivos libvirt
virsh nodedev-list --tree

# Verificar configuración IOMMU
dmesg | grep -i iommu
```

## 🎯 Próximos Pasos

1. **Instalar juegos** en Windows 10
2. **Configurar controles** si es necesario
3. **Optimizar rendimiento** según necesidades
4. **Configurar backup** de la VM
5. **Documentar configuraciones específicas**

## 📞 Soporte

Si encuentras problemas:
1. Revisar logs del sistema
2. Verificar compatibilidad de hardware
3. Consultar documentación técnica
4. Buscar ayuda en foros de la comunidad

---

**⚠️ Importante**: Esta configuración es específica para el hardware documentado. Adapta los IDs de GPU y configuraciones según tu hardware específico. 