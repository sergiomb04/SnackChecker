# SnackCheck — Automatización Backend de Verificación Nutricional

[![n8n](https://img.shields.io/badge/n8n-Workflow%20Automation-FF6D5A?logo=n8n)](https://n8n.io)
[![Open Food Facts](https://img.shields.io/badge/API-Open%20Food%20Facts%20v2-22C55E)](https://world.openfoodfacts.org)
[![Groq](https://img.shields.io/badge/AI-Groq%20LLM%20(Llama%203.3)-F55036)](https://groq.com)
[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](file:///c:/Users/sergi/Downloads/temp/CHANGELOG.md)

---

## 📋 Propósito

**SnackCheck** es un motor de automatización backend desarrollado sobre **n8n** diseñado para evaluar de manera instantánea y amigable el perfil nutricional de alimentos empaquetados mediante la lectura de su código de barras.

El servicio:
1. Consulta información fidedigna y en tiempo real desde la API pública de **Open Food Facts** (sin requerir autenticación ni cuotas).
2. Audita tres macronutrientes críticos (**azúcar**, **sal** y **grasa** por cada 100 g) bajo las reglas oficiales de semáforo de salud pública.
3. Cruza la lectura de semáforo con el **Nutri-Score** oficial del producto aplicando la **regla de arbitraje del peor caso** (un producto solo es *saludable* si ambas métricas concuerdan).
4. Genera una recomendación asistida por IA (**Groq LLM / Llama 3.3**) con un **tono dinámico y adaptativo** (alentador para elecciones saludables/moderadas, empático y cauteloso para productos no saludables).
5. Devuelve un veredicto estructurado en **texto plano** con marcadores visuales (🟢 / 🔴 / 🟡) y titular, o respuestas enriquecidas en **JSON** con los códigos HTTP adecuados para autodocumentación y gestión de errores.

---

## ⚙️ Cómo Funciona

```
                     [Webhook POST /nutrition-check]
                                    │
                     [Validate - Body Empty?]
                        /                  \
             (Sí: cuerpo vacío)         (No: tiene datos)
                    │                              │
         [Respond - Self Docs]      [Validate - Barcode Format]
              (HTTP 200)                   /            \
                                    (Válido)          (Inválido)
                                       │                   │
                     [API Call - Open Food Facts]   [Respond - Validation Error]
                              /              \               (HTTP 400)
                         (HTTP 404)       (HTTP 200)
                             │                 │
                [Respond - Not Found]   [Validate - Sufficient Nutriments?]
                     (HTTP 404)              /                    \
                                        (Vacíos)                (Completos)
                                           │                         │
                             [Respond - Insufficient Data] [Set - Extract Flat Fields]
                                       (HTTP 200)                    │
                                                       [Code - Nutrition Classifier]
                                                                     │
                                                       [Validate - Health Threshold]
                                                           /                   \
                                                   (No Saludable)         (Saludable/Mod)
                                                         │                       │
                                                  [Groq LLM Cautious]    [Groq LLM Encouraging]
                                                         \                       /
                                                        [Merge - Consolidate Outputs]
                                                                     │
                                                        [Set - Format Text Verdict]
                                                                     │
                                                        [Respond - Plain Text Verdict]
                                                                  (HTTP 200)
```

### Arquitectura Paso a Paso

1. **Entrada y Validación Inicial (`Webhook - Receive Barcode` & `Validate - Body Empty?`):**
   - Escucha peticiones `POST` en `/nutrition-check`.
   - Si el payload es un objeto vacío `{}` o no contiene atributos, la automatización se **autodocumenta** respondiendo un JSON interactivo (`HTTP 200`) que expone su uso, esquemas y códigos de ejemplo.
2. **Validación Sintáctica (`Validate - Barcode Format`):**
   - Exige que el campo `barcode` exista y sea estrictamente numérico (`/^[0-9]+$/`).
   - En caso de error (e.g. `abc-123`), devuelve un `HTTP 400 Bad Request` en formato JSON estructurado.
3. **Ingesta de Datos Abiertos (`API Call - Open Food Facts Product`):**
   - Realiza una llamada `GET` a `https://world.openfoodfacts.org/api/v2/product/{barcode}.json?fields=product_name,brands,nutriscore_grade,nutriments`.
   - Incorpora `neverError: true` para evitar que un código 404 externo interrumpa la ejecución.
4. **Manejo de Catálogo Inexistente (`Validate - Product Found?`):**
   - Evalúa si `status === 0` o `status_verbose === "product not found"`.
   - Devuelve un `HTTP 404 Not Found` en JSON con mensaje pedagógico y código de ejemplo.
5. **Control de Integridad Nutricional (`Validate - Sufficient Nutriments?`):**
   - Si el producto existe pero los campos `sugars_100g` o `fat_100g` no están informados o vienen vacíos, evita emitir falsos veredictos y retorna un JSON `HTTP 200` notificando datos insuficientes.
6. **Aplanado de Variables (`Set - Extract Flat Fields`):**
   - Extrae `product_name`, `brands`, `nutriscore_grade`, `sugars_100g`, `salt_100g`, `fat_100g`.
   - Extrae el campo de energía con notación de corchetes: `{{ $json.product.nutriments['energy-kcal_100g'] }}`.
7. **Motor de Reglas y Decisión (`Code - Nutrition Classifier`):**
   - **Semáforo Nutricional (por 100 g):**
     - *Azúcar:* `< 5 g` (bajo), `5 – 22.5 g` (medio), `> 22.5 g` (alto).
     - *Sal:* `> 1.5 g` (alto), `<= 1.5 g` (bajo).
     - *Grasa:* `> 17.5 g` (alto), `<= 17.5 g` (bajo).
   - **Puntuación de Preocupación (Concern Score):** Cantidad de nutrientes etiquetados como `alto` (rango entero de 0 a 3), calculado sobre un ítem único sin iteradores de arreglo.
   - **Lectura de Semáforo:**
     - 0 nutrientes altos → `saludable`
     - 1 nutriente alto → `moderado`
     - 2 o 3 nutrientes altos → `no saludable`
   - **Lectura Nutri-Score:**
     - `a` o `b` → `saludable`
     - `c` → `moderado`
     - `d` o `e` → `no saludable`
     - Si es `unknown` o no existe, se respalda exclusivamente en el semáforo.
   - **Regla del Peor Caso:** El veredicto final adopta la peor clasificación (`no saludable` > `moderado` > `saludable`). Esto garantiza que productos como bebidas gaseosas azucaradas (bajas en gramos por 100 ml pero con Nutri-Score E) no se cataloguen falsamente como saludables.
8. **Inferencia de IA Adaptativa (`Validate - Health Threshold` & `Groq LLM`):**
   - Bifurca el flujo según el veredicto final:
     - **Ruta No Saludable:** Invoca al LLM con instrucciones de tono delicadamente cauteloso, empático y reflexivo sobre el consumo ocasional.
     - **Ruta Saludable / Moderada:** Invoca al LLM con tono entusiasta, cercano y alentador para el consumo diario.
9. **Formateo y Entrega (`Set - Format Text Verdict` & `Respond - Plain Text Verdict`):**
   - Fusiona las ramas en `Merge` (`append`).
   - Ensambla el reporte multisección en texto plano con marcadores visuales (🟢 / 🔴 / 🟡) y energía en kcal.
   - Responde con `Content-Type: text/plain; charset=utf-8` y código `HTTP 200`.

---

## 🛠️ Configuración y Despliegue

### Requisitos Previos
- Instancia activa de [n8n](https://n8n.io) (versión 1.x o superior).
- Cuenta gratuita en [Groq Cloud](https://console.groq.com) para obtener una API Key de Groq.
- Conexión a Internet abierta (Open Food Facts no requiere autenticación ni tokens).

### Pasos de Instalación
1. Descarga o abre el archivo [`workflow.json`](file:///c:/Users/sergi/Downloads/temp/workflow.json).
2. En n8n, crea un nuevo flujo de trabajo, abre el menú contextual (tres puntos) y selecciona **Import from File...** (o copia el contenido JSON y pégalo directamente en el lienzo con `Ctrl + V`).
3. En el nodo `Groq Chat Model`, asocia tu credencial existente de tipo **Groq API**.
4. Activa el flujo mediante el interruptor **Active** (o utiliza el endpoint de prueba de n8n para desarrollo interactivo).

---

## 🚀 Guía de Uso con Ejemplos Reales

### 1. Producto No Saludable (Nutella)

```bash
curl -X POST http://localhost:5678/webhook/nutrition-check \
  -H "Content-Type: application/json" \
  -d '{"barcode": "3017620422003"}'
```

**Respuesta Obtenida (`HTTP 200 text/plain`):**
```text
🔴 Veredicto SnackCheck: NO SALUDABLE
Producto: Nutella (Ferrero)
Puntuacion de Preocupacion: 2/3 (Nutri-Score: E)

Analisis por 100g:
🔴 Azucar: 56.3g (alto)
🔴 Grasa: 30.9g (alto)
🟢 Sal: 0.107g (bajo)
⚡ Energia: 539 kcal

Recomendacion del Asistente:
Nutella tiene un sabor delicioso e inconfundible, pero debido a su alta concentracion de azucares y grasas refinadas es aconsejable disfrutarla de forma ocasional y en porciones moderadas dentro de una alimentacion equilibrada.
```

---

### 2. Producto Saludable (Copos de Avena Integral)

```bash
curl -X POST http://localhost:5678/webhook/nutrition-check \
  -H "Content-Type: application/json" \
  -d '{"barcode": "3560070614202"}'
```

**Respuesta Obtenida (`HTTP 200 text/plain`):**
```text
🟢 Veredicto SnackCheck: SALUDABLE
Producto: Copos de Avena Integral (Carrefour)
Puntuacion de Preocupacion: 0/3 (Nutri-Score: A)

Analisis por 100g:
🟢 Azucar: 0.7g (bajo)
🟢 Grasa: 7g (bajo)
🟢 Sal: 0g (bajo)
⚡ Energia: 372 kcal

Recomendacion del Asistente:
¡Excelente eleccion! Estos copos de avena integral presentan un perfil nutricional sobresaliente, sin azucares anadidos y con grasas de calidad, siendo ideales para acompanar tus desayunos o meriendas diarias con gran aporte energetico.
```

---

### 3. Arbitraje Peor Caso (Coca-Cola Original)

```bash
curl -X POST http://localhost:5678/webhook/nutrition-check \
  -H "Content-Type: application/json" \
  -d '{"barcode": "5449000000996"}'
```

**Respuesta Obtenida (`HTTP 200 text/plain`):**
```text
🔴 Veredicto SnackCheck: NO SALUDABLE
Producto: Coca-Cola Original (Coca-Cola)
Puntuacion de Preocupacion: 0/3 (Nutri-Score: E)

Analisis por 100g:
🟢 Azucar: 10.6g (medio)
🟢 Grasa: 0g (bajo)
🟢 Sal: 0g (bajo)
⚡ Energia: 42 kcal

Recomendacion del Asistente:
Aunque por cada 100 ml sus niveles individuales no superan el umbral critico de un alimento solido, su clasificacion Nutri-Score E refleja un aporte significativo de azucares simples sin valor nutricional adicional, por lo que se recomienda reservar su consumo para momentos excepcionales.
```

---

### 4. Solicitud Vacía (Autodocumentación)

```bash
curl -X POST http://localhost:5678/webhook/nutrition-check \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Respuesta Obtenida (`HTTP 200 application/json`):**
```json
{
  "service": "SnackCheck Nutrition Checker API",
  "status": "active",
  "description": "Envia un codigo de barras para auditar los macronutrientes de un snack.",
  "usage": {
    "method": "POST",
    "endpoint": "/nutrition-check",
    "required_body": {
      "barcode": "string (solo digitos numericos)"
    },
    "examples": [
      { "product": "Nutella", "barcode": "3017620422003" },
      { "product": "Copos de Avena", "barcode": "3560070614202" },
      { "product": "Coca-Cola", "barcode": "5449000000996" }
    ]
  }
}
```

---

## 🛡️ Manejo de Errores y Matriz de Códigos HTTP

| Código HTTP | Escenario | Content-Type | Estructura de Respuesta |
| :--- | :--- | :--- | :--- |
| **`200 OK`** | Petición sin cuerpo `{}` | `application/json` | Objeto explicativo con campos requeridos y ejemplos de uso. |
| **`400 Bad Request`** | Código de barras no provisto o no numérico (`abc-999`) | `application/json` | `{"error": "Codigo de barras invalido...", "example": {"barcode": "3017620422003"}}` |
| **`404 Not Found`** | Código no existe en Open Food Facts (`0000000000000`) | `application/json` | `{"error": "Producto no encontrado", "status": 0, "example_valid_barcode": "3017620422003"}` |
| **`200 OK`** | Producto existe pero carece de datos nutricionales | `application/json` | `{"warning": "Datos nutricionales insuficientes", "product_name": "..."}` |
| **`200 OK`** | Análisis exitoso y clasificado | `text/plain` | Reporte multisección con titular, semáforo visual y párrafo de IA. |

---

## ⚠️ Limitaciones Conocidas

1. **Dependencia de la Base de Datos:** Los datos se obtienen de Open Food Facts, una plataforma colaborativa abierta. Alimentos locales artesanales o referencias muy recientes pueden carecer de registro o de desglose completo de macronutrientes.
2. **Estandarización por 100g / 100ml:** La evaluación se calcula de manera homogénea por 100 g de producto, por lo que no pondera el tamaño real de la porción sugerida por el fabricante.
3. **Dependencia de Red y Cuota del LLM:** El tiempo de respuesta puede oscilar entre 0.8s y 2.0s según la latencia de la API de Groq Cloud y Open Food Facts.
