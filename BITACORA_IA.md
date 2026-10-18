# Bitácora de uso de IA

**Grupo:** 4 · **Integrantes:** Juan David Roa Castañeda y Laura Camila Rodríguez Granados
**Herramientas usadas:** Claude

> Las tres secciones son obligatorias. **`## Rechazado` es la que se califica.**
> Una bitácora que solo lista prompts aceptados vale la mitad.
>
> **Cómo se trabajó.** Usamos Claude como apoyo para resolver el taller a partir del enunciado,
> el material del curso y el borrador de hallazgos. La IA propuso código, comandos y
> explicaciones; nosotros los ejecutamos en nuestras máquinas, comparamos con la salida real y
> descartamos lo que no coincidía con nuestro repositorio. Después, a pedido nuestro, la IA
> **revisó su propio trabajo**: esa autocrítica está en `## Rechazado` (filas 9 y 10), junto con
> lo que quedó pendiente a sabiendas. Cada commit lo hace su autor desde su propia identidad
> git, declarada en `EQUIPO.md`; la IA no hace commits.
> **Pendiente:** Juan David añade aquí sus propios prompts y lo que rechace al revisar.

## Prompts

| # | Parte | Quién | Prompt (resumido si es largo) |
|---|-------|-------|-------------------------------|
| 1 | A–E | Laura | «Estamos realizando el taller del corte 2; adjunto el enunciado, el código del repositorio y nuestros hallazgos». Se adjuntaron `ENUNCIADO.md`, `main.py`, `esquemas.py`, `HALLAZGOS.md` y las plantillas. |
| 2 | B–C | Laura | «Verifícame que la Parte B ya esté subida y explícame de qué se trata la Parte C». |
| 3 | C | Laura | «¿Cómo hago la Parte C para testearla en mi computador?», con los pasos exactos en VS Code; luego, la aclaración de que el equipo es Ubuntu 24.04. |
| 4 | C | Laura | «Explícame los resultados que da `contar_consultas.py`» (2, 2, 1, 1 con 10 pólizas y 5, 2, 1, 1 con 2000). |
| 5 | C | Laura | Interpretación de la salida de `verificar_entrega.py`, que marcaba que la historia no contenía el commit del semilla. |
| 6 | D–E | Laura | «Quiero subir solo lo de `DICTAMEN_IA.md`, que sería la Parte D» y reescritura de esta bitácora con los integrantes del grupo. |
| 7 | | Juan David | *(Juan David completa aquí sus propios prompts.)* |

## Aceptado

| # | Qué propuso la IA | Por qué lo aceptamos | Qué cambiamos antes de usarlo |
|---|-------------------|----------------------|-------------------------------|
| 1 | Reproducir cada fila de `HALLAZGOS.md` sobre el commit semilla `5a5cc5b` y copiar la salida real de la terminal | El enunciado compara la «Salida obtenida» con la real | — |
| 2 | Sesión por petición (`get_db` con `yield`), `PRAGMA foreign_keys` en un listener `connect`, `BaseSettings` con `lru_cache` | Son las restricciones B2–B4 tal como las enseñan M8 · 4 y M9 · 7 | — |
| 3 | Estrategias de carga: `selectinload` en `/polizas`, `lazy` en `/polizas/{id}`, `joinedload` en `/siniestros`, `agregada` en `/resumen` | Se decidieron después de contar las alternativas en cada endpoint (`comparar_estrategias.py`) y los conteos de `contar_consultas.py` coinciden | — |
| 4 | Crear el entorno con Python 3.11 y no con el 3.14 del sistema | Las dependencias están fijadas para 3.11, igual que el `Dockerfile` | — |
| 5 | Clonar el semilla y copiar encima nuestro trabajo con `rsync`, excluyendo `.venv`, `.git`, `.env` y `*.db` | `verificar_entrega.py` exige que la historia descienda del semilla | — |
| 6 | Subir cada parte en un commit propio, con `git add` solo de los archivos de esa parte | Da una historia que describe el cambio (C2) | — |

## Rechazado

| # | Qué propuso la IA | Por qué lo rechazamos | Qué hicimos en su lugar |
|---|-------------------|-----------------------|-------------------------|
| 1 | (Borrador previo, sin comprobar) H1 «sesión global» y H2 «409» como filas separadas, con H1 primero | Al reproducirlas en orden, H1 deja la sesión global en `PendingRollbackError` y todas las filas siguientes daban 500: el calificador no podría reproducir el resto | Se unieron en una sola fila (H14), que muestra los dos síntomas (`500 500`), y se puso la última |
| 2 | (Borrador previo) Comandos `POST` con números de póliza fijos | La segunda ejecución choca con el número ya creado y rompe la sesión del semilla: la evidencia no es reproducible | Números con `$(date +%s)`, distintos en cada pasada |
| 3 | Usar `joinedload` en `/polizas` porque «una consulta es mejor que cinco» | Medido con 2000 pólizas: 1 consulta pero ≈209 ms contra ≈173 ms de `selectinload`; el JOIN uno-a-muchos devuelve 6000 filas repitiendo la póliza | `selectinload` (5 consultas: lotes de 500 en el `IN`) |
| 4 | Usar `joinedload` también en `/resumen` | Medido: 1 consulta pero ≈166 ms, porque construye 6000 objetos `Siniestro` solo para hacer `len()` y `sum()` | Consulta agregada con `GROUP BY` (1 consulta, ≈7,6 ms) |
| 5 | Hacer `SECRETO_FIRMA` obligatorio (sin valor por defecto) | `docker run -p 8000:8000` debe arrancar sin `.env` (B9) y la imagen no puede llevarlo: con el campo obligatorio el contenedor no arranca | Valor por defecto **no secreto**, solo para desarrollo; el real va por entorno (`--env-file`) |
| 6 | Pegar en `main.py` los bloques de `selectinload`, `joinedload` y la consulta agregada | La IA partió del `main.py` original del semilla; el nuestro ya tenía la Parte C y habría quedado código duplicado | No se pegó nada: se revisó el `main.py` real y solo se corrió `contar_consultas.py` |
| 7 | Copiar los archivos con `cp -r` sobre el clon del semilla | Arrastraría el `.git` viejo y sobrescribiría la historia del semilla | `rsync` con `--exclude` de `.venv`, `.git`, `.env` y `*.db` |
| 8 | Hacer `git push --force` como paso de rutina | Reescribe la historia remota y puede borrar el trabajo del otro integrante | Solo se usa tras avisar al compañero y comprobar con `git log` qué hay en el remoto |
| 9 | (Autocrítica) `GET /health` responde `estado="ok"` aunque la base falle (solo cambia `base_datos` a `"error"`) | El `HEALTHCHECK` de Docker solo mira el código HTTP: con la base caída el contenedor seguiría «healthy» | No se cambió: B10 pide «200 e indica si la base responde». En la sustentación lo explicamos como límite conocido (en producción se devolvería 503) |
| 10 | (Autocrítica) `tests/test_api.py` del semilla sigue en el repositorio | Espera `200` al crear una póliza (ahora es `201`) y escribe en la base real, contra B8 | Pendiente: reemplazarlo por tests con `dependency_overrides` sobre una base temporal |
