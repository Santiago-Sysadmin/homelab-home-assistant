# Sensores del servidor publicados en Home Assistant

El host Proxmox ejecuta un temporizador de systemd cada 2 minutos que publica sensores en Home Assistant mediante `POST /api/states/<entity_id>`. El script de ejemplo está en el repositorio principal: [`scripts/ha-sensors.sh`](https://github.com/Santiago-Sysadmin/homelab-proxmox-t14/blob/main/scripts/ha-sensors.sh).

## Cómo funciona

- Autenticación con un **token de larga duración** creado en el perfil de usuario de Home Assistant y guardado en un fichero con permisos restringidos en el host. El script lo lee y nunca lo imprime.
- Los estados publicados por la API REST **no sobreviven a un reinicio de Home Assistant**: reaparecen en el siguiente ciclo (≤2 min).
- Las URL y rutas se configuran en un fichero local de entorno que no se sube al repositorio.

## Sensores

| Entidad | Qué mide | Para qué |
|---|---|---|
| `sensor.servidor_temperatura_cpu` | Temperatura del procesador | Aviso por calor sostenido |
| `sensor.servidor_temperatura_disco` | Temperatura del NVMe | Seguimiento |
| `sensor.servidor_bateria` | Carga de la batería | Autonomía durante un corte |
| `binary_sensor.servidor_enchufado` | Si el portátil está conectado a la corriente | Detección de cortes de luz |
| `sensor.servidor_espacio_vms_libre` | % libre del almacenamiento de las VMs | Aviso por poco espacio |
| `sensor.servidor_espacio_copias_libre` | GB libres en el disco de copias | Aviso por poco espacio |
| `sensor.servidor_copia_vms_antiguedad` | Horas desde la última copia correcta de la máquina más atrasada | Detectar copias que no se hacen |
| `sensor.servidor_copia_fotos_antiguedad` | Horas desde la última copia de fotos | Ídem |
| `sensor.servidor_estado_copias` | `ok` / `atrasada` / `fallo` | Semáforo único para las copias |

## Integración nativa de Proxmox

Además se añadió la integración `proxmoxve` con un usuario **solo lectura** (rol `PVEAuditor`) y un token de API propio. Aporta, por cada guest y por el nodo: estado, uso de CPU, porcentaje de memoria y tiempo de actividad. Es la base de los avisos de «máquina caída» y de memoria o CPU altas.

## Buenas prácticas aplicadas

- Mínimo privilegio: el usuario de Proxmox no puede modificar nada.
- Secretos fuera del repositorio y con permisos restringidos.
- Los sensores se nombran con un prefijo común (`servidor_`) para filtrarlos fácilmente en paneles y automatizaciones.
