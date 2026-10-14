# Diagrama de Arquitectura y Flujo — SnackCheck

Este documento detalla el diseño de flujo y las ramificaciones lógicas de la automatización **SnackCheck**. 

Archivos asociados para inspección visual y edición:
- Formato editable en Excalidraw: [`snackcheck_diagram.excalidraw`](file:///c:/Users/sergi/Downloads/temp/snackcheck_diagram.excalidraw) (arrastrar directamente en [excalidraw.com](https://excalidraw.com)).
- Formato vectorial directo: [`snackcheck_diagram.svg`](file:///c:/Users/sergi/Downloads/temp/snackcheck_diagram.svg).

---

## 1. Diagrama de Flujo (Mermaid)

```mermaid
flowchart TD
    Start(["POST /nutrition-check<br/>Webhook - Receive Barcode"]) --> CheckEmpty{"¿Cuerpo Vacío?<br/>Validate - Body Empty?"}

    %% Rama Autodocumentación
    CheckEmpty -- "Sí (Cuerpo vacío)" --> DocResp["Respond - Self Documentation<br/>HTTP 200 (JSON)"]

    %% Rama Validación
    CheckEmpty -- "No (Tiene datos)" --> CheckFormat{"¿Barcode Válido?<br/>Validate - Barcode Format"}
    CheckFormat -- "No (No numérico o ausente)" --> ErrResp["Respond - Validation Error<br/>HTTP 400 Bad Request (JSON)"]

    %% Consulta Externa
    CheckFormat -- "Sí (Numérico)" --> ApiCall["API Call - Open Food Facts Product<br/>GET /api/v2/product/{barcode}.json<br/>neverError: true"]

    %% Comprobación Catálogo
    ApiCall --> CheckFound{"¿Producto Existe?<br/>Validate - Product Found?"}
    CheckFound -- "No (404 / status: 0)" --> NotFoundResp["Respond - Product Not Found<br/>HTTP 404 Not Found (JSON)"]

    %% Comprobación Integridad
    CheckFound -- "Sí (status: 1)" --> CheckNutri{"¿Nutrientes Suficientes?<br/>Validate - Sufficient Nutriments?"}
    CheckNutri -- "No (Sin azúcar/grasa)" --> InsuffResp["Respond - Insufficient Data<br/>HTTP 200 OK (JSON)"]

    %% Procesamiento
    CheckNutri -- "Sí (Datos completos)" --> SetFlat["Set - Extract Flat Fields<br/>Notación corchetes: energy-kcal_100g"]
    SetFlat --> CodeClassifier["Code - Nutrition Classifier<br/>• Semáforo (Azúcar, Sal, Grasa)<br/>• Concern Score (0 a 3)<br/>• Regla del Peor Caso"]

    %% Decisión IA
    CodeClassifier --> CheckVerdict{"¿Veredicto?<br/>Validate - Health Threshold"}
    CheckVerdict -- "No Saludable" --> AICautious["AI - Health Verdict (Cautionary)<br/>Groq LLM: Tono Cauteloso y Empático"]
    CheckVerdict -- "Saludable / Moderado" --> AIEncouraging["AI - Health Verdict (Encouraging)<br/>Groq LLM: Tono Alentador y Positivo"]

    %% Consolidación y Salida
    AICautious --> MergeNodes["Merge - Consolidate Outputs<br/>(Mode: Append)"]
    AIEncouraging --> MergeNodes
    MergeNodes --> FormatText["Set - Format Text Verdict<br/>Titular + Marcadores (🟢/🔴) + IA"]
    FormatText --> RespondFinal["Respond - Plain Text Verdict<br/>HTTP 200 (text/plain; charset=utf-8)"]

    %% Estilos
    classDef webhook fill:#dbeafe,stroke:#2563eb,stroke-width:2px;
    classDef decision fill:#fef3c7,stroke:#d97706,stroke-width:2px;
    classDef error fill:#fee2e2,stroke:#dc2626,stroke-width:2px;
    classDef warning fill:#ffedd5,stroke:#ea580c,stroke-width:2px;
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px;
    classDef process fill:#f1f5f9,stroke:#475569,stroke-width:2px;
    classDef ai fill:#fed7aa,stroke:#c2410c,stroke-width:2px;

    class Start webhook;
    class CheckEmpty,CheckFormat,CheckFound,CheckNutri,CheckVerdict decision;
    class ErrResp,NotFoundResp error;
    class InsuffResp warning;
    class DocResp,RespondFinal success;
    class ApiCall,SetFlat,CodeClassifier,MergeNodes,FormatText process;
    class AICautious,AIEncouraging ai;
```

---

## 2. Diagrama Estructurado (ASCII Art)

```text
                         [Webhook - Receive Barcode]
                                      │ (POST /nutrition-check)
                                      ▼
                          [Validate - Body Empty?]
                             /                \
               (Cuerpo vacío)                  (Con cuerpo)
                     │                              │
          [Respond - Self Docs]                     ▼
               (HTTP 200)               [Validate - Barcode Format]
                                            /               \
                                    (Inválido)             (Válido)
                                        │                     │
                             [Respond - Validation]           ▼
                                   (HTTP 400)      [API Call - Open Food Facts]
                                                       (GET /api/v2/product/...)
                                                           /              \
                                                      (HTTP 404)       (HTTP 200)
                                                          │                 │
                                                [Respond - Not Found]       ▼
                                                     (HTTP 404)     [Validate - Sufficient Nutriments?]
                                                                         /                    \
                                                                    (Vacíos)                (Completos)
                                                                       │                         │
                                                         [Respond - Insufficient Data]           ▼
                                                                   (HTTP 200)        [Set - Extract Flat Fields]
                                                                                                 │
                                                                                                 ▼
                                                                                    [Code - Nutrition Classifier]
                                                                                                 │
                                                                                                 ▼
                                                                                    [Validate - Health Threshold]
                                                                                        /                  \
                                                                                (No Saludable)        (Saludable/Mod)
                                                                                      │                      │
                                                                               [AI Cautionary]        [AI Encouraging]
                                                                                      \                      /
                                                                                       ▼                    ▼
                                                                                     [Merge - Consolidate Outputs]
                                                                                                   │
                                                                                                   ▼
                                                                                      [Set - Format Text Verdict]
                                                                                                   │
                                                                                                   ▼
                                                                                     [Respond - Plain Text Verdict]
                                                                                               (HTTP 200)
```

---

## 3. Descripción de Bloques Funcionales y Convención de Nombres

Cada nodo sigue estrictamente el estándar `[Acción] - [Propósito]`:

1. **`Webhook - Receive Barcode`**: Punto de entrada HTTP expuesto en el puerto de n8n para peticiones POST.
2. **`Validate - Body Empty?`**: Evalúa si el payload carece de propiedades con `Object.keys($json.body || {}).length === 0`.
3. **`Respond - Self Documentation`**: Retorna el manual interactivo del servicio en JSON con código 200.
4. **`Validate - Barcode Format`**: Filtra mediante expresión regular que el código posea únicamente dígitos enteros.
5. **`Respond - Validation Error`**: Responde 400 Bad Request si la sintaxis del código es errónea.
6. **`API Call - Open Food Facts Product`**: Efectúa la llamada HTTP a Open Food Facts con protección `neverError: true`.
7. **`Validate - Product Found?`**: Discrimina el flag `status === 0` de Open Food Facts.
8. **`Respond - Product Not Found`**: Devuelve 404 Not Found cuando el artículo no consta en la base de datos abierta.
9. **`Validate - Sufficient Nutriments?`**: Previene la emisión de clasificaciones con registros sin macronutrientes.
10. **`Respond - Insufficient Data`**: Notifica en formato JSON que no es posible auditar el producto por falta de datos.
11. **`Set - Extract Flat Fields`**: Desanida las propiedades y utiliza notación de corchetes para `nutriments['energy-kcal_100g']`.
12. **`Code - Nutrition Classifier`**: Ejecuta la lógica matemática del semáforo, el Nutri-Score, la puntuación de preocupación (0 a 3) y el veredicto del peor caso sobre un ítem único.
13. **`Validate - Health Threshold`**: Bifurca el flujo hacia el prompt de IA adecuado según la gravedad del veredicto.
14. **`AI - Health Verdict (Cautionary)`**: Genera la recomendación empática y precavida para productos no saludables.
15. **`AI - Health Verdict (Encouraging)`**: Genera la recomendación constructiva para snacks saludables y equilibrados.
16. **`Merge - Consolidate Outputs`**: Une las ramas de ejecución con el modo *append*.
17. **`Set - Format Text Verdict`**: Da estructura visual al veredicto incorporando titular, emojis de semáforo y calorías.
18. **`Respond - Plain Text Verdict`**: Entrega el veredicto definitivo en texto plano con código HTTP 200.
