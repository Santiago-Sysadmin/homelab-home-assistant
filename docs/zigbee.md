# Zigbee: guía de diagnóstico

Notas de trabajo para una red ZHA con coordinador USB en una VM de Proxmox.

## Pasar el coordinador a la VM

- Usar USB passthrough por **Vendor/Device ID** (no por puerto físico): sobrevive a cambiar el dongle de puerto.
- Usar un **alargador USB 2.0** para alejar el dongle del equipo, de hubs y de discos USB 3, que generan ruido en 2,4 GHz.
- Para desenchufar el dongle limpiamente: deshabilitar la entrada de ZHA desde la API (`config_entries/disable`) y volver a habilitarla después; no hace falta reiniciar Home Assistant.

## Coexistencia con el Wi-Fi

- Zigbee y Wi-Fi comparten la banda de 2,4 GHz. Los canales Wi-Fi 1, 6 y 11 son los habituales, pero **uno de ellos puede caer encima del canal Zigbee** en uso.
- Elegir los canales Wi-Fi de 2,4 GHz para que dejen libre el espectro del canal Zigbee y usar 20 MHz de ancho en esa banda.
- Si se usan repetidores o mallas, revisar también su canal.

## Cómo leer la calidad de la red

- ZHA muestra la tabla de vecinos con **LQI** (calidad del enlace). Es una foto del último escaneo, no un dato en vivo.
- Procedimiento fiable: pedir un reescaneo de topología (`zha/topology/update`), esperar ~100 s y leer después.
- Orientación: LQI >100 muy bueno · 50–100 correcto · <50 flojo.
- La prueba definitiva es enviar órdenes que no cambien el estado y medir el tiempo de respuesta (≈0,1 s es sano).

## Síntoma típico y causas

| Síntoma | Causa probable | Qué probar |
|---|---|---|
| «Device did not respond» tras ~15 s, aunque figure disponible | Cadena larga de saltos con enlaces flojos o interferencias | Reubicar canales Wi-Fi, alargador USB, repetidor intermedio |
| Dispositivos que desaparecen tras un corte de luz | Relés sin estado al encender definido o mala ruta | Revisar el ajuste de estado al encender y el historial |
| Errores aislados del tipo `ZDO AREQ` en los logs del coordinador | Ruido del chip | Vigilar si se repiten; sin impacto si son esporádicos |

## Si hay caídas del dongle

El ahorro de energía USB del host (autosuspend) podría afectar al dongle. Si se observaran desconexiones, excluir ese dispositivo del ahorro de energía en la configuración de TLP.
