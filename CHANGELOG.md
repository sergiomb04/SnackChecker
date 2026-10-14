# Changelog

Todas las modificaciones notables de este proyecto están documentadas bajo la especificación de [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/) y se adhieren a [Semantic Versioning](https://semver.org/lang/es/).

## [1.0.0] - 2026-10-14

### Added
- **Disparador Webhook (`POST /nutrition-check`):**
  - Recepción de solicitudes mediante endpoint HTTP en n8n.
  - Modo autodocumentado automático (`HTTP 200 JSON`) cuando la petición no incluye payload o viene vacía `{}`.
- **Validación Estricta de Entrada:**
  - Control de presencia y formato numérico del campo `barcode`.
  - Respuesta de error `HTTP 400 Bad Request` en JSON estructurado con ejemplos claros de sintaxis esperada.
- **Integración con Open Food Facts v2:**
  - Llamada HTTP GET sin necesidad de autenticación ni tokens.
  - Configuración `neverError: true` para interceptar respuestas 404 del catálogo sin abortar la ejecución del flujo.
  - Salida personalizada `HTTP 404 Not Found` en JSON con mensaje pedagógico y código de ejemplo válido.
- **Auditoría de Integridad de Datos:**
  - Detección de productos encontrados pero con registros de macronutrientes vacíos o ausentes.
  - Emisión de advertencia `HTTP 200 JSON` de "datos insuficientes", evitando catalogaciones erróneas.
- **Mapeo de Atributos:**
  - Aplanado de variables de producto, marcas y macronutrientes.
  - Acceso seguro al campo `energy-kcal_100g` mediante notación de corchetes.
- **Motor de Clasificación Nutricional (Nodo Code sobre ítem único):**
  - Aplicación de umbrales oficiales de semáforo para azúcar (`<5g`, `5-22.5g`, `>22.5g`), sal (`>1.5g`) y grasa (`>17.5g`).
  - Cálculo de la Puntuación de Preocupación (*Concern Score*) en escala discreta de 0 a 3.
  - Lógica de lectura Nutri-Score (`a/b`, `c`, `d/e`) con fallback automático a semáforo en productos con Nutri-Score desconocido.
  - Algoritmo de arbitraje del *peor caso* para proteger al consumidor ante alimentos altos en azúcares líquidos con bajo gramaje relativo.
- **Capa de Inteligencia Artificial Dinámica (Groq LLM / Llama 3.3):**
  - Enrutamiento condicional bifurcado según el veredicto de salud.
  - Cadena IA con tono alentador y constructivo para productos saludables y moderados.
  - Cadena IA con tono empático, reflexivo y delicadamente cauteloso para productos no saludables.
- **Formateo y Entrega Multisección:**
  - Ensamblado de reporte visual en texto plano (`text/plain; charset=utf-8`) con marcadores de semáforo (🟢 / 🔴 / 🟡).
  - Titular destacado, desglose de macronutrientes, energía en kcal y párrafo redactado por IA.
- **Documentación del Proyecto y Entregables:**
  - Exportación estandarizada del flujo en [`workflow.json`](file:///c:/Users/sergi/Downloads/temp/workflow.json) con documentación en línea en todos los nodos (`notesInFlow: true`).
  - Diagrama de flujo nativo en [`snackcheck_diagram.excalidraw`](file:///c:/Users/sergi/Downloads/temp/snackcheck_diagram.excalidraw) y renderizado en [`snackcheck_diagram.svg`](file:///c:/Users/sergi/Downloads/temp/snackcheck_diagram.svg).
  - Guía completa de uso y arquitectura en [`README.md`](file:///c:/Users/sergi/Downloads/temp/README.md).
  - Batería de pruebas [`TESTS.md`](file:///c:/Users/sergi/Downloads/temp/TESTS.md) y script automatizado [`run_tests.py`](file:///c:/Users/sergi/Downloads/temp/run_tests.py).
