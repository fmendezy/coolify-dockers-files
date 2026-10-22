# 0002 — Runners de IntraDEM con versión fija y sin autoactualización

- **Fecha:** 2026-10-22
- **Estado:** aceptada
- **Decide:** Francisco Mendez
- **Alcance:** servicio `intradem-runner` de Coolify en el dedicado `empire`
  (UUID `tqbfahmkr8jhhwu2togc1bjq`). Son 6 runners self-hosted del repositorio
  `fmendezy/IntraDEM`: `intradem-runner-1`, `-2`, `-secretos` y `-liviano-1` a `-3`.

## Contexto

El 2026-10-22, a las 14:32 UTC, GitHub obligó a los runners a autoactualizarse de la
versión 2.337.0 a la 2.338.0. Dentro de la imagen `myoung34/github-runner`, esa
actualización borró `/actions-runner/bin` y no instaló la versión nueva. Los 6 contenedores
quedaron en un bucle de reinicio (código 127, `Runner.Listener: No such file or directory`)
y se dieron de baja en GitHub. El CI de IntraDEM estuvo detenido unas 3 horas.

Se arregló fijando la imagen en `myoung34/github-runner:2.338.0-ubuntu-noble`, que ya
trae la versión nueva instalada.

## Decisión

1. **La versión del runner se fija en el tag de la imagen**, por ejemplo `2.338.0-ubuntu-noble`.
   Nunca se usa `latest`.
2. **Se desactiva la autoactualización** con `DISABLE_AUTO_UPDATE: 'true'` en el
   `environment` de los 6 runners. La imagen traduce la variable a `config.sh --disableupdate`.
   Al arrancar, el log muestra `Disable auto update option is enabled`.
3. **Para actualizar el runner se cambia el tag de la imagen**, nunca se deja que el runner
   se actualice a sí mismo.

## Por qué

- La autoactualización dentro de esta imagen falla y deja el runner inservible. Además,
  ocurre sin aviso, a la hora que GitHub decida.
- Cambiar el tag es un cambio explícito, que se puede revisar y revertir. La imagen
  ya trae la versión correcta y no necesita descargar nada al arrancar.

## Consecuencias y obligación

**GitHub deja de enviar trabajos a un runner que no se actualiza dentro de los 30 días
siguientes a la publicación de una versión nueva**, sea mayor, menor o de parche. Con
actualizaciones críticas de seguridad puede dejar de enviarlos antes
([referencia de GitHub](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)).

Por eso, cada vez que [actions/runner](https://github.com/actions/runner/releases) publica
una versión, hay que subir el tag de la imagen antes de 30 días.

> **Pendiente al escribir esto:** GitHub publicó la **v2.338.1 el 2026-10-22** y ya existe
> la imagen `myoung34/github-runner:2.338.1-ubuntu-noble`. Plazo para cambiarla:
> **2026-11-21**.

Si se olvida, el síntoma es que los trabajos de CI quedan en cola sin que ningún runner los
tome, aunque los runners aparezcan online.

## Procedimiento para subir la versión

1. Verificar que exista la imagen:
   `docker manifest inspect myoung34/github-runner:<versión>-ubuntu-noble`.
2. En Coolify, editar el compose del servicio `intradem-runner` y cambiar las 6 líneas
   `image:` de los runners, además del comentario que documenta el tag. Hacerlo en lote
   con la API (`GET`/`PATCH /api/v1/services/<uuid>`, con `docker_compose_raw` en base64,
   verificando que cambien exactamente esas líneas).
3. Esperar a que ningún runner esté ocupado:
   `gh api repos/fmendezy/IntraDEM/actions/runners --jq '.runners[] | "\(.name) \(.busy)"'`.
4. Reiniciar el servicio: `POST /api/v1/services/<uuid>/restart?latest=true`.
5. Verificar que los 6 aparezcan `online` en GitHub y que su log diga
   `Current runner version: '<versión>'`, `Disable auto update option is enabled` y
   `Listening for Jobs`.

## Historial

- **2026-10-22:** caída del CI de 14:32 a 17:18 UTC. Fijados en 2.338.0. Más tarde se
  agregó `DISABLE_AUTO_UPDATE` y se reinició con los 6 runners libres; los 6 quedaron online.
