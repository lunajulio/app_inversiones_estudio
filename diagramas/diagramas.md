# Diagramas de solución — App de Inversiones CREAN

> Modelo conceptual y esquema de operación · Prueba técnica CREAN

La solución produce dos señales por cliente —**probabilidad de adopción** y **monto potencial de inversión**— y las combina en un valor esperado que prioriza la base. Este documento traza, primero, cómo esa señal entra al mundo de los procesos CREAN, y después, cómo se sostiene en el tiempo: quién la genera, quién la consume y qué la mantiene vigente.

---

## Parte 1 — Modelo conceptual y diagrama de procesos

### 1.1 De Bronze a Gold: dos modelos independientes que se multiplican

Descripción del flujo con arquitectura Medallion para comprender cómo se construye la señal de priorización que alimenta los procesos CREAN. La capa **Bronze** integra las fuentes; la capa **Silver** produce dos modelos independientes —uno de propensión y otro de monto—; y la capa **Gold** combina ambos en un valor esperado que llega al tablero y a los procesos CREAN.

![modelo de capas](diagramas\modelo_capas.png)

- **Bronze** integra 7 fuentes en una ABT única a nivel cliente (860,223 registros), con banderas de confianza que trazan qué tan completo es el histórico de cada fuente.
- **Silver 0** segmenta toda la base solo por comportamiento (sin demografía) en 8 clusters vía K-means — revela que el 71% de los clientes no tiene hoy producto de inversión activo, el tamaño real de la bolsa de oportunidad.
- **Silver 1** estima la probabilidad de adopción deliberadamente **sin** variables demográficas, para que el ranking se explique por comportamiento financiero observable.
- **Silver 2** estima el monto potencial con regresión Gamma (adecuada para montos positivos y asimétricos); para clientes sin historial de inversión usa *look-alike* demográfico contra clientes similares que sí invierten.
- **Gold** es la capa de negocio donde se presenta la metodología de solución, se exponen los resultados obtenidos, y se proponen acciones basadas en las evidencias a través de un tablero.

### 1.2 El mecanismo completo: de la ABT a los procesos CREAN

De izquierda a derecha: las fuentes se integran en una ABT gobernada, tres componentes analíticos leen de ella, sus salidas convergen en un motor de priorización, y ese motor alimenta directamente dos procesos CREAN. 

![diagrama de procesos](diagramas\diagrama_procesos.png)

> ⚙️ Los dos puntos de decisión marcan lo que de verdad cambia el resultado: **cuánto presupuesto comercial recibe cada decil**, y **si el resultado real obliga a recalibrar el modelo**. Las líneas punteadas son gobierno/retroalimentación, no cómputo directo.

#### Cobertura de los 7 procesos CREAN

De los siete procesos, la solución alimenta directamente tres, habilita el monitoreo de un cuarto, y deja los tres restantes fuera de su alcance — son procesos operativos que no dependen de un modelo de propensión o de monto.

| Proceso CREAN | Vínculo con la solución | Cobertura |
|---|---|---|
| Afiliar / Desafiliar al servicio | El modelo de propensión ordena a los clientes por probabilidad de adopción; el motor de priorización decide a quién contactar primero. | 🟢 directa |
| Gestionar ingresos y gastos | El modelo de monto potencial proyecta el volumen de inversión esperado en rango — insumo directo para metas de monetización del lanzamiento. | 🟢 directa |
| Administrar información | El diseño de la ABT —granularidad, consistencia de identificadores, trazabilidad vía `confianza_historia`— gobierna el ciclo de vida de estos datos derivados. | 🟢 directa |
| Monitorear el servicio | Los scores predichos crean la línea base contra la cual comparar resultados reales; la solución habilita el monitoreo, no lo opera por sí sola. | 🟠 indirecta |
| Gestionar el uso del servicio | No se modela el funcionamiento operativo de la app una vez el cliente ya la usa. | ⚪ fuera de alcance |
| Conciliar transacciones y contabilidad | No se procesan transacciones ni registros contables — la ABT parte de saldos ya conciliados. | ⚪ fuera de alcance |
| Administrar el servicio | No se gestionan ANS, parámetros ni novedades operativas del entorno. | ⚪ fuera de alcance |

#### Actores y puntos de decisión

Cada nodo del diagrama tiene un dueño y, en dos casos, una pregunta que solo un humano (o una regla de negocio explícita) puede responder.

**Actores involucrados**

- **Equipo de datos y analítica** — diseña la ABT y entrena los modelos de propensión y monto potencial.
- **Motor de priorización** — componente del sistema, no una persona: combina los dos scores en valor esperado y ordena la base en deciles.
- **Gestores comerciales** — ejecutan el contacto con los clientes que el motor priorizó.
- **Finanzas y producto** — usan el volumen proyectado para dimensionar metas de monetización del lanzamiento.
- **Gobierno de datos** — sostiene la trazabilidad y calidad de la información a lo largo de todo el ciclo.

**Puntos de decisión**

- **¿Qué decil recibe presupuesto comercial?** La calcula el motor de priorización; la ejecuta el equipo comercial en el lanzamiento.
- **¿El resultado real cae dentro del rango proyectado?** La responde el monitoreo; si no, dispara una recalibración del modelo.
- **¿Qué fuente o variable necesita ajuste?** La resuelve gobierno de datos a partir de lo que revele el monitoreo.

---

## Parte 2 — Esquema de operación en el ecosistema CREAN 

*Generar* y *actualizar* no son el mismo evento. **Generar** es entrenar el modelo y ajustar sus parámetros con datos nuevos. **Actualizar** es aplicar un modelo ya entrenado sobre la base más reciente para refrescar los scores. 

![esquema de operacion](diagramas\esquema_operacion.png)

> El anillo verde (Generar → Consumir → Seguimiento → Mantenimiento) es el ciclo que corre todos los meses sin intervención. El anillo violeta (Evolución) solo se activa cuando el seguimiento —el punto de decisión— detecta que el problema no se resuelve recalibrando, sino cambiando algo de fondo.

#### Cadencia y disparadores

Cada etapa tiene un dueño y una condición de disparo distinta — ninguna corre "porque sí" en cada ciclo.

| Etapa | Qué ocurre | Disparador |
|---|---|---|
| Generar y actualizar | Se reconstruye la ABT con los snapshots más recientes y se aplican los modelos ya entrenados sobre toda la base — scoring, no reentrenamiento. | 🟢 mensual |
| Consumir | Comercial prioriza contactos por decil; finanzas usa el rango de volumen esperado para sus metas; gobierno de datos revisa la confianza de la estimación. | 🟢 continuo |
| Seguimiento | Se compara la adopción y el monto reales del mes contra lo proyectado el mes anterior — calibración y *lift* real vs. esperado. | 🟢 mensual |
| Mantenimiento | Ajustes rutinarios: snapshots faltantes, umbrales de winsorización, recalibración menor cuando el sesgo se mueve dentro del rango ya conocido. | 🟢 trimestral |
| Evolución | Cambio estructural: reemplazar el proxy invesbot ∪ inversión virtual por adopción real de la app, incorporar validación temporal con más historia acumulada, o rediseñar el target si cambia el producto. | 🟣 condicional |

---

## Síntesis

La solución no reemplaza ningún proceso CREAN — le entrega a tres de ellos (**afiliación**, **ingresos y gastos**, **administración de información**) una señal priorizada y trazable. Además, deja habilitado un cuarto (**monitoreo**) para cerrar el ciclo con datos reales en vez de supuestos; y define de antemano cuándo un ajuste es rutina y cuándo exige repensar un supuesto, incluido el más importante: el día que la app tenga adopción real, el proxy que hoy la aproxima deja de ser necesario.
