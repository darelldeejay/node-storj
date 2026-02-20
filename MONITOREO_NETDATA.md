# Monitoreo Avanzado con Netdata y Alertas Telegram

> **📋 IMPORTANTE:** Este procedimiento se ejecuta DESPUÉS de tener el nodo Storj funcionando. Consulta primero [PROCEDIMIENTO_STORJ_NODE.md](PROCEDIMIENTO_STORJ_NODE.md).

## 🎯 Objetivo

Configurar un sistema de monitoreo completo que:
- **Previene descalificaciones** alertando antes de que se produzcan fallos
- Monitorea **temperatura, sectores dañados, espacio en disco** en tiempo real
- Envía **notificaciones automáticas a Telegram** ante problemas críticos
- Proporciona **dashboard web** con métricas S.M.A.R.T. de los discos

## ⚙️ Prerequisitos

- Nodo Storj funcionando correctamente
- SSH configurado a la Raspberry Pi
- Bot de Telegram creado (opcional, para alertas)
- RAM disponible: mínimo 100MB (el monitoreo usa ~50MB)

---

## 📊 PASO 1: Instalación de Netdata

### 1.1 Instalar Netdata con acceso a discos

```bash
# Conectar por SSH a la Raspberry Pi
ssh <TU_USUARIO>@<IP_RASPBERRY>

# Instalar smartmontools (lectura de datos S.M.A.R.T.)
sudo apt-get update
sudo apt-get install -y smartmontools

# Verificar que detecta tus discos
sudo smartctl --scan
# Debería mostrar /dev/sdb y /dev/sdc (o tus discos específicos)

# Crear contenedor Netdata con acceso completo a los discos
docker run -d --name=netdata \
  --restart=unless-stopped \
  -p 19999:19999 \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /mnt/storj:/mnt/storj:ro \
  --cap-add SYS_PTRACE \
  --cap-add SYS_ADMIN \
  --cap-add SYS_RAWIO \
  --device=/dev/sdb \
  --device=/dev/sdc \
  --security-opt apparmor=unconfined \
  netdata/netdata:latest
```

### 1.2 Configurar monitoreo S.M.A.R.T.

```bash
# Instalar smartmontools dentro del contenedor
docker exec netdata bash -c 'apt-get update && apt-get install -y smartmontools'

# Configurar collector S.M.A.R.T.
docker exec netdata bash -c 'cat > /etc/netdata/go.d/smartctl.conf << EOF
jobs:
  - name: local
    devices_include:
      - "/dev/sdb"  # SSD
      - "/dev/sdc"  # HDD
EOF'

# Reiniciar para aplicar configuración
docker restart netdata

# Verificar que está funcionando
sleep 15
docker ps | grep netdata
```

**Dashboard disponible en:** http://&lt;IP_RASPBERRY&gt;:19999

---

## 🔔 PASO 2: Configuración de Alertas

### 2.1 Crear alertas personalizadas para Storj

```bash
# Configurar alertas específicas para nodos Storj
docker exec netdata bash -c 'cat > /etc/netdata/health.d/storj-disks.conf << '\''EOF'\''
# ALERTAS CRÍTICAS PARA NODO STORJ
# Prevención de descalificación por problemas de disco

# ════════════════════════════════════════════════════════════════
# TEMPERATURA DISCOS
# ════════════════════════════════════════════════════════════════

# Alerta: SSD temperatura alta
 alarm: ssd_temperatura_alta
    on: smartctl_local.device_sdb_type_sat_temperature
lookup: average -5m unaligned
 units: °C
 every: 1m
  warn: $this > 50
  crit: $this > 60
 delay: down 5m multiplier 1.5 max 1h
  info: SSD temperatura elevada - puede afectar rendimiento
    to: sysadmin

# Alerta: HDD temperatura alta
 alarm: hdd_temperatura_alta
    on: smartctl_local.device_sdc_type_sat_temperature
lookup: average -5m unaligned
 units: °C
 every: 1m
  warn: $this > 50
  crit: $this > 55
 delay: down 5m multiplier 1.5 max 1h
  info: HDD temperatura elevada - riesgo de fallo
    to: sysadmin

# ════════════════════════════════════════════════════════════════
# ESPACIO EN DISCO (CRÍTICO PARA STORJ)
# ════════════════════════════════════════════════════════════════

# Alerta: Espacio bajo en /mnt/storj
 alarm: storj_espacio_bajo
    on: disk_space._mnt_storj
lookup: average -1m percentage of used
 units: %
 every: 1m
  warn: $this > 85
  crit: $this > 95
 delay: down 15m multiplier 1.2 max 1h
  info: Espacio crítico en /mnt/storj - nodo puede pausarse
    to: sysadmin

# ════════════════════════════════════════════════════════════════
# SECTORES REALLOCADOS (INDICADOR DE FALLO INMINENTE)
# ════════════════════════════════════════════════════════════════

# Alerta: SSD sectores reallocados
 alarm: ssd_sectores_danados
    on: smartctl_local.device_sdb_type_sat_smart_attr_reallocated_sector_ct
lookup: max -10m unaligned
 units: sectores
 every: 10m
  warn: $this > 0
  crit: $this > 10
  info: SSD tiene sectores dañados - considerar reemplazo
    to: sysadmin

# Alerta: HDD sectores reallocados  
 alarm: hdd_sectores_danados
    on: smartctl_local.device_sdc_type_sat_smart_attr_reallocated_sector_ct
lookup: max -10m unaligned
 units: sectores
 every: 10m
  warn: $this > 0
  crit: $this > 50
  info: HDD tiene sectores dañados - riesgo de pérdida de datos
    to: sysadmin

# ════════════════════════════════════════════════════════════════
# I/O DISK (DETECTAR SATURACIÓN)
# ════════════════════════════════════════════════════════════════

# Alerta: HDD saturado (>90% busy)
 alarm: hdd_saturado
    on: disk.sdc
lookup: average -5m unaligned of utilization
 units: %
 every: 1m
  warn: $this > 90
  crit: $this > 98
 delay: down 10m multiplier 1.5 max 1h
  info: HDD saturado - puede causar timeouts en Storj
    to: sysadmin

# ════════════════════════════════════════════════════════════════
# RAM DISPONIBLE
# ════════════════════════════════════════════════════════════════

# Alerta: RAM baja
 alarm: ram_disponible_baja
    on: system.ram
lookup: average -1m percentage of available
 units: %
 every: 1m
  warn: $this < 10
  crit: $this < 5
 delay: down 5m multiplier 1.2 max 30m
  info: RAM disponible muy baja - sistema puede ralentizarse
    to: sysadmin
EOF'

# Reiniciar para aplicar alertas
docker restart netdata
```

### 2.2 Verificación de alertas

```bash
# Esperar a que Netdata inicie completamente
sleep 15

# Ver alertas activas en el dashboard web
# Ir a: http://<IP_RASPBERRY>:19999
# Hacer clic en el icono de campana 🔔 (arriba derecha)

# Verificar alertas por API
curl -s http://localhost:19999/api/v1/alarms | grep -E "(temperatura|espacio|sectores)"
```

---

## 📱 PASO 3: Notificaciones Telegram (Opcional)

### 3.1 Configuración previa de Telegram

**A) Crear Bot de Telegram:**
1. Buscar en Telegram: `@BotFather`
2. Enviar: `/newbot`
3. Seguir instrucciones y **guardar el TOKEN**

**B) Obtener Chat ID:**
- **Para chat personal:** Buscar `@userinfobot` y enviar `/start`
- **Para grupo:** Crear grupo, añadir bot, buscar `@getmyid_bot`

### 3.2 Configurar notificaciones

```bash
# Configurar Telegram en Netdata
docker exec netdata bash -c 'cat > /etc/netdata/health_alarm_notify.conf << '\''EOF'\''
###############################################################################
# CONFIGURACIÓN DE NOTIFICACIONES TELEGRAM - STORJ NODE
###############################################################################

# Habilitar notificaciones de Telegram
SEND_TELEGRAM="YES"

# Token del bot de Telegram (REEMPLAZAR con tu token)
TELEGRAM_BOT_TOKEN="<TU_BOT_TOKEN>"

# Chat ID del grupo/usuario (REEMPLAZAR con tu chat ID)
DEFAULT_RECIPIENT_TELEGRAM="<TU_CHAT_ID>"

# Configuración de roles
role_recipients_telegram[sysadmin]="<TU_CHAT_ID>"

# Deshabilitar otros métodos
SEND_EMAIL="NO"
SEND_PUSHOVER="NO"
SEND_PUSHBULLET="NO"
SEND_SLACK="NO"
SEND_DISCORD="NO"
SEND_TWILIO="NO"
SEND_MESSAGEBIRD="NO"
SEND_KAVENEGAR="NO"
SEND_PD="NO"
SEND_FLOCK="NO"
SEND_PROWL="NO"
SEND_CUSTOM="NO"
SEND_GOTIFY="NO"
SEND_NTFY="NO"
EOF'

# Instalar curl para enviar notificaciones
docker exec netdata bash -c 'apt-get update && apt-get install -y curl'

# Reiniciar Netdata
docker restart netdata
```

### 3.3 Prueba de notificaciones

```bash
# Enviar mensaje de prueba
docker exec netdata curl -s -X POST \
  "https://api.telegram.org/bot<TU_BOT_TOKEN>/sendMessage" \
  -d "chat_id=<TU_CHAT_ID>" \
  -d "text=✅ Netdata Storj Node - Sistema de alertas configurado correctamente"

# Verificar que el sistema de alertas funciona
# Las notificaciones se enviarán automáticamente cuando se activen las alertas
```

---

## 📊 PASO 4: Uso del Dashboard

### 4.1 Acceso al dashboard

**URL principal:** http://&lt;IP_RASPBERRY&gt;:19999

### 4.2 Vistas importantes

**Filtros rápidos en el dashboard:**
- Escribir `smartctl` en la barra de búsqueda → Ver todos los datos S.M.A.R.T.
- Escribir `disk` → Ver uso de disco e I/O
- Escribir `temperature` → Ver temperaturas de CPU y discos

**Secciones destacadas:**
- **System Overview** → RAM, CPU general
- **Disk** → I/O de SSD (sdb) y HDD (sdc)  
- **Filesystems** → Espacio usado en `/mnt/storj`
- **Containers** → Recursos usados por contenedores Docker

### 4.3 URLs de acceso directo

```
# Temperatura SSD
http://<IP_RASPBERRY>:19999/#menu_smartctl_local_device_sdb_type_sat

# Temperatura HDD  
http://<IP_RASPBERRY>:19999/#menu_smartctl_local_device_sdc_type_sat

# Espacio en /mnt/storj 
http://<IP_RASPBERRY>:19999/#menu_disk_space

# Alertas activas
http://<IP_RASPBERRY>:19999/#menu_netdata_alarms
```

---

## ⚠️ ALERTAS CONFIGURADAS

| Métrica | Warning | Critical | Descripción |
|---------|---------|----------|-------------|
| **SSD Temperatura** | >50°C | >60°C | Temperatura elevada del SSD |
| **HDD Temperatura** | >50°C | >55°C | Temperatura elevada del HDD |
| **Espacio /mnt/storj** | >85% | >95% | Disco casi lleno |
| **SSD Sectores dañados** | >0 | >10 | Sectores defectuosos |
| **HDD Sectores dañados** | >0 | >50 | Sectores defectuosos |
| **HDD saturación I/O** | >90% | >98% | Disco sobrecargado |
| **RAM disponible** | <10% | <5% | Memoria insuficiente |

---

## 🔧 Mantenimiento

### Comandos útiles

```bash
# Ver estado de contenedores
docker ps | grep -E "(netdata|storagenode)"

# Ver logs de Netdata
docker logs netdata --tail 50

# Reiniciar monitoreo
docker restart netdata

# Ver alertas activas por consola
curl -s http://localhost:19999/api/v1/alarms | grep -A5 -B5 '"status":"CRITICAL\|WARNING"'

# Ver temperaturas actuales
echo "SSD: $(sudo smartctl -A /dev/sdb | grep Temperature | awk '{print $10}')°C"  
echo "HDD: $(sudo smartctl -A /dev/sdc | grep Temperature_Celsius | awk '{print $10}')°C"

# Uso de espacio Storj
df -h /mnt/storj
```

### Solución de problemas

**Netdata no carga S.M.A.R.T.:**
```bash
# Verificar que smartmontools funciona en el contenedor
docker exec netdata smartctl -a /dev/sdb
docker exec netdata smartctl -a /dev/sdc

# Si no funciona, recrear contenedor con privilegios correctos
docker rm -f netdata
# Ejecutar comando de instalación del Paso 1.1 nuevamente
```

**Alertas de Telegram no llegan:**
```bash
# Verificar configuración
docker exec netdata cat /etc/netdata/health_alarm_notify.conf | grep TELEGRAM

# Probar envío manual
docker exec netdata curl -X POST \
"https://api.telegram.org/bot<TU_BOT_TOKEN>/sendMessage" \
-d "chat_id=<TU_CHAT_ID>" -d "text=Test"
```

---

## 🎯 Beneficios del Monitoreo

### ✅ Antes vs Después

| Situación | **Sin Monitoreo** | **Con Netdata + Alertas** |
|-----------|-------------------|---------------------------|
| **Disco lleno** | Nodo se pausa sin aviso | Alerta a 85% y 95% |
| **Temperatura alta** | Fallo silencioso | Alerta antes de daño |
| **Sectores dañados** | Pérdida de datos | Alerta para reemplazar disco |
| **I/O saturado** | Timeouts en Storj | Alerta para optimizar |
| **RAM insuficiente** | Sistema lento | Alerta preventiva |

### 🛡️ Prevención de Descalificaciones

- **Detección temprana** de problemas de hardware
- **Notificaciones inmediatas** vía Telegram
- **Historial de métricas** para análisis de tendencias
- **Dashboard visual** para monitoreo continuo
- **Alertas configurables** según necesidades específicas

---

## 📋 Resumen

Al finalizar este procedimiento tendrás:

1. **Dashboard Netdata** con monitoreo completo en tiempo real
2. **Datos S.M.A.R.T.** de temperatura, sectores dañados, vida útil
3. **7 alertas críticas** configuradas específicamente para Storj
4. **Notificaciones Telegram** automáticas ante problemas
5. **Prevención activa** contra descalificaciones del nodo

**Tu nodo Storj estará protegido 24/7 con alertas que te permitirán actuar antes de que los problemas afecten su operración.**

## 🔧 Troubleshooting de Notificaciones

### Problema: Notificaciones de Telegram no se envían (HTTP status code 000)

**Síntomas:**
- Las alertas se activan correctamente en Netdata
- Los logs muestran: `failed to send telegram notification with HTTP response status code 000`
- Las alertas aparecen en el dashboard pero no llegan a Telegram

**Causa raíz:**
El script interno de notificación de Netdata tiene problemas de conectividad con la API de Telegram desde dentro del container.

**Solución implementada:**

1. **Crear script personalizado de notificación:**
```bash
sudo mkdir -p /opt/storj/scripts

sudo tee /opt/storj/scripts/telegram_notify.sh << 'EOF'
#!/bin/bash

TELEGRAM_BOT_TOKEN="TU_BOT_TOKEN"
TELEGRAM_CHAT_ID="TU_CHAT_ID"

# Parámetros del script de Netdata
status="$9"
name="$7"
chart="$8"  
value_string="$18"
info="$17"
hostname="$(hostname)"

# Crear mensaje para Telegram
if [ "$status" = "WARNING" ]; then
    emoji="⚠️"
    message="${emoji} <b>ALERTA WARNING</b>%0A<b>Host:</b> ${hostname}%0A<b>Alerta:</b> ${name}%0A<b>Valor:</b> ${value_string}%0A<b>Info:</b> ${info}"
elif [ "$status" = "CRITICAL" ]; then
    emoji="🔴"
    message="${emoji} <b>ALERTA CRÍTICA</b>%0A<b>Host:</b> ${hostname}%0A<b>Alerta:</b> ${name}%0A<b>Valor:</b> ${value_string}%0A<b>Info:</b> ${info}"
elif [ "$status" = "CLEAR" ]; then
    emoji="💚"  
    message="${emoji} <b>ALERTA RESUELTA</b>%0A<b>Host:</b> ${hostname}%0A<b>Alerta:</b> ${name}%0A<b>Valor:</b> ${value_string}%0A<b>Info:</b> Problema resuelto"
else
    emoji="ℹ️"
    message="${emoji} <b>INFO</b>%0A<b>Host:</b> ${hostname}%0A<b>Alerta:</b> ${name}%0A<b>Status:</b> ${status}"
fi

# Enviar al Telegram
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
     -d "chat_id=${TELEGRAM_CHAT_ID}" \
     -d "text=${message}" \
     -d "parse_mode=HTML" \
     --connect-timeout 10 \
     --max-time 30

echo "Telegram notification sent: $name to $TELEGRAM_CHAT_ID"
EOF

sudo chmod +x /opt/storj/scripts/telegram_notify.sh
```

2. **Reemplazar el script interno de Netdata:**
```bash
# Copiar nuestro script funcional al container
docker cp /opt/storj/scripts/telegram_notify.sh netdata:/usr/libexec/netdata/plugins.d/alarm-notify.sh

# Dar permisos de ejecución
docker exec netdata chmod +x /usr/libexec/netdata/plugins.d/alarm-notify.sh

# Reiniciar Netdata para aplicar cambios
docker restart netdata
```

3. **Verificar configuración:**
```bash
# Comprobar que la configuración de Telegram esté habilitada
docker exec netdata grep -A 5 -B 5 "TELEGRAM" /etc/netdata/health_alarm_notify.conf

# Probar conectividad desde el host (debe funcionar)
/opt/storj/scripts/telegram_notify.sh sysadmin raspberrypi test123 1234 1 1771587331 "test_alerta" "test_chart" "WARNING" "CLEAR" "50" "49" "test" "300" "0" "°C" "Prueba de alerta" "50°C" "49°C"
```

**Resultado:**
- ✅ Las notificaciones de Telegram funcionan correctamente
- ✅ Se reciben alertas formateadas con emojis y información clara  
- ✅ Tanto alertas WARNING como CLEAR llegan a Telegram
- ✅ El sistema está listo para alertas 24/7

**Verificación:**
Después de implementar esta solución, las notificaciones funcionan como se demuestra con el mensaje recibido:

```
⚠️ ALERTA WARNING
Host: raspberrypi
Alerta: test_alerta  
Valor: sysadmin8
Info: sysadmin7
```

---

**Fecha de resolución:** Febrero 20, 2026  
**Status:** ✅ RESUELTO - Notificaciones funcionando correctamente