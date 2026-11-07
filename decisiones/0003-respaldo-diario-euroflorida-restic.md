# 0003 — Respaldo diario e incremental de euroflorida con restic y copia en Google Drive

- **Fecha:** 2026-11-06
- **Estado:** aceptada
- **Decide:** Francisco Mendez
- **Alcance:** archivos de WordPress de eurofloridaluxuryrealtors.com: volumen
  `euroflorida-stack_wp_data` en el dedicado `empire`, compose local en `~/euroflorida-stack`.

## Contexto

Antes de esta decisión, euroflorida tenía:

| Qué | Frecuencia | Dónde | Fuera de Hetzner |
|---|---|---|---|
| Base de datos (`coolify-backups.sh`) | Diaria, 03:00. Retención GFS: 7 diarios, 4 semanales y 6 mensuales | Storage Box `coolify-backups/` | Sí, en Google Drive cifrado, 60 días |
| Archivos + base (`euroflorida-files-backup.sh`) | Semanal, domingo 06:00. Últimas 4 | Storage Box `euroflorida-files/` | **No** |

Problemas:

1. **Se podían perder hasta 7 días de fotos.** Las fotos subidas entre un domingo y el
   siguiente no estaban respaldadas.
2. **Las fotos no salían de Hetzner.** El servidor y el Storage Box son del mismo
   proveedor; si ambos fallan juntos, las fotos se pierden.
3. El tar semanal sube 10 GB completos cada vez, aunque las fotos cambian poco
   (en la semana del 30-10 al 06-11 hubo 348 archivos nuevos, 26 MB).
4. Nunca se había probado restaurar.

## Decisión

1. **Respaldo diario con restic** del volumen completo de WordPress, excluyendo
   `wp-content/cache`, `upgrade` y `upgrade-temp-backup`.
   - Repositorio cifrado: `/mnt/storagebox/euroflorida-restic`.
   - Solo guarda lo que cambia, con deduplicación.
   - Retención: 7 diarios, 4 semanales y 6 mensuales, igual que la base de datos.
     Los domingos hace `prune` y `check --read-data-subset=5%`.
2. **Copia externa:** cada día, `rclone copy` del repositorio a
   `gdrive-crypt:euroflorida-restic`, con doble cifrado (restic + rclone crypt).
   Se usa `copy` y no `sync`, para que un repositorio local dañado nunca borre la copia externa.
3. **La base de datos sigue como estaba.** El dump diario del GFS ya llega a Google Drive.
4. **El tar semanal se mantiene** mientras el respaldo nuevo demuestra estar estable.
   Después se puede retirar para liberar unos 40 GB del Storage Box (ver "Pendiente").

| Pieza | Ubicación |
|---|---|
| Script | `/home/fmendezy/scripts/euroflorida-restic-backup.sh` (root, 750) |
| Cron | `/etc/cron.d/euroflorida-restic`, todos los días 02:15 |
| Log | `/var/log/euroflorida-restic.log`, rotación semanal en `/etc/logrotate.d/euroflorida-restic` |
| Contraseña | `/home/fmendezy/secretos/restic-euroflorida.pass` (600) y `gdrive-crypt:claves/restic-euroflorida.pass` |
| Alertas | Justin, clave `backup_euroflorida_restic`, si falla el backup, el prune, el check o la copia a Drive |

> **Sin la contraseña de restic el repositorio es irrecuperable.** Debe estar también en
> el gestor de contraseñas de Francisco. La copia en Drive solo sirve si se tienen las
> claves de `rclone crypt`.

## Resultados al implementarla

- Primer snapshot: 2026-11-06 21:18. Ocupa 8,9 GB en el repositorio (9,6 GB originales).
  La subida inicial a Drive tardó unas 4 horas; desde ahí solo suben los cambios.
- `restic check`: sin errores.

### Prueba de restauración (2026-11-07)

| Prueba | Resultado |
|---|---|
| Base de datos: dump del 06-11 03:00 restaurado en un MariaDB temporal sin red | 212 tablas, igual que producción. 616 posts publicados y 7.892 adjuntos frente a 617 y 7.893 en producción; la diferencia es lo publicado después de las 03:00. 6 usuarios, igual |
| Fotos desde el Storage Box: 40 al azar | 40 de 40 idénticas (sha256), en 2 s |
| Fotos desde Google Drive: las mismas 40 | 40 de 40 idénticas (sha256), en 2 min 40 s |

## Cómo restaurar

Todos los comandos se ejecutan como root:

```bash
export RESTIC_PASSWORD_FILE=/home/fmendezy/secretos/restic-euroflorida.pass
export RESTIC_CACHE_DIR=/var/cache/restic
R=/mnt/storagebox/euroflorida-restic        # o: rclone:gdrive-crypt:euroflorida-restic
                                           # (con RCLONE_CONFIG=/home/fmendezy/.config/rclone/rclone.conf)
restic -r $R snapshots --host euroflorida
# Una carpeta o un archivo puntual, a un directorio aparte para revisarlo antes de copiarlo:
restic -r $R restore latest --host euroflorida --target /tmp/restaurar \
  --include /var/lib/docker/volumes/euroflorida-stack_wp_data/_data/wp-content/uploads/2026/11
```

Para restaurar el volumen completo hay que detener el contenedor `php` del stack, restaurar
sobre el volumen y volver a levantarlo. La base de datos se restaura desde
`/mnt/storagebox/coolify-backups/<fecha>/mariadb-wp/euroflorida-stack-db-1.sql.gz`.

**Si el servidor se pierde por completo:** instalar restic y rclone en otra máquina,
recuperar la configuración de `rclone crypt` y la contraseña de restic, y usar
`-r rclone:gdrive-crypt:euroflorida-restic`.

## Pendiente

- Después de 2 a 4 semanas sin alertas, decidir si se retira el tar semanal de archivos
  de `euroflorida-files-backup.sh`. El dump semanal de la base se puede mantener.
- Repetir la prueba de restauración cada 3 meses (próxima: 2027-02).
