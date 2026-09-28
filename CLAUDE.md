# Prueba técnica ArrowFin: Machine Learning Engineer

Guía de trabajo del proyecto. El enunciado está en `Technical_Assessment_Machine_Learning_Engineer.docx`
y los datos en `dataset/` (leer `dataset/README.md`).

## Principios

- **Lo más sencillo posible.** Solo pandas y numpy. Sin modelos de ML en la Tarea 1: reglas y estadística básica.
- **Explicable a alguien que no es técnico.** Cada decisión se puede contar en una frase.
- **Time-box de 3 horas.** Si algo no cabe, se anota en el README como "qué haría con más tiempo".
- **Honestidad con los datos.** Todo problema detectado en los datos se documenta en el README, no se oculta.
- **Declarar el uso de LLM.** El README indica qué partes se hicieron con Claude.

## Estructura del repositorio

```
dataset/            datos originales (no se modifican)
analysis.ipynb      notebook principal: carga, tabla de PnL, top 10 y chequeos (Tareas 1 y 5)
output/             resultados generados (pnl_sessions.csv, top10.csv)
slides/             1 o 2 slides para la mesa de riesgo (Tarea 3)
README.md           entregable: background, respuestas y reflexión
requirements.txt    dependencias (entorno en .venv, kernel de Jupyter "Python (ArrowFin)")
```

## Reglas de los datos (siempre aplicar)

1. **Valor por punto:** PnL = diferencia de precio × cantidad × `point_value_usd` del instrumento. Nunca un valor fijo.
2. **Sesión:** de 17:00 a 16:00 hora de Chicago (`America/Chicago`). La fecha de sesión es la del día en que cierra:
   `session_date = (filled_at en hora de Chicago + 7h).date()`.
3. **Comisiones:** se restan siempre (`commission_usd` ya viene por fill).
4. **Órdenes partidas:** para contar actividad se usan órdenes (`order_id`), no fills.
5. **Posiciones abiertas:** al corte (2026-08-25T14:30Z) se valoran con `market_prices.csv`.
6. **Trader, no cuenta:** los resultados se agregan por `trader_id` (algunos traders tienen varias cuentas).
7. **Casos especiales documentados:** cuentas con balance 0, cerradas o restringidas, traders con poco historial
   y precios sospechosos de MCL. Se reportan, no se eliminan en silencio.

## Metodología por tarea

### Tarea 1: Top 10 de traders con una sesión inusualmente mala

1. **Tabla de PnL:** una fila por trader y sesión, con PnL realizado (emparejamiento FIFO por cuenta e instrumento),
   PnL no realizado, comisiones, PnL neto y contratos operados.
2. **Sesión evaluada:** la de hoy (2026-08-25), que es la que el revisor verá mañana.
3. **Base de cada trader:** media y desviación estándar de su PnL neto en las sesiones **anteriores** a hoy
   (nunca se incluye la sesión evaluada).
4. **Score:** `z = (PnL hoy − media histórica) / desviación histórica`. Cuanto más negativo, más inusual.
   Así se compara a cada trader con su propio historial y no por el tamaño de la pérdida.
5. **Filtro:** solo cuentan sesiones con PnL neto negativo (una sesión "mala" tiene que perder dinero).
6. **Poco historial:** los traders con menos de 5 sesiones previas no tienen z fiable. Se listan aparte.
7. **Resultado:** top 10 ordenado por z, con PnL de hoy, media histórica y z, para que se entienda el porqué.

### Tarea 2: Etiquetas y evaluación (texto en el README)

- **(a) Etiqueta:** "sesión mala" = z ≤ −2 frente al propio historial, o que rompa una regla de la cuenta.
  Se explica dónde es ambigua y cómo medir el acuerdo entre dos personas (etiquetar la misma muestra y comparar con kappa de Cohen).
  Primer modelo: regresión logística con features de sesiones previas y validación con split temporal (entrenar en el pasado, validar en el futuro).
- **(b) Evaluación:** precisión del top 10 frente a lo que después pasó de verdad (liquidaciones y violaciones de reglas, datos que este dataset no trae).
  Riesgo de leakage: usar la sesión evaluada dentro de la base histórica, o features calculadas con información posterior.

### Tarea 3: Slides para la mesa de riesgo

- Una o dos slides, sin jerga: qué medimos, un ejemplo con un trader real del top 10, qué significa el score y sus límites.

### Tarea 4: Code review

- Revisión en formato de PR: qué bloquea y qué es comentario, cuál se arregla primero y el veredicto (no se aprueba).
- Puntos principales: la media histórica incluye la sesión actual (leakage), `POINT_VALUE` fijo,
  división por cero en `pace`, umbral de −1000 absoluto y cálculo O(n²).

### Tarea 5: Respuesta al cliente (T-020)

- Revisar primero los datos de T-020 frente a su propio historial.
- Máximo 8 frases: qué se midió, qué salió, sus límites y un siguiente paso concreto. Sin ceder ni cerrarse.

### Tarea 6: Reflexión

- Dos párrafos: la decisión de la que estoy más orgulloso con su costo, y qué haría con un segundo día.

## Convenciones

- Código legible con nombres claros y comentarios solo donde algo no es obvio.
- Todo el código va en el notebook `analysis.ipynb`, no en archivos `.py`.
- Entorno: `.venv` con `requirements.txt`. Abrir con `.venv/bin/jupyter notebook analysis.ipynb` y usar el kernel "Python (ArrowFin)".
- El notebook corre de principio a fin sin errores (Kernel → Restart & Run All) antes de darlo por terminado.
- Una sección por paso, con un título en markdown que explique qué hace y por qué.
- Celdas cortas que muestren resultados intermedios (`head()`, conteos) para facilitar el debugging.
- Montos en USD y fechas de sesión en formato `YYYY-MM-DD`.
