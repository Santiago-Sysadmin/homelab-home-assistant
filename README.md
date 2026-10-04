# Home Assistant en Proxmox: automatización doméstica local

> Parte del proyecto [`homelab-proxmox-t14`](https://github.com/Santiago-Sysadmin/homelab-proxmox-t14). Aquí se documenta en detalle todo lo relacionado con Home Assistant: migración, Zigbee, integración con Alexa, monitorización del servidor y lecciones aprendidas.
>
> **Política de publicación:** este repositorio no contiene direcciones IP, puertos, MACs, nombres de red Wi-Fi, tokens, identificadores de webhook ni información que revele horarios o presencia en casa. Los nombres de dispositivos y de entidades se han generalizado.

**EN —** Home Assistant OS runs as a VM on Proxmox and controls the home locally (Zigbee via ZHA, Alexa via Matter). This repo documents the migration (keeping all paired devices), a cleanup of dead integrations, the diagnosis of an unstable Zigbee mesh caused by Wi-Fi interference, and how the server publishes its own health sensors to Home Assistant to drive mobile alerts. Documentation is in Spanish; no network details are published.

## Resumen

Home Assistant OS (HAOS) se ejecuta como máquina virtual en Proxmox VE y controla la domótica de la vivienda de forma **local**: luces y relés Zigbee, integración con Alexa mediante Matter, sensores y avisos al móvil. Además actúa como **centro de alertas del propio servidor**: recibe sensores de Proxmox y notifica problemas (cortes de luz, copias atrasadas, temperatura, memoria…).

## Arquitectura

```text
Proxmox VE
 └── VM Home Assistant OS (2 vCPU, 4 GB, arranque prioritario)
      ├── USB passthrough: coordinador Zigbee (Sonoff Zigbee 3.0)
      ├── ZHA ............ red Zigbee (luces, relés, enchufes, sensores)
      ├── Matterbridge ... expone dispositivos a Alexa por Matter
      ├── Integración proxmoxve ... estado de host y guests (usuario de solo lectura)
      ├── API REST ........ sensores publicados por el host (ver docs/sensores-servidor.md)
      └── Notificaciones .. app móvil y bot de Telegram
```

## Qué se ha hecho

### 1. Migración desde la instalación anterior

- Copia completa de Home Assistant antes de empezar.
- Despliegue de HAOS como VM independiente y validación de red con la CLI del Supervisor.
- Paso del dongle Zigbee a la VM mediante USB passthrough por Vendor/Device ID.
- Restauración de ZHA reutilizando el mismo coordinador, de modo que **los dispositivos ya emparejados y las automatizaciones siguieron funcionando** sin volver a emparejar nada.
- La VM arranca con prioridad (`startup order`) justo después del DNS para que la domótica esté disponible tras un reinicio del servidor.

### 2. Alexa por Matter

- Se usa **Matterbridge** como add-on para publicar dispositivos de Home Assistant hacia Alexa mediante Matter.
- Observación útil: los mensajes `Peer no longer responding` en sus logs coinciden con cortes de red o de luz (el altavoz pierde la conexión) y se resuelven solos al volver; no indican un fallo de Home Assistant.

### 3. Auditoría y limpieza

- Auditoría **en solo lectura** por línea de comandos a través del agente invitado de la VM (sin necesidad de token): versión de Core/Supervisor/OS, salud del sistema, add-ons, integraciones, base de datos, copias y dispositivos Zigbee con su último contacto.
- Detección de integraciones huérfanas que apuntaban a equipos que ya no existen (MQTT, Wyoming, dispositivos sueltos, Emulated Hue).
- Limpieza con método seguro: copia completa previa → parar el Core → editar los registros de `.storage` con un script → arrancar → verificar que no hay errores de arranque. Los componentes personalizados sin uso se **movieron** a una carpeta de retirados en lugar de borrarse.
- Retirada de apps móviles de dispositivos que ya no se usan.
- Una automatización que dependía de una entidad eliminada se desactivó sola; se documentó y se eliminó con copia previa.

### 4. Zigbee: diagnóstico de una red inestable

Síntoma: algunas órdenes a relés Zigbee fallaban tras ~15 s con «device did not respond» aunque ZHA los marcaba como disponibles.

Análisis:

- Los relés afectados estaban en una **cadena de 3–4 saltos** con enlaces de baja calidad (LQI muy bajo).
- El Wi-Fi de 2,4 GHz de la casa usaba canales que solapaban con el canal Zigbee en uso, y el dongle estaba junto a un hub USB y un disco USB 3 (ruido en 2,4 GHz).

Acciones:

- Reubicar los canales Wi-Fi de 2,4 GHz en canales que dejan libre el espectro del canal Zigbee, con ancho de 20 MHz.
- Alejar el dongle con un **alargador USB 2.0**.
- Tras el cambio, las órdenes respondieron en ~0,1 s.

Lecciones:

- Las tablas de vecinos de ZHA son una **foto del último escaneo de topología** y pueden estar desactualizadas: hay que solicitar un reescaneo y esperar ~100 s antes de interpretarlas.
- Criterio orientativo de LQI: >100 muy bueno, 50–100 correcto, <50 flojo; lo que realmente importa es que las órdenes contesten rápido.
- Si volviera a ocurrir: reiniciar eléctricamente el relé, añadir un repetidor Zigbee en mitad de la cadena o valorar cambiar el canal Zigbee (con riesgo de dispositivos huérfanos).
- Más detalle en [`docs/zigbee.md`](docs/zigbee.md).

### 5. Resiliencia ante cortes de luz y de internet

- Comprobación del historial para verificar que los relés **no se encienden solos** al volver la corriente (estado al encender).
- Tras un corte de internet, algunas integraciones en la nube pueden quedar en `setup_retry`: se recargan por la API sin reiniciar Home Assistant.
- Aviso al móvil cuando se va y vuelve la luz, con duración del corte (ver [`docs/automatizaciones.md`](docs/automatizaciones.md)).

### 6. Monitorización del servidor desde Home Assistant

- El host Proxmox publica sensores por la **API REST** cada 2 minutos (temperatura, batería, espacio, antigüedad de copias…). El script está en el repositorio principal ([`scripts/ha-sensors.sh`](https://github.com/Santiago-Sysadmin/homelab-proxmox-t14/blob/main/scripts/ha-sensors.sh)).
- Integración oficial `proxmoxve` con un **usuario de solo lectura** (rol `PVEAuditor`) y token de API dedicado: mínimo privilegio.
- Automatizaciones con semáforo y emojis para que el aviso se entienda de un vistazo.
- Panel propio «Servidor» creado por la API WebSocket de Home Assistant.
- Detalle en [`docs/sensores-servidor.md`](docs/sensores-servidor.md) y [`docs/automatizaciones.md`](docs/automatizaciones.md).

### 7. Copias de seguridad

- Home Assistant hace copias parciales diarias dentro de la VM.
- La VM completa entra en las copias diarias a Proxmox Backup Server, con verificación periódica.
- Antes de cualquier cambio importante se crea una copia completa con nombre descriptivo (`pre-<motivo>`).

## Decisiones técnicas

| Decisión | Motivo |
|---|---|
| HAOS en VM (no contenedor) | Compatibilidad total con Supervisor, add-ons y copias nativas, y passthrough USB del coordinador. |
| Todo local | La domótica no depende de la nube; solo algunas integraciones de fabricante la necesitan. |
| Matter vía Matterbridge | Permite exponer dispositivos a Alexa sin depender de emulación heredada (Emulated Hue). |
| Alertas en Home Assistant | Reutiliza una herramienta ya presente en lugar de desplegar una pila de monitorización completa para un servidor pequeño. |
| Usuario de solo lectura para Proxmox | Si el token se filtrara, no podría modificar nada. |
| Avisos por rol | Los avisos técnicos van solo al administrador; al resto de la casa solo le llega lo que le afecta, con textos pensados para ellos. |

## Pendientes

- [ ] Reservar por MAC en el DHCP los dispositivos que Home Assistant usa por IP y que cambian al renovar (ver [`homelab-pihole`](https://github.com/Santiago-Sysadmin/homelab-pihole)).
- [ ] Eliminar automatizaciones y entidades `unavailable` heredadas.
- [ ] Valorar un repetidor Zigbee adicional para las zonas más alejadas.
- [ ] Revisar el reajuste de memoria de la VM cuando se amplíe la RAM del host.
