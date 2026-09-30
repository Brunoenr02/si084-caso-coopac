## INSTRUCCIONES DEL EXAMEN PRÁCTICO DE LA UNIDAD

| | |
| --- | --- |
| **Duración** | **40 minutos** |
| **Modalidad** | Individual, en el laboratorio |
| **Permitido** | Apuntes, repositorio del curso e internet |
| **Terminal** | bash · Linux, macOS, WSL o Git Bash en Windows |

## Qué necesitas

| Material | Dónde se obtiene | Cómo compruebas que lo tienes |
| --- | --- | --- |
| Docker con Compose | https://docs.docker.com/get-docker/ · lo instalaste en el taller 01 | `docker compose version` |
| Imagen `postgres:16` | La descargaste en el taller 01. Si no está, `docker pull postgres:16` | `docker images postgres` |
| Paquete `si084-caso-coopac.zip` | Aula virtual, Semana 06, recurso «Caso COOPAC · paquete» | Se descomprime sin errores |
| Git y cuenta de GitHub | https://git-scm.com/ y https://github.com/ | `git --version` |

## CASO DEL EXAMEN

La **COOPAC Santa Rosa** es una cooperativa de ahorro y crédito. Su Consejo de Administración te pide responder una sola pregunta. **¿Cumple el servidor de base de datos del core financiero la política de seguridad que el Consejo aprobó?** El corte es al 31/12/2025.

El paquete trae todo lo que necesitas.

| Archivo o carpeta | Qué es |
| --- | --- |
| `docker-compose.yml` | El servidor de base de datos del core, tal como lo configuró el área de TI |
| `politica/POLITICA-DE-SEGURIDAD.md` | Las siete reglas, R1 a R7, contra las que auditas |
| `respaldos/` | Los respaldos del servidor, la tarea que los genera y su registro |
| `datos/` | Los datos que carga la base al arrancar. No los modifiques |
| `evidencias/` | Donde guardas la salida de cada procedimiento |
| `INFORME.md` | La plantilla del informe que entregas |

## Paso 1 · Levanta el servidor

- Descarga `si084-caso-coopac.zip` del aula virtual. Déjalo en tu carpeta **Descargas**.

- Abre una terminal bash. En Windows es **Git Bash**; en Linux y macOS, la **Terminal**.

- Copia estas tres órdenes. La primera te lleva a Descargas, la segunda descomprime el paquete y crea la carpeta `si084-caso-coopac`, y la tercera te deja dentro de ella.

```bash
cd ~/Downloads
unzip si084-caso-coopac.zip
cd si084-caso-coopac
```

- Levanta el servidor. La orden termina sola cuando aparece Healthy, en unos 10 segundos.

```bash
docker compose up -d --wait
```

- Abre con cualquier editor el archivo `politica/POLITICA-DE-SEGURIDAD.md` y lee las siete reglas.

**Todas las órdenes del examen se escriben en esta misma terminal, dentro de la carpeta `si084-caso-coopac`.** Si cierras la terminal, al abrirla de nuevo vuelve a entrar con `cd ~/Downloads/si084-caso-coopac`.

## Paso 2 · Ejecuta los siete procedimientos

Cada procedimiento comprueba una regla de la política. Copia la orden, ejecútala y anota el resultado en la **sección 1 de `INFORME.md`**, con cifras y con «Sí» o «No» en la columna ¿Cumple?. La orden guarda su salida en `evidencias/`.

### P1 · Regla R1 · Exposición de la base de datos

```bash
docker compose ps | tee evidencias/P1_puertos.txt
```

**Anota** en qué dirección y en qué puerto publica `sr_bd`. `0.0.0.0` significa abierto a toda la red.

### P2 · Regla R2 · Contraseñas

```bash
grep -n "PASSWORD" docker-compose.yml | tee evidencias/P2_credenciales.txt
```

**Anota** en qué archivo y línea está la contraseña del administrador y cuántos caracteres tiene. No copies la contraseña en el informe.

### P3 · Regla R3 · Cuentas con privilegio de superusuario

```bash
docker exec sr_bd psql -U postgres -d core -c "\du" | tee evidencias/P3_roles.txt
```

**Anota** qué cuentas, además de `postgres`, tienen el atributo Superuser.

### P4 · Regla R4 · Cuentas de personas cesadas y cuentas sin responsable

```bash
docker exec sr_bd psql -U postgres -d core -c "SELECT u.usuario, e.nombre, e.fecha_cese FROM usuarios u JOIN empleados e ON e.documento = u.documento WHERE u.estado = 'activo' AND e.fecha_cese IS NOT NULL ORDER BY e.fecha_cese" | tee evidencias/P4_cesados.txt

docker exec sr_bd psql -U postgres -d core -c "SELECT usuario, perfil FROM usuarios WHERE documento IS NULL AND estado = 'activo' ORDER BY perfil, usuario" | tee evidencias/P4_genericas.txt
```

**Anota** cuántas cuentas activas pertenecen a personas cesadas, a cuántas personas distintas corresponden y cuál es el cese más antiguo. Anota también cuántas cuentas activas no tienen documento y cuántas de ellas tienen perfil ADMIN.

### P5 · Regla R5 · Segregación de funciones en los desembolsos

Esta orden la completas tú. Reemplaza `<condición>` por la regla R5 escrita en SQL, en las dos órdenes. Las columnas que necesitas son `usuario_registra`, `usuario_aprueba`, `monto` y `umbral_aprobacion`.

```bash
docker exec sr_bd psql -U postgres -d core -c "SELECT count(*) AS casos, sum(monto) AS monto_total, count(DISTINCT usuario_registra) AS usuarios FROM desembolsos WHERE <condición>" | tee evidencias/P5_segregacion.txt

docker exec sr_bd psql -U postgres -d core -c "SELECT id, fecha, monto, usuario_registra FROM desembolsos WHERE <condición> ORDER BY monto DESC" | tee -a evidencias/P5_segregacion.txt
```

**Anota** cuántos desembolsos incumplen la regla, su monto total y cuántos usuarios distintos los hicieron.

### P6 · Regla R6 · Registro de conexiones y modificaciones

```bash
docker exec sr_bd psql -U postgres -d core -c "SHOW log_connections" -c "SHOW log_statement" | tee evidencias/P6_registro.txt
```

**Anota** los dos valores. Para cumplir R6 deberían ser `on` y `mod`. Busca en `docker-compose.yml` la línea que los configura.

### P7 · Regla R7 · Respaldo y prueba de restauración

- Mira qué respaldos existen y cómo terminó la tarea programada.

- Restaura el último respaldo en una base nueva llamada `restauracion`.

- Revisa qué tablas quedaron restauradas.

```bash
ls respaldos/ | tee evidencias/P7_respaldos.txt
tail -n 5 respaldos/respaldo.log | tee -a evidencias/P7_respaldos.txt

docker exec sr_bd psql -U postgres -c "CREATE DATABASE restauracion"
docker exec -i sr_bd psql -U postgres -d restauracion < respaldos/core_2025-11-14.sql
docker exec sr_bd psql -U postgres -d restauracion -c "\dt" | tee evidencias/P7_restauracion.txt
```

**Anota** la fecha del último respaldo, cuántos días pasaron hasta el corte del 31/12/2025 y qué tabla del core falta en lo restaurado. La causa está en `respaldos/respaldo.sh` y en `respaldos/respaldo.log`.

## Paso 3 · Redacta dos hallazgos

- Elige **las dos reglas incumplidas que consideres más graves** para la cooperativa.

- Redacta cada una en las secciones 2 y 3 de `INFORME.md`.

| Elemento | Qué escribes |
| --- | --- |
| Título | El problema en una línea |
| Condición | Lo que encontraste, con cifras y el archivo de evidencia |
| Criterio | La regla de la política y su control de referencia, con el código |
| Causa | Por qué ocurre, según lo que viste en los archivos del paquete. «Falta de control» no es una causa |
| Efecto | Qué le puede pasar a la cooperativa, en soles, cuentas o días |
| Recomendación | Qué hacer, quién lo hace y en qué plazo |

## Paso 4 · Sube tu evidencia a GitHub

- Crea en https://github.com/new un repositorio **público y vacío** llamado `si084-caso-coopac`.

- Escribe en la carátula de `INFORME.md` tu nombre, tu código y la URL `https://github.com/<usuario>/si084-caso-coopac/tree/examen-u1`.

- Sella la evidencia, sube el repositorio y crea la etiqueta `examen-u1`. Cambia `<usuario>` por tu usuario de GitHub.

```bash
sha256sum evidencias/*.txt > evidencias/SHA256SUMS.txt
git init -b main
git add .
git commit -m "Caso COOPAC · examen práctico U1"
git remote add origin https://github.com/<usuario>/si084-caso-coopac.git
git push -u origin main
git tag examen-u1
git push origin examen-u1
```

En macOS, usa `shasum -a 256` en lugar de `sha256sum`.

## Paso 5 · Entrega el informe

- Abre `INFORME.md` en tu repositorio de GitHub, pulsa **Ctrl+P** y elige **Guardar como PDF**.

- Sube el PDF al aula virtual.

- Apaga el servidor con `docker compose down -v`.

## Qué entregas

| | |
| --- | --- |
| **Archivo** | `SI084-EXPRAC-U1-<ApellidoNombre>.pdf` |
| **Dónde se sube** | Aula virtual, tarea «Examen práctico · Unidad I» |
| **Cuándo vence** | Al terminar los 40 minutos, en la sesión de laboratorio |
| **Repositorio** | `si084-caso-coopac` en tu cuenta de GitHub, con la etiqueta `examen-u1` y la carpeta `evidencias/` completa |

**El informe es lo que se califica; el repositorio es lo que lo prueba.** Un resultado sin su archivo en `evidencias/` se califica como no logrado. No se califica un informe sin la URL del repositorio en la carátula.

## Cómo se califica

| Qué | Puntos |
| --- | --- |
| P1, P2, P3 y P6 · resultado correcto y ¿Cumple? bien marcado | 1 cada uno, 4 en total |
| P4 · las dos cifras de cesados y las dos de cuentas sin responsable | 2 |
| P5 · condición SQL correcta, con casos, monto y usuarios | 2 |
| P7 · restauración ejecutada, tabla faltante y días sin respaldo | 2 |
| Hallazgo 1 · condición con cifras, criterio con código, causa con evidencia, efecto y recomendación | 4 |
| Hallazgo 2 · lo mismo | 4 |
| Repositorio con la etiqueta, las evidencias y `SHA256SUMS.txt` | 2 |
| **Total** | **20** |

---

**SI-084 · Auditoría de Sistemas** · Dr. Oscar Juan Jimenez Flores · Tacna, Perú