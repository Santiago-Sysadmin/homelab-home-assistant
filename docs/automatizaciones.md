# Automatizaciones de aviso del servidor

Resumen de las automatizaciones que vigilan el servidor desde Home Assistant. Los avisos se envían con un formato visual (emojis y semáforo 🟢🟡🔴) para entenderlos de un vistazo.

## Para el administrador

| Automatización | Condición (orientativa) | Aviso |
|---|---|---|
| Máquina apagada / de vuelta | Un guest pasa a «caído» y vuelve | Qué máquina y cuánto tiempo estuvo caída |
| Memoria del host alta | Uso sostenido por encima de ~90 % | Aviso preventivo antes de que haya cuelgues |
| Memoria de una VM alta | Uso sostenido por encima de ~92 % | Ídem para esa VM |
| CPU alta | Uso sostenido por encima de ~85 % | Posible proceso descontrolado |
| Temperatura alta | Por encima de ~85 °C | Revisar ventilación |
| Copias atrasadas o fallidas | `estado_copias` distinto de `ok` | Qué copia y desde cuándo |
| Poco espacio | Almacenamiento de VMs o disco de copias bajo un umbral | Evitar que las copias fallen por falta de espacio |
| Resumen diario | Cada mañana | Estado general en un solo mensaje |

## Corte de luz

- Un servicio del host detecta la pérdida y la vuelta de corriente.
- Al volver, espera a que haya internet, y envía un **webhook** a Home Assistant con la duración del corte y el nivel de batería.
- Home Assistant lo convierte en una notificación para las personas de la casa.
- El identificador del webhook es un secreto: se guarda en un fichero con permisos restringidos en el host y **no se incluye en este repositorio**.

## Avisos para el resto de la casa

Solo reciben lo que les afecta (por ejemplo, que las fotos o internet no funcionan y que ya se está resolviendo), con textos escritos para personas no técnicas que **no les piden avisar al administrador**.

## Notas de diseño

- Los umbrales se aplican con una duración mínima para evitar falsas alarmas por picos puntuales.
- Si el servidor DNS cae, el aviso puede no llegar hasta que el DNS vuelva; es una limitación conocida del diseño actual.
- Los avisos del propio Proxmox y de Proxmox Backup Server llegan también por **webhook** y se filtran para que solo lleguen advertencias y errores.
