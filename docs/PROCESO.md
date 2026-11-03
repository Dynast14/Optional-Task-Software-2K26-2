# Bitácora de la tarea — FEKA STORE (Optional Task Software 2K26-2)

Este archivo registra, paso por paso, qué se hizo, por qué y con qué comando, para no perder el hilo.
El informe final (smells, técnicas, antes/después) está en [`INFORME.md`](INFORME.md).

> Entorno usado para ejecutar: Linux (VM de Cowork) con **Python 3.13.16** y **uv 0.13.4**.
> El entorno virtual se creó fuera del repositorio (`UV_PROJECT_ENVIRONMENT=$HOME/venv-feka`) para no ensuciar la carpeta.
> En Windows basta con `uv sync --locked` (crea `.venv`, que está en `.gitignore`).

---

## Paso 0 — Leer el enunciado y el código

- Leí `README.md`: 6 partes obligatorias (ejecutar, pruebas ≥ 80 %, SonarQube, ≥ 4 smells, ≥ 3 refactorizaciones, diagramas) y 3 adicionales opcionales.
- Leí todo el código (`main.py`, `app/database.py`, `app/products.py`, `app/customers.py`, `app/orders.py`) y la prueba inicial.
- Observación: casi toda la lógica está en `app/orders.py` → `OrderService`. Ahí están la mayoría de los smells.
- Git mostraba todos los archivos como modificados: era solo el fin de línea (CRLF de Windows). Se resolvió con `git config core.autocrlf true` (configuración local del repo); no hay cambios reales.

## Paso 1 — Ejecutar la aplicación (recorrido inicial)

Comandos:

```bash
uv sync --locked
uv run main.py            # en la VM se usó: uvicorn main:app --host 127.0.0.1 --port 8000
```

Se reprodujeron los 7 pasos del README con `curl` contra la API real (no con TestClient).
Resultado completo: [`evidencias/recorrido_antes.txt`](evidencias/recorrido_antes.txt).

| Paso | Esperado | Obtenido |
| --- | --- | --- |
| Cotizar / crear pedido | subtotal 5000, descuento 500, envío 1000, total 5500 | ✅ igual |
| Confirmar | 200, `confirmed` | ✅ |
| Stock del producto | 3 | ✅ 3 |
| Notificaciones | 1 | ✅ 1 |
| Repetir confirmación | 409 sin descontar stock | ✅ 409 y stock sigue en 3 |
| Swagger `/docs` | disponible | ✅ HTTP 200 |

> **Pendiente para ti (Windows):** sacar capturas de Swagger haciendo este mismo recorrido (el enunciado pide capturas).

## Paso 2 — Pruebas hasta ≥ 80 % de cobertura (ANTES de refactorizar)

Punto de partida: 4 pruebas, **67 %** de cobertura.

Se agregaron pruebas separadas por tipo:

| Archivo | Tipo | Qué cubre |
| --- | --- | --- |
| `tests/conftest.py` | fixtures | Base SQLite temporal por prueba (`tmp_path`), helpers para crear datos |
| `tests/test_pricing_rules.py` | **unitarias** | Descuentos por categoría, truncamiento a centavos, envíos y su límite exacto (9999 / 10000), umbral sobre el subtotal *antes* del descuento, fórmula del total, 404 de cliente/producto, que cotizar no toca stock, validaciones de `OrderInput` (líneas vacías, cantidad 0 o > 10000, productos repetidos, envío inválido, campos extra, tipos estrictos) |
| `tests/test_products_api.py` | integración | CRUD, orden por id, 404, 422, 204 sin cuerpo, no borrar producto con pedidos (409) |
| `tests/test_customers_api.py` | integración | CRUD, categoría por defecto, correo único (409 al crear y al editar), 404, 422, no borrar cliente con pedidos |
| `tests/test_orders_api.py` | integración | Ciclo de vida completo: crear, cotizar sin reservar stock, listado resumen vs. detalle, precios congelados, `PUT` recalcula con precios y categoría actuales, cancelados/confirmados son finales, notificación, stock exacto, stock consumido por otro pedido, 404 en todas las rutas, persistencia tras reinicio, **503** cuando SQLite no está disponible, `LAB_DB_PATH`, rollback de transacciones |
| `tests/test_orders.py` | integración (original) | Se dejó **sin modificar** |

Resultado: **84 pruebas, 99,67 % de cobertura** (`main.py` línea 38 es `uvicorn.run(...)` dentro de `if __name__ == "__main__"`, no se ejecuta en pruebas; no se excluyó nada).
Salida: [`evidencias/pytest_antes.txt`](evidencias/pytest_antes.txt).

```bash
uv run pytest --cov --cov-report=term-missing --cov-report=xml --cov-report=html --cov-fail-under=80
```
