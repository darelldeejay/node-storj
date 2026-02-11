# Storj Node - Guías de Instalación

Documentación y procedimientos para la gestión de nodos Storj en Raspberry Pi.

## 📄 Contenido

- **[PROCEDIMIENTO_STORJ_NODE.md](PROCEDIMIENTO_STORJ_NODE.md)** - Guía completa de instalación y migración de nodo Storj con LVM optimizado
- **[MONITOREO_NETDATA.md](MONITOREO_NETDATA.md)** - Sistema de monitoreo avanzado con alertas Telegram

## 🎯 Objetivo

Este repositorio contiene procedimientos detallados para:
- Eliminación segura de nodos descalificados
- Configuración optimizada de almacenamiento LVM con caché SSD
- Generación de identidad (protocolo 2026)
- Instalación y configuración de storagenode con Docker
- **Sistema de monitoreo completo con Netdata**
- **Alertas preventivas por Telegram**
- **Prevención de descalificaciones mediante monitoreo S.M.A.R.T.**
- Verificación y mantenimiento del nodo

## ⚙️ Requisitos

### Básicos (Nodo Storj):
- Raspberry Pi 4 (u otro hardware compatible)
- Raspberry Pi OS (64-bit)  
- Docker instalado
- Disco SSD para caché (recomendado 100-200GB)
- Disco HDD para almacenamiento principal (2TB+)
- Puerto TCP/UDP 28967 abierto en router
- DynDNS configurado (recomendado)

### Adicionales (Monitoreo):
- RAM adicional: +50MB para Netdata
- Bot de Telegram (opcional, para notificaciones)
- Acceso SSH configurado

## 🚀 Uso

### 📋 Flujo de Trabajo Recomendado

1. **Fase 1 - Instalación del Nodo:**
   - Lee el [PROCEDIMIENTO_STORJ_NODE.md](PROCEDIMIENTO_STORJ_NODE.md) completo antes de comenzar
   - Ejecuta todos los pasos hasta tener el nodo Storj funcionando
   - Verifica que el dashboard sea accesible y el nodo esté operativo

2. **Fase 2 - Configuración de Monitoreo:**  
   - Una vez que el nodo esté estable (24+ horas funcionando)
   - Sigue la guía [MONITOREO_NETDATA.md](MONITOREO_NETDATA.md)
   - Configura alertas y notificaciones Telegram (opcional)

**Importante:** Reemplaza todos los placeholders (`<TU_USUARIO>`, `<IP_RASPBERRY>`, `<TU_BOT_TOKEN>`, etc.) con tus valores reales antes de ejecutar los comandos.

## ⚠️ Advertencias

- Este procedimiento es para **migración de nodos descalificados** o **nueva instalación**
- **NO ejecutes** estos pasos en un nodo activo y funcional
- Asegúrate de tener **backups** de tus archivos de identidad si tienes un nodo operativo
- La descalificación en Storj es **permanente e irreversible**

## 🔔 Sistema de Monitoreo

El sistema incluye:
- **Dashboard web en tiempo real** (Netdata)
- **Monitoreo S.M.A.R.T.** de temperatura, sectores dañados, vida útil de discos
- **7 alertas críticas** configuradas específicamente para nodos Storj:
  - Temperatura elevada de SSD/HDD (>50°C warning, >55-60°C crítico)
  - Espacio en disco bajo (>85% warning, >95% crítico)  
  - Sectores dañados detectados (>0 warning, >10-50 crítico)
  - Saturación de I/O del disco (>90% warning, >98% crítico)
  - RAM insuficiente (<10% warning, <5% crítico)
- **Notificaciones automáticas por Telegram** ante problemas
- **Prevención de descalificaciones** mediante detección temprana de fallos

**Beneficio principal:** Tu nodo nunca más será descalificado sin aviso previo.

## 📝 Licencia

Documentación proporcionada "tal cual" sin garantías. Úsala bajo tu propio riesgo.

## 🤝 Contribuciones

Si encuentras errores o mejoras, siéntete libre de abrir un issue o pull request.
