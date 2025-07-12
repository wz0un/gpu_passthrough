# Ejemplos de Configuración - GPU Passthrough

## 🎯 Configuraciones por Tipo de Hardware

### Configuración Actual (NVIDIA)
- **GPU**: NVIDIA (IDs: `10de:11c0`, `10de:0e0b`)
- **CPU**: Intel con IOMMU
- **Sistema**: Linux con KVM

## 🖥️ Configuraciones Alternativas

### 1. GPU AMD Radeon

#### Configuración GRUB
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=1 amd_iommu=on rd.driver.pre=vfio-pci video=efifb:off vfio_pci.ids=1002:67df,1002:aaf0"
```

#### Archivo `/etc/modules`
```
vfio
vfio_iommu_type1
vfio_pci ids=1002:67df,1002:aaf0
```

#### Archivo `/etc/modprobe.d/amdgpu.conf`
```
softdep amdgpu pre: vfio vfio_pci
softdep radeon pre: vfio vfio_pci
```

#### Archivo `/etc/modprobe.d/vfio_pci.conf`
```
options vfio_pci ids=1002:67df,1002:aaf0
options vfio-pci disable_vga=1
```

### 2. GPU NVIDIA Diferente

#### Para GTX 1080 (IDs: `10de:1b80`, `10de:10f0`)
```bash
# GRUB
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=1 intel_iommu=on rd.driver.pre=vfio-pci video=efifb:off vfio_pci.ids=10de:1b80,10de:10f0"

# /etc/modules
vfio
vfio_iommu_type1
vfio_pci ids=10de:1b80,10de:10f0

# /etc/modprobe.d/vfio_pci.conf
options vfio_pci ids=10de:1b80,10de:10f0
options vfio-pci disable_vga=1
```

#### Para RTX 3080 (IDs: `10de:2206`, `10de:1aef`)
```bash
# GRUB
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=1 intel_iommu=on rd.driver.pre=vfio-pci video=efifb:off vfio_pci.ids=10de:2206,10de:1aef"

# /etc/modules
vfio
vfio_iommu_type1
vfio_pci ids=10de:2206,10de:1aef

# /etc/modprobe.d/vfio_pci.conf
options vfio_pci ids=10de:2206,10de:1aef
options vfio-pci disable_vga=1
```

### 3. CPU AMD Ryzen

#### Configuración GRUB para AMD
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=1 amd_iommu=on rd.driver.pre=vfio-pci video=efifb:off vfio_pci.ids=10de:11c0,10de:0e0b"
```

#### Archivo `/etc/modules` para AMD
```
vfio
vfio_iommu_type1
vfio_pci ids=10de:11c0,10de:0e0b
```

### 4. Configuración con Múltiples GPUs

#### Para 2 GPUs NVIDIA
```bash
# GRUB
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=1 intel_iommu=on rd.driver.pre=vfio-pci video=efifb:off vfio_pci.ids=10de:11c0,10de:0e0b,10de:1b80,10de:10f0"

# /etc/modules
vfio
vfio_iommu_type1
vfio_pci ids=10de:11c0,10de:0e0b,10de:1b80,10de:10f0

# /etc/modprobe.d/vfio_pci.conf
options vfio_pci ids=10de:11c0,10de:0e0b,10de:1b80,10de:10f0
options vfio-pci disable_vga=1
```

## 🔧 Scripts de Desconexión por Hardware

### Script para GPU NVIDIA (actual)
```bash
#!/bin/sh
# detachCard2.sh
virsh nodedev-detach pci_0000_03_00_0 
virsh nodedev-detach pci_0000_03_00_1
```

### Script para GPU AMD
```bash
#!/bin/sh
# detachCard_amd.sh
virsh nodedev-detach pci_0000_01_00_0
virsh nodedev-detach pci_0000_01_00_1
```

### Script para Múltiples GPUs
```bash
#!/bin/sh
# detachCard_multi.sh
# GPU 1
virsh nodedev-detach pci_0000_03_00_0 
virsh nodedev-detach pci_0000_03_00_1
# GPU 2
virsh nodedev-detach pci_0000_04_00_0
virsh nodedev-detach pci_0000_04_00_1
```

## 🖥️ Configuraciones de VM por Hardware

### VM para GPU NVIDIA
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

### VM para GPU AMD
```xml
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
</hostdev>
```

### VM para Múltiples GPUs
```xml
<!-- GPU 1 -->
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
<!-- GPU 2 -->
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x08" slot="0x00" function="0x0"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x04" slot="0x00" function="0x1"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x09" slot="0x00" function="0x0"/>
</hostdev>
```

## 🔍 Cómo Encontrar IDs de GPU

### Comando para Listar GPUs
```bash
# Listar todas las GPUs
lspci -nn | grep -i vga

# Listar GPUs NVIDIA específicamente
lspci -nn | grep -i nvidia

# Listar GPUs AMD específicamente
lspci -nn | grep -i amd

# Información detallada de GPU
lspci -nnv | grep -A 10 -B 2 -i vga
```

### Ejemplo de Salida
```bash
$ lspci -nn | grep -i nvidia
03:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP104 [GeForce GTX 1070] [10de:1b81] (rev a1)
03:00.1 Audio device [0403]: NVIDIA Corporation GP104 High Definition Audio Controller [10de:10f0] (rev a1)
```

**IDs extraídos**: `10de:1b81`, `10de:10f0`

## 🛠️ Configuraciones Especiales

### Para Sistemas con GPU Integrada + Dedicada
```bash
# Usar GPU integrada para host, dedicada para VM
# Configurar X11 para usar GPU integrada
export DISPLAY=:0
xrandr --listproviders
```

### Para Sistemas Headless
```bash
# Configurar para ejecutar sin monitor
# Agregar en GRUB
video=efifb:off nomodeset
```

### Para Optimización de Rendimiento
```bash
# Configuraciones adicionales en GRUB
intel_pstate=performance
cpufreq.default_governor=performance
```

## 📋 Checklist de Adaptación

### Para Adaptar a Tu Hardware:

1. **Identificar GPU**:
   ```bash
   lspci -nn | grep -i vga
   ```

2. **Extraer IDs**:
   - Formato: `[vendor:device]`
   - Ejemplo: `10de:1b81` → `10de:1b81`

3. **Verificar CPU**:
   ```bash
   # Para Intel
   dmesg | grep -i intel_iommu
   
   # Para AMD
   dmesg | grep -i amd_iommu
   ```

4. **Adaptar Configuraciones**:
   - Reemplazar IDs en todos los archivos
   - Ajustar parámetros IOMMU según CPU
   - Modificar scripts de desconexión

5. **Verificar Grupos IOMMU**:
   ```bash
   find /sys/kernel/iommu_groups/ -type l
   ```

6. **Probar Configuración**:
   ```bash
   # Verificar módulos
   lsmod | grep vfio
   
   # Verificar dispositivos
   lspci -nn -D | grep -i vga
   ```

## ⚠️ Consideraciones por Hardware

### NVIDIA
- **Ventajas**: Excelente soporte en Windows
- **Desventajas**: Error 43 posible, requiere configuración especial
- **Solución Error 43**: Usar `kvm.hidden=on`

### AMD
- **Ventajas**: Menos problemas de compatibilidad
- **Desventajas**: Drivers pueden ser menos estables
- **Consideración**: Usar drivers más recientes

### Intel
- **Ventajas**: Excelente soporte IOMMU
- **Desventajas**: GPU integrada limitada
- **Recomendación**: Usar con GPU dedicada

---

**💡 Consejo**: Siempre haz backup de tus configuraciones antes de hacer cambios, y prueba en un entorno de desarrollo si es posible. 