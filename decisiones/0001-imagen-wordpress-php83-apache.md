# 0001 — Fijar los sitios WordPress en `wordpress:php8.3-apache`

- **Fecha:** 2026-10-21
- **Estado:** aceptada
- **Decide:** Francisco Mendez
- **Alcance:** los 16 servicios WordPress de Coolify en el dedicado `empire` que usaban `wordpress:latest`

## Contexto

El 2026-10-21 PHP publicó las versiones 8.2.35, 8.3.36, 8.4.27 y 8.5.12 para corregir
PHP2Shell (CVE-2026-109618 y CVE-2026-109619): dos fallas críticas (CVSS 9.2) de
ejecución remota de código sin autenticación, que ya se estaban explotando.

Los 16 sitios usaban `wordpress:latest`. Ese día aparecieron dos problemas:

1. **Desfase de Docker Hub.** Durante varias horas, `wordpress:latest` siguió trayendo
   PHP 8.3.35, sin el parche. Mientras tanto, cualquier redeploy en Coolify que hiciera
   pull dejaba el sitio vulnerable aunque ya existiera el parche.
2. **`latest` no fija nada.** El día que la imagen oficial pase su default a PHP 8.4 o
   8.5, un redeploy cualquiera cambiaría la versión de PHP sin aviso y podría romper
   temas o plugins antiguos.

En estos sitios, la versión de WordPress (el core) está en el volumen
`/var/www/html`, no en la imagen. La imagen solo define PHP, sus extensiones y Apache.

## Decisión

Usar `wordpress:php8.3-apache` en lugar de `wordpress:latest`.

- Fija lo que importa: **PHP 8.3 y Apache con mod_php**.
- Sigue recibiendo los parches de seguridad de PHP 8.3 cada vez que se hace pull y redeploy.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Seguir con `wordpress:latest` | Cambios de versión de PHP sin aviso. |
| Fijar por digest (`wordpress@sha256:…`) | Es reproducible al byte, pero nunca recibe parches solo. Cada CVE obligaría a editar 16 servicios a mano, y en servicios sin repositorio eso termina olvidándose. El riesgo de quedar sin parchar es mayor que el beneficio. |
| Fijar la versión de WordPress en el tag (`6.x.y-php8.3-apache`) | No aporta nada: el core vive en el volumen, y el tag igual se reconstruye con cada versión nueva de PHP. |

## Consecuencias

- Los sitios dejan de cambiar de versión menor de PHP por accidente.
- Los parches de PHP 8.3 **no se aplican solos**: hace falta pull y redeploy. Ante un CVE
  de PHP hay que revisar que Docker Hub ya haya publicado la imagen corregida antes de
  hacer el redeploy (ver "Verificar la versión" más abajo).
- **PHP 8.3 tiene soporte de seguridad hasta el 2027-12-31.** Antes de esa fecha hay que
  migrar a `php8.4-apache` o `php8.5-apache`, probando cada sitio.
- Pasar a otra versión de PHP queda como un cambio explícito y revisable, no como un efecto lateral.

## Procedimiento

### Cambiar la imagen de un servicio

En Coolify: servicio → *Edit Compose File* → cambiar la línea `image:` del servicio
`wordpress` → guardar → *Deploy*.

No edites `/data/coolify/services/<uuid>/docker-compose.yml` en el servidor: Coolify
lo regenera desde su base de datos en cada deploy.

Para hacerlo en lote, usa la API de Coolify:

1. `GET /api/v1/services/<uuid>` y toma `docker_compose_raw`.
2. Reemplaza la línea de la imagen. Verifica que se haya cambiado exactamente una línea.
3. `PATCH /api/v1/services/<uuid>` con `{"docker_compose_raw": "<compose en base64>"}`.
4. Vuelve a hacer `GET` y compara.

Guardar el compose no reinicia el contenedor. El cambio se aplica en el próximo deploy.

### Verificar la versión de PHP

```bash
# Imagen publicada en Docker Hub
docker pull -q wordpress:php8.3-apache && docker run --rm --entrypoint php wordpress:php8.3-apache -r 'echo PHP_VERSION;'

# Lo que corre cada sitio
for c in $(docker ps --format '{{.Names}}' | grep '^wordpress-'); do echo "$c $(docker exec "$c" php -r 'echo PHP_VERSION;')"; done
```

## Servicios incluidos

Dolce Ora, Academia Fundación DEM, Acoger DEM, Fundación DEM, Gym Brigada 1581,
Aleana Sanfiel Web 2026, La Cuca del Maipo, Fundación Viña Florece, Centro Cultural
Maipo, TraumaBuin.cl, Optindus, Lassen Haus Catálogo, LyR Climatización y Gasfitería,
Alimentos Mansel, Constructora ByU y Muebles Ebenezer.

Quedan fuera los sitios que construyen su propia imagen desde un repositorio
(IntraDEM, gomezyasociados.cl, euroflorida), porque la versión se fija en su Dockerfile:

- **IntraDEM:** usa `FROM wordpress:latest`. Conviene aplicarle esta misma decisión en su repo.
- **gomezyasociados.cl:** sigue en PHP 7.4 con un backport del parche.
- **euroflorida:** usa `wordpress:php8.3-fpm`.

## Historial

- **2026-10-21:** se aplicó la decisión a los 16 servicios, solo en el compose, sin
  reiniciar contenedores. Ese mismo día los contenedores ya corrían PHP 8.3.36, con una
  imagen construida localmente mientras Docker Hub publicaba la suya.
