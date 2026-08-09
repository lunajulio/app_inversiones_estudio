
# App de Inversiones CREAN — Estudio de propensión y monto potencial

> Prueba técnica CREAN (Creación y Aceleración de Nuevos Negocios) · flujo de trabajo implementado para resolver el reto

CREAN prepara el lanzamiento de una nueva app de inversiones para clientes actuales del banco, y necesita responder tres preguntas que hoy no tienen una visión integrada: **quién** es más probable que la adopte, **cuánto** invertiría, y **qué tan grande** es esa oportunidad de negocio. Este repositorio integra siete fuentes de datos financieros y demográficos, construye dos modelos analíticos sobre esa base, dimensiona la oportunidad resultante, y propone cómo esa solución opera dentro de los procesos de CREAN. Este documento resume el flujo de trabajo de punta a punta — el detalle técnico vive en los notebooks y en [diagramas.md](diagramas.md).

## Cómo se resuelve cada requerimiento del reto

| # | Requerimiento del reto | Dónde se resuelve |
|---|---|---|
| 1 | Diseñar un modelo de datos analítico (granularidad, consistencia, trazabilidad) | *Modelo de datos analítico* + Fase 1 · Integración y ETL |
| 2 | Analizar y caracterizar la base de clientes | Fase 2 · Caracterización de clientes |
| 3 | Construir uno o más modelos analíticos (adopción y monto potencial) | Fase 3 · Modelos analíticos |
| 4 | Dimensionar la oportunidad de negocio | Fase 4 · Dimensionamiento |
| 5 | Modelo conceptual y diagrama de procesos, ligado a los procesos CREAN | Fase 5 · Modelo conceptual CREAN |
| 6 | Esquema de operación dentro del ecosistema CREAN | Fase 6 · Esquema de operación |
| 7 | Tablero con recomendaciones accionables | Fase 7 · Tablero |

Los requerimientos 1 a 4 son el problema analítico en sí; los requerimientos 5 a 7 son cómo esa solución se comunica y vive dentro de CREAN. El flujo de trabajo de este proyecto sigue exactamente ese mismo orden.

---

## Diagnóstico de datos

El banco entregó siete fuentes en formato tabular — Clientes, Ahorro/Corriente, Bolsillos, Fiducuenta, CDT e Inversión Virtual, Invesbot y Estimador de ingresos —, cada una con su propio grano y calidad. El primer hallazgo, y el que termina condicionando todo el proyecto, es que el historial disponible cubre apenas ~13 meses y es disperso: ningún cliente tiene un registro mensual completo. Eso descarta de entrada cualquier enfoque basado en tendencias largas y obliga a trabajar con estadísticos por cliente (mediana, mínimo, máximo, estabilidad) en vez de series de tiempo.

## Supuestos

Cada decisión de transformación —qué ventana de tiempo usar, cómo tratar los valores extremos, qué hacer con un cliente sin un producto— quedó documentada en el punto del notebook donde se toma, no oculta detrás del código. Entre los supuestos más importantes: los valores extremos se winsorizan al percentil 99 solo entre tenedores del producto; a quien no tiene un producto se le asume saldo 0, no dato faltante; y el monto invertido se mide con dos regímenes distintos según qué tan confiable es la fecha de adopción de cada cliente.

**Riesgos:** la historia corta impide validar los modelos fuera de tiempo (con datos de un período distinto al usado para entrenarlos); la adopción digital se aproxima con un proxy (invesbot ∪ inversión virtual) porque la app real todavía no existe; y el monto potencial para quien nunca ha invertido es una extrapolación por parecido demográfico, no una medición directa — ningún cliente de la base se parece demasiado a un inversionista real.

## Modelo de datos analítico

La solución se organiza como una **arquitectura medallion** (bronze → silver → gold), que separa con claridad qué tan crudo o qué tan listo para negocio está cada dato:

- **Bronze** — las siete fuentes se integran en una única ABT (*Analytical Base Table*) a nivel cliente, con un identificador consistente y una bandera de confianza por variable que documenta qué tan sostenido está cada dato.
- **Silver** — sobre esa ABT corren tres análisis independientes: segmentación de clientes, propensión de adopción y monto potencial de inversión.
- **Gold** — la capa de negocio: combina propensión y monto en un valor esperado por cliente, listo para priorización comercial y consumo en el tablero.

El detalle visual de este flujo, con las herramientas y variables de cada paso, está en [diagramas.md](diagramas.md).

## Herramientas utilizadas

| Categoría | Herramientas |
|---|---|
| Procesamiento y modelado | Python (pandas, numpy), scikit-learn (K-means, HistGradientBoosting Classifier/Regressor, permutation importance), Optuna (tuning de hiperparámetros) |
| Entorno de desarrollo | Google Colab |
| Persistencia de datos | SQLite (ABT, paneles mensuales y tablas de score) |
| Visualización analítica | Matplotlib (dentro de los notebooks) |
| Tablero final | HTML, CSS y JavaScript |
| Diagramas | Lucidchart |
| Publicación y control de versiones | Git, GitHub, GitHub Pages |

---

## Fase 1 · Integración y ETL

Las siete fuentes se combinan en la ABT de la capa bronze: 860,223 clientes, un `numero_id` único por fila, verificada sin duplicación tras los merges. Se corrigen outliers de patrimonio, activos y pasivos con winsorización dirigida, y se calcula la bandera de confianza que acompaña a cada estadístico derivado.

## Fase 2 · Caracterización de clientes

Con K-means se agrupa a los clientes en 8 clusters, usando solo comportamiento financiero — sin edad, género ni segmento, para no confundir generación con comportamiento. El hallazgo central: el 71% de la base tiene poco o ningún producto con el banco, que es donde está la oportunidad comercial más grande.

## Fase 3 · Modelos analíticos

Dos modelos independientes, entrenados por separado. Uno de clasificación estima la probabilidad de adopción sin variables demográficas, para que la priorización no dependa del perfil del cliente. El otro, una regresión Gamma, estima el monto potencial de inversión, este sí usa variables demográficas, porque es la única forma de aproximar a alguien que nunca ha invertido, por parecido con quienes sí lo hacen.

## Fase 4 · Dimensionamiento

Probabilidad de adopción × monto potencial da el valor esperado por cliente. Sumado sobre toda la base, la oportunidad a doce meses ronda los $3.88 billones de pesos, con un 63% de ese valor concentrado en el 10% de clientes con mayor potencial, casi 80,000 personas.

## Conclusiones

El problema analítico (fases 1 a 4) queda resuelto con dos señales complementarias — quién y cuánto — que juntas priorizan a la base completa por decil. La limitación de fondo es la misma en las cuatro fases: con ~13 meses de historia, cada resultado es una aproximación razonada y documentada, no una certeza; el resto de la solución (fases 5 a 7) explica cómo esa aproximación se comunica y opera dentro de CREAN sin perder esa honestidad.

## Fase 5 · Modelo conceptual CREAN

El modelo conceptual traza el mismo recorrido bronze → silver → gold, pero mostrando actores, flujos de información y puntos de decisión: cuándo un cliente sale del ranking por ya tener el producto, cuándo cae en el decil prioritario, y cuándo su perfil activa una revisión de cumplimiento. Está ligado a los procesos CREAN de afiliación, gestión de ingresos/gastos y administración de información. Detalle completo en [diagramas.md](diagramas.md).

## Fase 6 · Esquema de operación

Define cómo la solución se mantiene viva después de entregada: con qué frecuencia se generan y actualizan los scores, quién los consume, y qué dispara un reentrenamiento frente a un simple ajuste rutinario. Detalle completo en [diagramas.md](diagramas.md).

## Fase 7 · Tablero

Tablero interactivo con las cifras, gráficas, supuestos y recomendaciones del estudio, dirigido a audiencia técnica y comercial por igual.

**→ [lunajulio.github.io/app_inversiones_estudio](https://lunajulio.github.io/app_inversiones_estudio/)**