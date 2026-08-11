# Qué falta y cómo ejecutarlo

Este archivo es solo una guía de trabajo para cerrar el Momento 1. No es parte
de la rúbrica, es para no perder el hilo de qué falta y en qué orden hacerlo.

## Estado actual

| Entregable | Estado |
|---|---|
| E1 — Repo propio | Listo |
| E2 — Dominio + ER (`docs/dominio_de_negocio.md`) | Listo |
| E3 — 9 migraciones en `sql_migrations/` | Listas, sin correr todavía |
| E4 — Workflow (`.github/workflows/flyway-migrate.yml`) | Listo |
| E5 — Evidencia de un run exitoso y uno fallido corregido | **Pendiente** |
| E6 — README | Listo |
| Secreto `NEON_MAIN_DATABASE_URL` en GitHub | Falta confirmar si ya se creó |

Lo único que falta es **ejecutar todo** y capturar la evidencia. El resto ya
está escrito.

## Por qué no se puede simplemente correr todo de una

Si aplicas las 9 migraciones de una sola vez, el bug de `phone VARCHAR(10)`
(migración 4) y su corrección (migración 7) se aplican en el mismo momento —
nunca verías el error fallar de verdad, y no tendrías evidencia real para C5
de la rúbrica. Por eso el orden de ejecución va en dos tandas, tanto en `dev`
como en `main`.

## Paso a paso

### 1. Preparar `flyway.conf` (una sola vez)

```bash
cp flyway.conf.example flyway.conf
```

Edita `flyway.conf` con los datos de la branch **dev** de tu proyecto Neon
(el connection string traducido a formato JDBC — usuario y clave en sus
propias líneas, no dentro de la URL).

### 2. Aplicar en `dev` hasta el bug (sin el fix)

```bash
flyway info                            # confirma que ves las 9 migraciones, todas Pending
flyway migrate -target=202608111537    # aplica solo hasta la migración del bug
```

### 3. Provocar el error en `dev` y capturarlo

```bash
psql "$(grep '^NEON_DEV_DATABASE_URL=' .env | cut -d= -f2-)" \
  -c "UPDATE users SET phone = '+57 3001234567' WHERE id = 1;"
```

Debería fallar con `value too long for type character varying(10)`. Guarda
esa salida (captura de pantalla o el texto) — es tu primera pieza de
evidencia para E5/C5.

### 4. Aplicar el resto en `dev` (incluye el fix)

```bash
flyway migrate    # sin -target: aplica el resto, incluida la corrección
```

Repite el mismo `UPDATE` de arriba — ahora debería funcionar sin error.
Guarda también esta salida como evidencia de la corrección.

### 5. Configurar el secreto en GitHub (si no está ya)

**Settings → Secrets and variables → Actions → New repository secret**

| Nombre | Valor |
|---|---|
| `NEON_MAIN_DATABASE_URL` | Connection string completo de la branch **main** |

### 6. Primer push a `main` — solo hasta el bug

```bash
git add sql_migrations/V202608111534__create_schema.sql \
        sql_migrations/V202608111535__seed_data.sql \
        sql_migrations/V202608111536__create_penalties_table.sql \
        sql_migrations/V202608111537__add_phone_to_users.sql \
        .github/workflows/flyway-migrate.yml \
        README.md \
        docs/
git commit -m "feat(db): library baseline, penalties table, phone column"
git push origin main
```

Espera a que el workflow termine (pestaña Actions). Debería quedar en verde
— este es tu primer run exitoso para E5.

### 7. Provocar el error contra `main` y capturarlo

Mismo `UPDATE` de antes, pero usando el connection string de `main` en vez de
`dev`. Guarda la salida del error — evidencia de que el bug también existe en
producción, no solo en tu máquina.

### 8. Segundo push a `main` — el resto (incluye el fix)

```bash
git add sql_migrations/R__fn_calculate_penalty_fee.sql \
        sql_migrations/R__sp_pay_penalty.sql \
        sql_migrations/V202608111538__fix_users_phone_length.sql \
        sql_migrations/V202608111539__add_index_loans_user_id.sql \
        sql_migrations/V202608111540__add_index_loans_copy_id.sql
git commit -m "fix(db): widen phone column; add penalty functions and indexes"
git push origin main
```

Otro run en Actions, debería quedar en verde también. Repite el `UPDATE`
contra `main` — ahora funciona. Esa es tu evidencia de la corrección en
producción.

### 9. Armar `docs/evidencias/`

Con las capturas de los pasos 3, 4, 6, 7 y 8, guárdalas en
`docs/evidencias/` (nombres sugeridos: `dev_error.png`, `dev_fix.png`,
`main_run_1.png`, `main_error.png`, `main_run_2.png`, `main_fix.png`, o
enlaces directos a los runs de Actions). Con eso, E5 queda cerrado.

### 10. Borrar este archivo

Una vez todo lo anterior esté hecho, este `pendientes.md` ya no sirve para
nada — bórralo antes de la entrega final, no es parte de lo que pide la
rúbrica.
