# Procedimiento: Eliminación y Reinstalación Nodo Storj

> **⚠️ IMPORTANTE:** Antes de usar este procedimiento, reemplaza todos los placeholders con tus valores reales:
> - `<TU_USUARIO>` - Tu usuario en la Raspberry Pi
> - `<IP_RASPBERRY>` - IP local de tu Raspberry Pi (ej: 192.168.1.100)
> - `<TU_WALLET_ADDRESS>` - Tu dirección de wallet Ethereum (ERC-20)
> - `<TU_EMAIL>` - Tu correo electrónico
> - `<TU_DOMINIO_DDNS>` - Tu dominio DDNS configurado
> - `<TU_NODE_ID>` - Se generará automáticamente al crear la identidad
> - `<ESPACIO_TB>` - Espacio que dedicarás al nodo (ej: 2TB, 5TB, etc.)
> - `<TAMAÑO_CACHE>` - Tamaño de caché SSD en GB (ej: 100, 190, etc.)
> - `<UUID_DE_TU_DISCO>` - Se obtiene con el comando blkid

## Contexto
- **Raspberry Pi 4** con Raspberry Pi OS (aarch64)
- **IP:** <IP_RASPBERRY>
- **Usuario:** <TU_USUARIO>
- **Nodo anterior:** Descalificado permanentemente en 4 satélites
- **Causa:** Fallo I/O por problemas en LVM antiguo

---

## PARTE 1: ELIMINACIÓN DEL NODO ANTERIOR

### 1. Detener y eliminar contenedor Docker
```bash
docker stop storagenode
docker rm storagenode
```

### 2. Respaldar información del nodo antiguo
```bash
# Guardar Node ID y configuración
docker run --rm \
  --mount type=bind,source="/mnt/storj/identity",destination=/app/identity \
  --mount type=bind,source="/mnt/storj/config",destination=/app/config \
  storjlabs/storagenode:latest info > /home/<TU_USUARIO>/nodo_antiguo_info.txt

# Verificar Node ID antiguo (descalificado)
# Ejemplo: 13feeG7xwma82cp73zvDk4QG8fcoq9NHHP9XniE9iDowYAunhF
```

### 3. Desmontar y eliminar LVM antiguo
```bash
# Desmontar
sudo umount /mnt/storj

# Eliminar LVM
sudo lvremove -f storj-vg/storj-data
sudo vgremove -f storj-vg
sudo pvremove /dev/sdb1 /dev/sdc1
```

---

## PARTE 2: CONFIGURACIÓN NUEVO STORAGE CON CACHÉ SSD

### 4. Preparar discos
```bash
# SSD: /dev/sdb (238.5GB) - Para caché
# HDD: /dev/sdc (7.3TB) - Para datos

# Crear particiones
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary 0% 100%
sudo parted /dev/sdc mklabel gpt
sudo parted /dev/sdc mkpart primary 0% 100%
```

### 5. Configurar LVM con caché SSD
```bash
# Habilitar bloques mixtos en LVM
sudo sed -i 's/allow_mixed_block_sizes = 0/allow_mixed_block_sizes = 1/' /etc/lvm/lvm.conf

# Crear Physical Volumes
sudo pvcreate /dev/sdb1
sudo pvcreate /dev/sdc1

# Crear Volume Group
sudo vgcreate storj-vg /dev/sdc1

# Crear Logical Volume para datos (7TB)
sudo lvcreate -L 7T -n storj-data storj-vg

# Extender VG con SSD
sudo vgextend storj-vg /dev/sdb1

# Crear cache pool (190GB en SSD)
sudo lvcreate -L 190G -n storj-cache storj-vg /dev/sdb1

# Convertir a cache pool
sudo lvconvert --type cache-pool storj-vg/storj-cache

# Añadir caché al volumen de datos (modo writeback)
sudo lvconvert --type cache --cachevol storj-cache --cachemode writeback storj-vg/storj-data
```

### 6. Formatear y montar
```bash
# Formatear ext4
sudo mkfs.ext4 /dev/storj-vg/storj-data

# Crear punto de montaje
sudo mkdir -p /mnt/storj

# Montar
sudo mount /dev/storj-vg/storj-data /mnt/storj

# Obtener UUID para auto-mount
sudo blkid /dev/storj-vg/storj-data
# Ejemplo UUID: 142cf648-4025-4b2a-b6d0-26a55d5cfbd4

# Configurar auto-mount persistente (reemplaza <UUID> con el valor obtenido arriba)
echo "UUID=<UUID_DE_TU_DISCO> /mnt/storj ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

---

## PARTE 3: GENERACIÓN NUEVA IDENTIDAD

### 7. Generar nueva identidad (sin authorization token)
```bash
# Crear directorio
sudo mkdir -p /mnt/storj/identity

# Generar identidad (versión 2026, no requiere token)
docker run --rm -v /mnt/storj/identity:/app/identity storjlabs/storagenode:latest identity create --identity-dir /app/identity

# Cambiar permisos
sudo chown -R <TU_USUARIO>:<TU_USUARIO> /mnt/storj/identity

# Obtener Node ID
docker run --rm -v /mnt/storj/identity:/app/identity storjlabs/storagenode:latest identity info --identity-dir /app/identity
# Nuevo Node ID: <TU_NODE_ID>
```

---

## PARTE 4: CONFIGURACIÓN DEL NODO

### 8. Crear estructura de directorios
```bash
sudo mkdir -p /mnt/storj/config
sudo chown -R <TU_USUARIO>:<TU_USUARIO> /mnt/storj/config
```

### 9. Ejecutar SETUP inicial (CRÍTICO)
```bash
# Este paso crea:
# - config.yaml
# - storage-dir-verification (archivo binario con Node ID)
# - Estructura de subdirectorios (blobs, temp, trash)

docker run --rm -e SETUP="true" \
  --mount type=bind,source="/mnt/storj/identity",destination=/app/identity \
  --mount type=bind,source="/mnt/storj/config",destination=/app/config \
  storjlabs/storagenode:latest
```

### 10. Iniciar contenedor del nodo
```bash
docker run -d \
  --restart unless-stopped \
  --stop-timeout 300 \
  -p 28967:28967/tcp \
  -p 28967:28967/udp \
  -p 14002:14002 \
  -e WALLET="<TU_WALLET_ADDRESS>" \
  -e EMAIL="<TU_EMAIL>" \
  -e ADDRESS="<TU_DOMINIO_DDNS>:28967" \
  -e STORAGE="<ESPACIO_TB>" \
  --mount type=bind,source="/mnt/storj/identity",destination=/app/identity \
  --mount type=bind,source="/mnt/storj/config",destination=/app/config \
  --name storagenode \
  storjlabs/storagenode:latest
```

---

## PARTE 5: VERIFICACIÓN

### 11. Comprobar estado del nodo
```bash
# Ver logs de inicio
docker logs storagenode 2>&1 | grep -E "(Node.*started|Public server)"

# Salida esperada:
# Node <TU_NODE_ID> started
# Public server started on [::]:28967
# Private server started on 127.0.0.1:7778

# Verificar uptime estable
docker ps | grep storagenode
```

### 12. Acceso
- **Dashboard:** http://<IP_RASPBERRY>:14002
- **Portainer:** http://<IP_RASPBERRY>:9000

---

## CONFIGURACIÓN CRÍTICA POST-INSTALACIÓN

### Port Forwarding en Router
**OBLIGATORIO** para que el nodo funcione:
```
Protocolo: TCP + UDP
Puerto Externo: 28967
IP Interna: <IP_RASPBERRY>
Puerto Interno: 28967
```

### Verificar DynDNS
- Dominio: `<TU_DOMINIO_DDNS>`
- Debe apuntar a la IP pública actual

---

## INFORMACIÓN DEL NODO NUEVO

| Parámetro | Valor |
|-----------|-------|
| **Node ID** | <TU_NODE_ID> |
| **Wallet** | <TU_WALLET_ADDRESS> |
| **Email** | <TU_EMAIL> |
| **Dirección** | <TU_DOMINIO_DDNS>:28967 |
| **Storage Total** | <ESPACIO_TB> TB |
| **Caché SSD** | <TAMAÑO_CACHE>GB modo writeback |
| **Puerto Público** | 28967 (TCP/UDP) |
| **Dashboard** | 14002 |

---

## COMANDOS ÚTILES DE MANTENIMIENTO

```bash
# Ver logs en tiempo real
docker logs -f storagenode

# Reiniciar nodo
docker restart storagenode

# Ver uso de espacio
df -h /mnt/storj

# Ver caché LVM
sudo lvs -a -o +cache_mode,cache_settings

# Detener nodo
docker stop storagenode

# Estado de contenedores
docker ps -a
```

---

## NOTAS IMPORTANTES

1. **Archivo storage-dir-verification:**
   - Archivo binario de 32 bytes con el Node ID
   - **NO eliminar nunca** - causaría pérdida total del nodo
   - Se crea automáticamente con el paso SETUP
   - Ubicación: `/mnt/storj/config/storage/storage-dir-verification`

2. **Período de vetting:**
   - Primeros 1-3 meses: tráfico bajo mientras pasa auditorías
   - Después: aumento progresivo de datos almacenados
   - Normal que aparezca poca actividad al inicio

3. **Backup de identidad:**
   - Carpeta `/mnt/storj/identity/` contiene archivos irreemplazables
   - Hacer backup periódico en lugar seguro
   - Sin estos archivos, el nodo se pierde completamente

4. **No reutilizar Node ID antiguo:**
   - Si tienes un nodo descalificado, **NUNCA** intentes reutilizar su identidad
   - La descalificación es permanente e irreversible
   - Siempre genera una identidad nueva desde cero

---

## NOTAS ADICIONALES

Este procedimiento fue probado exitosamente en Raspberry Pi 4 con Raspberry Pi OS aarch64.
Adaptable a otros sistemas Linux con Docker instalado.
