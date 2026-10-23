# Dictamen sobre `ia_tests_propuesta.py` — Parte D

**Grupo:** 4 · **Integrantes:** Juan David Roa Castañeda y Laura Camila Rodríguez Granados

> La batería propuesta por la IA pasa en verde tanto sobre el semilla como sobre nuestro servicio
> corregido, y aun así no comprueba casi nada. Por cada defecto dañamos a propósito el servicio ya
> corregido (una mutación), ejecutamos el test original, que sigue en verde, y luego el de
> `tests/test_ia_corregido.py`, que se pone en rojo. Después deshacemos la mutación con
> `git checkout main.py esquemas.py`. Las salidas vienen de nuestra terminal, recortadas a las
> líneas de resultado.

## Defecto 1

- **Qué está mal**: Hay tests que nunca pueden fallar, porque sus aserciones dan por bueno tanto el resultado correcto como el incorrecto: `test_consultar_poliza_inexistente` acepta `in (200, 404)`; `test_crear_poliza_responde` y `test_primas_variadas_son_aceptadas` aceptan `in (200, 201, 422)` (creada, no creada o rechazada: todo vale); `test_listar_polizas_no_falla` solo exige `< 500`; y `test_score_de_poliza_recien_creada` acepta `"puntaje" in ... or "error" in ...`. Un test que pasa con cualquier respuesta no verifica el contrato (404, 201, 422).
- **Por qué es un defecto** (módulo · sección): M10 · 4. Framework pytest — toda aserción debe poder fallar: tiene que fijar el valor esperado y no un conjunto que también incluye el error. El contrato que se debería comprobar es el de M6 · 5. Routing y CRUD (201 al crear, 404 si no existe).
- **Cómo lo comprobamos**: mutación «el 404 deja de devolverse»: ante una póliza inexistente el servicio responde 200.

```diff
--- a/main.py
+++ b/main.py
@@ -51,7 +51,7 @@ def firmar(numero: str, secreto: str) -> str:
 def _buscar_poliza(db: Session, id_poliza: int) -> Poliza:
     poliza = db.get(Poliza, id_poliza)
     if poliza is None:
-        raise HTTPException(status.HTTP_404_NOT_FOUND, f"no existe la póliza {id_poliza}")
+        raise HTTPException(status.HTTP_200_OK, f"no existe la póliza {id_poliza}")
     return poliza
```

```
$ pytest ia_tests_propuesta.py -v -p no:warnings
ia_tests_propuesta.py::test_crear_poliza_responde PASSED                 [ 16%]
ia_tests_propuesta.py::test_primas_variadas_son_aceptadas PASSED         [ 33%]
ia_tests_propuesta.py::test_consultar_poliza_inexistente PASSED          [ 50%]
ia_tests_propuesta.py::test_listar_polizas_no_falla PASSED               [ 66%]
ia_tests_propuesta.py::test_resumen_devuelve_lista PASSED                [ 83%]
ia_tests_propuesta.py::test_score_de_poliza_recien_creada PASSED         [100%]
============================== 6 passed in 1.88s ===============================
```

```
$ pytest tests/test_ia_corregido.py -q -p no:warnings
E       AssertionError: assert 200 == 404
E        +  where 200 = <Response [200 OK]>.status_code
E        +    where <Response [200 OK]> = get('/polizas/999999')
E        +      where get = <starlette.testclient.TestClient object at 0x7f31a0fee510>.get
FAILED tests/test_ia_corregido.py::test_consultar_poliza_inexistente_da_404
1 failed, 28 passed in 2.14s
```

- **Corrección**: cada aserción fija un solo resultado, el que dicta el contrato: `== 404` en GET, PUT, `POST /polizas/{id}/siniestros` y `/score` de algo inexistente (`test_consultar_poliza_inexistente_da_404`), `== 201` al crear, `== 422` para cada entrada inválida (parametrizado), `== 409` para el número repetido, y `== 200` más el contenido exacto al listar. No queda ningún `in (...)` ni `< 500`.

## Defecto 2

- **Qué está mal**: Hay tests que no prueban lo que afirman probar. (a) El bloque «Aislamiento de la base de datos» asegura que las pruebas «nunca escriben en la base de datos de la aplicación», pero sustituye un `get_db` **definido en el mismo archivo**, que la aplicación nunca utiliza: `app.dependency_overrides[get_db] = get_db` no reemplaza nada y todas las peticiones terminan escribiendo en `app.db` (además, si hubiera reemplazado la dependencia real con un `MagicMock`, no se habría probado ninguna persistencia). (b) `test_score_de_poliza_recien_creada` no verifica que la póliza se haya creado ni que la predicción quede registrada, que es justo lo que hace `/score`; `test_crear_poliza_responde` tampoco vuelve a leer lo que creó.
- **Por qué es un defecto** (módulo · sección): M10 · 5. TestClient de FastAPI y M10 · 6. Fixtures en pytest — el aislamiento se logra con `app.dependency_overrides` sobre la dependencia que la app realmente usa (`database.get_db`) y una base temporal; además, un test de persistencia debe leer lo que quedó guardado.
- **Cómo lo comprobamos**: mutación «se elimina un `commit`»: `/score` calcula y responde, pero la predicción no se guarda. Además comparamos `app.db` antes y después de correr la batería original.

```diff
--- a/main.py
+++ b/main.py
@@ -159,7 +159,6 @@ def puntuar(datos: PuntuacionEntrada, db: Session = Depends(get_db),
     prediccion = Prediccion(poliza=poliza, puntaje=puntaje,
                             alto_riesgo=puntaje > settings.umbral_alto_riesgo)
     db.add(prediccion)
-    db.commit()
     return PuntuacionSalida(numero=poliza.numero, puntaje=round(puntaje, 4),
                             alto_riesgo=prediccion.alto_riesgo)
```

```
$ md5sum app.db && pytest ia_tests_propuesta.py -v -p no:warnings && md5sum app.db
f7ba45a7a32a667e57e5448e4fe060c9  app.db
ia_tests_propuesta.py::test_crear_poliza_responde PASSED                 [ 16%]
ia_tests_propuesta.py::test_primas_variadas_son_aceptadas PASSED         [ 33%]
ia_tests_propuesta.py::test_consultar_poliza_inexistente PASSED          [ 50%]
ia_tests_propuesta.py::test_listar_polizas_no_falla PASSED               [ 66%]
ia_tests_propuesta.py::test_resumen_devuelve_lista PASSED                [ 83%]
ia_tests_propuesta.py::test_score_de_poliza_recien_creada PASSED         [100%]
============================== 6 passed in 1.94s ===============================
f6656890bfa74b59108d0a7ec1b249aa  app.db
```

La batería original sigue en verde aunque la predicción no se guarde, **y modificó `app.db`**: el
hash cambió porque escribió sus pólizas `POL-IA-...` en la base de la aplicación.

```
$ pytest tests/test_ia_corregido.py -q -p no:warnings
FAILED tests/test_ia_corregido.py::test_score_de_poliza_recien_creada_registra_la_prediccion[auto]
FAILED tests/test_ia_corregido.py::test_score_de_poliza_recien_creada_registra_la_prediccion[hogar]
FAILED tests/test_ia_corregido.py::test_score_de_poliza_recien_creada_registra_la_prediccion[vida]
3 failed, 26 passed in 2.18s
```

- **Corrección**: los tests usan la fixture `cliente` de `tests/conftest.py`, que sustituye `database.get_db` mediante `app.dependency_overrides` con una sesión sobre una base SQLite temporal (`tmp_path`) distinta para cada test; `app.db` queda intacta. Cada test relee lo que afirma: `test_crear_poliza_devuelve_201_y_queda_persistida` hace `GET /polizas/{id}` y compara; `test_score_de_poliza_recien_creada_registra_la_prediccion` exige que la creación devuelva 201 y que `/predicciones` incluya el `numero` con el mismo `alto_riesgo`; el `PUT` y el siniestro declarado también se vuelven a consultar.

## Defecto 3

- **Qué está mal**: Hay tests que a veces pasan y a veces fallan. `_poliza()` elige `tipo` con `random.choice` y `prima` con `random.uniform` **sin semilla**, «para no probar siempre lo mismo»: cada ejecución prueba un caso diferente y no se sabe cuál. Si el servicio falla solo con ciertos valores, el test falla solo en algunas ejecuciones y el error no se puede reproducir. A esto se suma que comparten la base real, cuyo estado cambia entre corridas.
- **Por qué es un defecto** (módulo · sección): M10 · 7. Reproducibilidad sin Docker — un resultado debe poder repetirse: la aleatoriedad se usa con semilla fija o, mejor aún en tests, los casos se enumeran (M10 · 6. Fixtures en pytest, `parametrize`).
- **Cómo lo comprobamos**: mutación «un validador rechaza lo que debería aceptar»: el `Literal` de `tipo` pierde `"vida"`. Ejecutamos la batería original diez veces seguidas sobre el mismo código mutado.

```diff
--- a/esquemas.py
+++ b/esquemas.py
@@ -4,7 +4,7 @@ from typing import Literal, Optional
 
 from pydantic import BaseModel, ConfigDict, Field, field_validator, model_validator
 
-TipoPoliza = Literal["auto", "hogar", "vida"]
+TipoPoliza = Literal["auto", "hogar"]
 EstadoSiniestro = Literal["abierto", "pagado", "rechazado"]
```

```
$ for i in $(seq 10); do echo "corrida $i: $(pytest ia_tests_propuesta.py -q -p no:warnings | tail -1)"; done
corrida 1: 6 passed in 1.83s
corrida 2: 6 passed in 1.77s
corrida 3: 6 passed in 1.75s
corrida 4: 6 passed in 1.90s
corrida 5: 1 failed, 5 passed in 1.97s
corrida 6: 1 failed, 5 passed in 1.92s
corrida 7: 6 passed in 1.85s
corrida 8: 1 failed, 5 passed in 1.76s
corrida 9: 1 failed, 5 passed in 1.99s
corrida 10: 1 failed, 5 passed in 1.79s
```

Cinco corridas verdes y cinco rojas con el mismo código: cuando el sorteo da `tipo="vida"`, la
creación responde 422, `/score` responde 404 y falla `test_score_de_poliza_recien_creada`; con los
otros tipos pasa. `test_crear_poliza_responde`, que también sortea `vida`, nunca falla porque
acepta el 422 (Defecto 1).

```
$ pytest tests/test_ia_corregido.py -q -p no:warnings
FAILED tests/test_ia_corregido.py::test_primas_y_tipos_validos_son_aceptados[0.01-vida]
FAILED tests/test_ia_corregido.py::test_primas_y_tipos_validos_son_aceptados[1-vida]
FAILED tests/test_ia_corregido.py::test_primas_y_tipos_validos_son_aceptados[1250000.5-vida]
FAILED tests/test_ia_corregido.py::test_primas_y_tipos_validos_son_aceptados[5000000-vida]
FAILED tests/test_ia_corregido.py::test_score_de_poliza_recien_creada_registra_la_prediccion[vida]
5 failed, 24 passed in 2.06s
```

- **Corrección**: datos fijos y casos enumerados: `@pytest.mark.parametrize` recorre **todos** los tipos (`auto`, `hogar`, `vida`) y primas en los bordes (`0.01`, `1`, `1_250_000.5`, `5_000_000`) en cada corrida, y los inválidos (`0`, `-1`, tipo `"moto"`, fechas invertidas, siniestro negativo) tienen su propio caso con `== 422`. Los números de póliza son fijos (`POL-IA-00001`) porque cada test parte de una base vacía. Con el mismo código, el resultado es siempre el mismo.
