# Storj Node - Guías de Instalación

Documentación y procedimientos para la gestión de nodos Storj en Raspberry Pi.

## 📄 Contenido

- **[PROCEDIMIENTO_STORJ_NODE.md](PROCEDIMIENTO_STORJ_NODE.md)** - Guía completa de instalación y migración de nodo Storj con LVM optimizado

## 🎯 Objetivo

Este repositorio contiene procedimientos detallados para:
- Eliminación segura de nodos descalificados
- Configuración optimizada de almacenamiento LVM con caché SSD
- Generación de identidad (protocolo 2026)
- Instalación y configuración de storagenode con Docker
- Verificación y mantenimiento del nodo

## ⚙️ Requisitos

- Raspberry Pi 4 (u otro hardware compatible)
- Raspberry Pi OS (64-bit)
- Docker instalado
- Disco SSD para caché (recomendado 100-200GB)
- Disco HDD para almacenamiento principal (2TB+)
- Puerto TCP/UDP 28967 abierto en router
- DynDNS configurado (recomendado)

## 🚀 Uso

Lee el [PROCEDIMIENTO_STORJ_NODE.md](PROCEDIMIENTO_STORJ_NODE.md) completo antes de comenzar.

**Importante:** Reemplaza todos los placeholders (`<TU_USUARIO>`, `<IP_RASPBERRY>`, etc.) con tus valores reales antes de ejecutar los comandos.

## ⚠️ Advertencias

- Este procedimiento es para **migración de nodos descalificados** o **nueva instalación**
- **NO ejecutes** estos pasos en un nodo activo y funcional
- Asegúrate de tener **backups** de tus archivos de identidad si tienes un nodo operativo
- La descalificación en Storj es **permanente e irreversible**

## 📝 Licencia

Documentación proporcionada "tal cual" sin garantías. Úsala bajo tu propio riesgo.

## 🤝 Contribuciones

Si encuentras errores o mejoras, siéntete libre de abrir un issue o pull request.
