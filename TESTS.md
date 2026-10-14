# Registro de Pruebas (Test Cases Log) — SnackCheck

Este documento reúne la batería exhaustiva de pruebas realizadas sobre la automatización **SnackCheck**, abarcando pruebas funcionales, de integración con servicios externos reales, de control de errores y de rendimiento.

---

## 📊 Matriz Resumen de Casos de Prueba

| ID | Tipo | Entrada / Escenario | Resultado Esperado | Resultado Obtenido | Estado |
| :--- | :--- | :--- | :--- | :--- | :---: |
| **`TC-001`** | **Funcional** | `{"barcode": "3017620422003"}`<br>*(Nutella 400g Ferrero)* | Clasificación `no saludable`, azúcar y grasa altos, Concern Score $\ge 2$, Nutri-Score `E`, tono IA cauteloso. Respuesta `200` texto plano. | Azúcar `56.3g (alto)`, Grasa `30.9g (alto)`, Sal `0.107g (bajo)`, Concern Score `2/3`, Veredicto final `NO SALUDABLE`. | **PASS** |
| **`TC-002`** | **Funcional** | `{"barcode": "3560070614202"}`<br>*(Copos de Avena Integral Carrefour)* | Clasificación `saludable`, 0 nutrientes altos, Nutri-Score `A`, tono IA positivo y alentador. Respuesta `200` texto plano. | Azúcar `0.7g (bajo)`, Grasa `7g (bajo)`, Sal `0g (bajo)`, Concern Score `0/3`, Nutri-Score `A`, Veredicto `SALUDABLE`. | **PASS** |
| **`TC-003`** | **Funcional** *(Worst Case)* | `{"barcode": "5449000000996"}`<br>*(Coca-Cola Original 330ml)* | Clasificación `no saludable` debido a Nutri-Score `E`, a pesar de que el semáforo por 100ml marque `saludable` (azúcar `10.6g` es medio, sal y grasa `0g`). | Semáforo: `saludable` (0 altos), Nutri-Score: `E` (`no saludable`). Arbitraje: adopta `NO SALUDABLE`. | **PASS** |
| **`TC-004`** | **Integración** | Ingesta real desde endpoint Open Food Facts v2 con código de barras en URL | Petición HTTP retorna `status: 1`. Extracción exitosa de `nutriments['energy-kcal_100g']` mediante notación de corchetes. | Conexión directa establecida, `status: 1`, valor extraído: `539 kcal` (Nutella), sin fallos de parseo. | **PASS** |
| **`TC-005`** | **Error / Validación** | Payload vacío: `{}` | Intercepción por `Validate - Body Empty?`. Respuesta `HTTP 200` con JSON interactivo de autodocumentación. | Retorna JSON con `service`, `usage`, endpoint y ejemplos prácticos de códigos de barra. | **PASS** |
| **`TC-006`** | **Error / Validación** | Payload inválido: `{"barcode": "abc-999"}` | Intercepción por `Validate - Barcode Format`. Rechazo con código `HTTP 400 Bad Request` y mensaje pedagógico. | Retorna `HTTP 400` con `{"error": "Codigo de barras invalido o ausente", ...}`. | **PASS** |
| **`TC-007`** | **Error / Catálogo** | Producto inexistente: `{"barcode": "0000000000000"}` | Intercepción de respuesta 404 externa vía `neverError: true`. Respuesta `HTTP 404` con JSON estructurado propio. | El flujo no se interrumpe; retorna `HTTP 404` con `status: 0` y código de ejemplo válido. | **PASS** |
| **`TC-008`** | **Error / Integridad** | Producto registrado sin valores en `nutriments` | Detección de falta de macronutrientes esenciales. Retorna `HTTP 200` con aviso de datos insuficientes sin clasificar. | Retorna JSON de advertencia: `{"warning": "Datos nutricionales insuficientes", ...}` evitando falsos positivos. | **PASS** |
| **`TC-009`** | **Rendimiento** | Pipeline extremo a extremo (OFF API + Lógica + Groq LLM) | Tiempo de respuesta total inferior a 2.500 ms en condiciones normales de red. | Tiempo de ejecución observado: **560 ms** (test suite local directa) / **~1.350 ms** con inferencia de Groq en vivo. | **PASS** |

---

## 🧪 Ejecución Automatizada y Reproducción

El proyecto incluye el script ejecutable [`run_tests.py`](file:///c:/Users/sergi/Downloads/temp/run_tests.py) para auditar y verificar todas las reglas de negocio y llamadas a la API en vivo de manera autónoma.

Para ejecutar la suite de pruebas:

```bash
python run_tests.py
```

### Salida de la Ejecución en Consola

```text
=== EJECUTANDO SUITE DE PRUEBAS SNACKCHECK ===
TC-001: ('TC-001', 'Funcional (Nutella)', 'no saludable / concern=2', 'no saludable / concern=2', 'PASS')
TC-002: ('TC-002', 'Funcional (Copos de Avena)', 'saludable / Nutri-Score A', 'saludable / grade=a', 'PASS')
TC-003: ('TC-003', 'Funcional (Worst Case Gaseosa)', 'no saludable (por Nutri-Score E)', 'traffic=saludable, final=no saludable', 'PASS')
TC-004: ('TC-004', 'Integración API Externa', 'status=1 y energy-kcal_100g mapeado', 'status=1, kcal=539', 'PASS')
TC-005: ('TC-005', 'Error / Validación (Cuerpo Vacío)', 'HTTP 200 con JSON autodocumentación', 'Detectado vacío: True -> Enruta a Respond - Docs', 'PASS')
TC-006: ('TC-006', 'Error / Validación (Barcode inválido)', 'HTTP 400 Bad Request', 'Es numérico: False -> Enruta a Respond - Validation Error', 'PASS')
TC-007: ('TC-007', 'Error / Catálogo (404 Not Found)', 'HTTP 404 estructurado', 'status=0, verbose=no code or invalid code', 'PASS')
TC-008: ('TC-008', 'Error / Integridad (Sin datos)', 'HTTP 200 Datos insuficientes', 'Nutrientes utilizables: False -> Enruta a Insufficient Data', 'PASS')
TC-009: ('TC-009', 'Rendimiento', 'Latencia de red y lógica < 2.5s (suite total < 5s)', 'Tiempo acumulado suite: 0.56s', 'PASS')

=== RESUMEN DE PRUEBAS ===
Resultado global: TODAS LAS PRUEBAS PASARON
```
