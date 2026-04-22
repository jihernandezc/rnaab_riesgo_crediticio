<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>

<div style="position: sticky; top: 0; background-color: white; padding: 10px 0; border-bottom: 1px solid #ddd; z-index: 999; text-align: center; width: 100%;">
  <a href="index.html" style="text-decoration: none;">🏠 <b>Inicio</b></a> | 
  <a href="exploracion.html" style="text-decoration: none;">🔍 <b>Exploración</b></a> |
  <a href="modelos.html" style="text-decoration: none;">🧠 <b>Modelos</b></a> | 
  <a href="casodeuso.html" style="text-decoration: none;">🏦 <b>Caso de Uso</b></a> | 
  <a href="sitio-web.html" style="text-decoration: none;">🌐 <b>Sitio Web</b></a>
</div>

<br>

# Implementación de la Scorecard Crediticia

En el sector financiero, las probabilidades puras (0 a 1) resultan difíciles de interpretar para los tomadores de decisiones o el cliente final. Por ello, transformamos la salida de nuestra red neuronal en una escala de puntaje estandarizada mediante la metodología de **Log-Odds**.

### Metodología de Cálculo

Para la transformación, aplicamos una configuración estándar de la industria que garantiza la estabilidad y comparabilidad del modelo:

#### 1. Cálculo de los Odds
Primero, transformamos la probabilidad de default en la relación de "buenos" frente a "malos" pagadores:

<div align="center" markdown="1">

$$\text{Odds} = \frac{1 - p}{p}$$

*Ecuación 1. Cálculo de Odds a partir de la Probabilidad de Default (p)*
</div>

#### 2. Definición de Parámetros de Escalamiento
Para mapear estos *odds* a nuestra escala de 300-850 puntos, calculamos dos constantes fundamentales basadas en el **PDO** (Points to Double the Odds) y el **Score Base**:

*   **Factor:** Determina la magnitud del cambio en el puntaje.

<div align="center" markdown="1">

$$\text{Factor} = \frac{PDO}{\ln(2)}$$

*Ecuación 2. Factor de Escalamiento*
</div>

*   **Offset (Desplazamiento):** Ajusta la escala para que el **Score Base** corresponda a los **Odds Base** definidos.

<div align="center" markdown="1">

$$\text{Offset} = \text{Score}_{\text{base}} - (\text{Factor} \cdot \ln(\text{Odds}_{\text{base}}))$$

*Ecuación 3. Cálculo del Offset para Alinear el Score Base con los Odds Base*
</div>

#### 3. Fórmula Final del Scorecard
Combinando los componentes anteriores, la ecuación final que asigna el puntaje a cada cliente es:

<div align="center" markdown="1">

$$\text{Score} = \text{Offset} + \text{Factor} \cdot \ln(\text{Odds})$$

*Ecuación 4. Cálculo del Score Crediticio a partir de los Odds*
</div>

> **Nota:** En el código, se aplica un truncamiento (*clip*) al resultado final para asegurar que ningún puntaje exceda los límites operativos de [300, 850], manteniendo la consistencia con los estándares de la industria.

## Análisis del Score en la Población

Tras aplicar el modelo al conjunto de prueba, observamos lo siguiente:

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig8_distribucion_score.png" width="900" />
    <p><em>Figura 1. Distribución del Score Crediticio</em></p>
</div>

### Estadísticas Clave del Modelo:
*   **Score Mínimo:** 530.1
*   **Score Máximo:** 702.6
*   **Media General:** 605.0
*   **Mediana:** 604.0

Como se observa en los boxplots de la **Figura 1**, aunque existe una diferencia entre las medias, esta no es tan pronunciada:
*   **Buen pagador (Promedio):** 608.4 puntos.
*   **Mal pagador (Promedio):** 592.5 puntos.

Lo anterior nos permite entender por qué cuesta tanto separar a los buenos pagadores de los malos pagadores y es que sus distribuciones están solapadas en gran parte del tramo de los scores, lo cual influye en el resultado de las predicciones del modelo.

## Segmentación de Riesgo y Estrategia de Negocio

Para que el puntaje sea útil en el día a día, dividimos a los clientes en 5 niveles de riesgo. Es importante mencionar que aunque usamos la escala tradicional de 300 a 850 puntos (FICO), los cortes los calibramos según cómo se comportaron realmente los clientes en este dataset, no por estándares externos.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig8_segmentacion_riesgo.png" width="900" />
    <p><em>Figura 2. Segmentación por Nivel de Riesgo</em></p>
</div>

<div align="center" markdown="1">

*Tabla 1. Segmentación de Riesgo*

| Segmento | Rango de Score | % Población | Mora Observada | Acción Recomendada |
| :--- | :---: | :---: | :---: | :--- |
| **Muy bajo riesgo** | **> 650** | 2.9% | **2.4%** | Aprobación inmediata y tasas preferenciales. Zona de alta densidad de buenos pagadores. |
| **Bajo riesgo** | **620 - 650** | 21.7% | **8.6%** | Aprobación estándar. El volumen de "buenos pagadores" supera ampliamente al de "malos". |
| **Riesgo medio** | **590 - 620** | **48.4%** | **19.7%** | Zona crítica de solapamiento. Requiere validación de ingresos o garantías adicionales. |
| **Riesgo alto** | **565 - 590** | 24.3% | **36.6** | Mayor densidad de "malos pagadores" que de "buenos". Tasas de castigo o rechazo. |
| **Muy alto riesgo** | **< 565** | 2.7% | **57.6%** | Rechazo automático. Representa la cola izquierda donde la probabilidad de mora es máxima. |

</div>

Si comparamos estos resultados con un score crediticio tradicional (donde 650 suele ser un puntaje regular), en nuestro modelo un **650** es en realidad un puntaje **excelente**, ubicando al cliente en el segmento de **Muy Bajo Riesgo**. Esta diferencia en la escala ocurre por dos razones clave:

1.  Los datos provienen de *peer-to-peer lending* (LendingClub), un mercado con perfiles de riesgo más volátiles que la banca hipotecaria tradicional. El modelo ha mapeado el éxito dentro de este ecosistema específico, donde alcanzar un score superior a **650** indica una solidez financiera excepcional frente al promedio de la muestra (605).

2.  Notamos que al bajar de los **590 puntos**, entramos en el "valle de la mora". En este punto, la densidad de malos pagadores (color rojo en la Figura 1) empieza a ganar terreno rápidamente sobre los buenos pagadores. El modelo marca una frontera de seguridad clara en los **620 puntos**: por encima de este valor, la probabilidad de éxito es significativamente mayor, mientras que por debajo, el riesgo se vuelve errático y difícil de manejar.

Como se observa en la **Figura 2a**, el modelo logra una distribución balanceada. La mayoría de los clientes se sitúan en la zona media y baja de riesgo, permitiendo un flujo de caja saludable, mientras que la **Figura 2b** valida la potencia del modelo: a medida que el score baja, la columna roja de incumplimiento crece de forma exponencial, confirmando que el algoritmo realmente sabe quién va a fallar.

## ¿Qué mueve el Score?

Como en las Redes Neuronales no podemos ver los coeficientes así de fácil como en una regresión, usamos **Permutación**: básicamente agarramos una variable (digamos, el ingreso) y le desordenamos todos los datos al azar para dejarla inservible. Si al hacer esto el rendimiento del modelo (AUC-ROC) se desploma, confirmamos que esa variable era fundamental. Si el modelo ni se inmuta, es que esa variable no estaba aportando información útil para predecir el impago.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig10_importancia_variables.png" width="900" />
    <p><em>Figura 3. Impacto de Variables en el Score</em></p>
</div>

De la **Figura 3** se desprende que **`int_rate` (Tasa de interés):** es el factor más crítico para determinar el riesgo crediticio, seguida de **`annual_inc` (Ingreso anual);** el modelo depende de este dato para entender la capacidad de respuesta económica del cliente. El tercer lugar lo ocupa **`loan_amnt` (Monto del préstamo)** ya que la red le da mucho peso a qué tan grande es la deuda que se está asumiendo. **`dti` y `term`** son las siguientes en la lista. El modelo las usa para medir la presión financiera mensual y el tiempo de exposición al riesgo.

# Casos de Uso

El modelo de riesgo crediticio no opera en el vacío: detrás de cada predicción hay una persona con una historia financiera específica. Los siguientes casos ilustran cómo el scorecard apoya decisiones reales de crédito, con sus aciertos y sus limitaciones.

<style>
.cases-grid { display: flex; flex-direction: column; gap: 1.5rem; padding: 0.5rem 0; }
.case-card { background: #ffffff; border: 1px solid #e5e7eb; border-radius: 12px; overflow: hidden; }
.case-header { display: flex; align-items: center; gap: 14px; padding: 1rem 1.25rem 0.75rem; border-bottom: 1px solid #e5e7eb; }
.avatar { width: 46px; height: 46px; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-weight: 500; font-size: 14px; flex-shrink: 0; }
.case-meta { flex: 1; }
.case-name { font-size: 15px; font-weight: 600; color: #111827; margin: 0 0 2px; }
.case-role { font-size: 12px; color: #6b7280; margin: 0; }
.badge { display: inline-block; font-size: 11px; font-weight: 600; padding: 3px 10px; border-radius: 6px; margin-left: auto; white-space: nowrap; }
.badge-low  { background: #dcfce7; color: #166534; }
.badge-mid  { background: #fef9c3; color: #854d0e; }
.badge-high { background: #fee2e2; color: #991b1b; }
.case-body { display: grid; grid-template-columns: 1fr 1fr; }
.case-context { padding: 1rem 1.25rem; border-right: 1px solid #e5e7eb; }
.case-context p { font-size: 13.5px; color: #374151; line-height: 1.65; margin: 0 0 0.65rem; }
.case-context p:last-child { margin-bottom: 0; }
.case-context strong { color: #111827; }
.case-metrics { padding: 1rem 1.25rem; display: flex; flex-direction: column; gap: 10px; justify-content: center; }
.metric-row { display: flex; justify-content: space-between; align-items: center; }
.metric-label { font-size: 12px; color: #6b7280; }
.metric-value { font-size: 13px; font-weight: 600; color: #111827; }
.score-bar-wrap { margin-top: 2px; }
.score-track { height: 6px; border-radius: 3px; background: #e5e7eb; width: 100%; }
.score-fill  { height: 100%; border-radius: 3px; }
.score-labels { display: flex; justify-content: space-between; font-size: 10px; color: #9ca3af; margin-top: 3px; }
.verdict { padding: 0.6rem 1.25rem; border-top: 1px solid #e5e7eb; display: flex; align-items: flex-start; gap: 8px; font-size: 12.5px; color: #6b7280; line-height: 1.5; }
.verdict-icon { width: 18px; height: 18px; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: 700; flex-shrink: 0; margin-top: 1px; }
.icon-ok   { background: #dcfce7; color: #166534; }
.icon-fail { background: #fee2e2; color: #991b1b; }
@media (max-width: 580px) {
  .case-body { grid-template-columns: 1fr; }
  .case-context { border-right: none; border-bottom: 1px solid #e5e7eb; }
}
</style>

<div class="cases-grid">

<!-- CASO 1: ACIERTO -->

<div class="case-card">
  <div class="case-header">
    <div class="avatar" style="background:#fef3c7;color:#92400e;">C1</div>
    <div class="case-meta">
      <p class="case-name">María López <span style="font-size: 0.8em; color: #666;">(ID: 378291)</span></p>
      <p class="case-role">Debt consolidation · Utilización: 56.90% · DTI: 20.64%</p>
    </div>
    <span class="badge badge-medium">Riesgo medio</span>
  </div>
  <div class="case-body">
    <div class="case-context">
      <p>El solicitante buscaba consolidar deudas, presentando un nivel de endeudamiento relativamente alto (<strong>DTI: 20.64%</strong>) y una utilización de crédito elevada.</p>
      <p>El modelo asignó un score de <strong>593 puntos</strong>, ubicándolo en riesgo medio, reflejando correctamente señales de vulnerabilidad financiera.</p>
    </div>
    <div class="case-metrics">
      <div class="metric-row">
        <span class="metric-label">Score crediticio</span>
        <span class="metric-value">593 pts</span>
      </div>
      <div class="score-bar-wrap">
        <div class="score-track"><div class="score-fill" style="width:53%;background:#f59e0b;"></div></div>
        <div class="score-labels"><span>300</span><span>600</span><span>850</span></div>
      </div>
      <div class="metric-row">
        <span class="metric-label">Segmento detectado</span>
        <span class="metric-value" style="color:#f59e0b;">Riesgo Medio</span>
      </div>
      <div class="metric-row">
        <span class="metric-label">Resultado real</span>
        <span class="metric-value" style="color:#991b1b;">Mal pagador</span>
      </div>
    </div>
  </div>
  <div class="verdict">
    <div class="verdict-icon icon-ok">✓</div>
    <span><strong>ACIERTO:</strong> El modelo identificó correctamente el riesgo asociado a un alto nivel de endeudamiento y utilización, anticipando el incumplimiento del cliente.</span>
  </div>
</div>

<!-- CASO 2: ACIERTO -->

<div class="case-card">
  <div class="case-header">
    <div class="avatar" style="background:#dbeafe;color:#1d4ed8;">C2</div>
    <div class="case-meta">
      <p class="case-name">Carlos Peña<span style="font-size: 0.8em; color: #666;">(ID: 9388)</span></p>
      <p class="case-role">Wedding · Utilización: 35.00% · DTI: 16.14%</p>
    </div>
    <span class="badge badge-low">Bajo riesgo</span>
  </div>
  <div class="case-body">
    <div class="case-context">
      <p>El crédito fue solicitado para un evento personal (boda), con indicadores financieros relativamente saludables: utilización moderada y DTI controlado.</p>
      <p>El modelo asignó <strong>631 puntos</strong>, clasificando correctamente al cliente como de bajo riesgo.</p>
    </div>
    <div class="case-metrics">
      <div class="metric-row">
        <span class="metric-label">Score crediticio</span>
        <span class="metric-value">631 pts</span>
      </div>
      <div class="score-bar-wrap">
        <div class="score-track"><div class="score-fill" style="width:60%;background:#3b82f6;"></div></div>
        <div class="score-labels"><span>300</span><span>600</span><span>850</span></div>
      </div>
      <div class="metric-row">
        <span class="metric-label">Segmento detectado</span>
        <span class="metric-value" style="color:#1d4ed8;">Bajo Riesgo</span>
      </div>
      <div class="metric-row">
        <span class="metric-label">Resultado real</span>
        <span class="metric-value" style="color:#166534;">Buen pagador</span>
      </div>
    </div>
  </div>
  <div class="verdict">
    <div class="verdict-icon icon-ok">✓</div>
    <span><strong>ACIERTO:</strong> El modelo capturó adecuadamente un perfil estable, donde niveles moderados de deuda y uso del crédito indican bajo riesgo de incumplimiento.</span>
  </div>
</div>

<!-- CASO 3: ERROR -->

<div class="case-card">
  <div class="case-header">
    <div class="avatar" style="background:#fee2e2;color:#991b1b;">C3</div>
    <div class="case-meta">
      <p class="case-name">Alejandra Vanegas <span style="font-size: 0.8em; color: #666;">(ID: 104578)</span></p>
      <p class="case-role">Debt consolidation · Utilización: 90.20% · DTI: 15.06%</p>
    </div>
    <span class="badge badge-high">Riesgo alto</span>
  </div>
  <div class="case-body">
    <div class="case-context">
      <p>El solicitante presenta una <strong>utilización extremadamente alta (90.20%)</strong>, una de las variables más penalizadas por el modelo.</p>
      <p>Esto llevó a un score de <strong>576 puntos</strong>, clasificándolo como riesgo alto, a pesar de que su DTI se mantiene en niveles moderados.</p>
    </div>
    <div class="case-metrics">
      <div class="metric-row">
        <span class="metric-label">Score crediticio</span>
        <span class="metric-value">576 pts</span>
      </div>
      <div class="score-bar-wrap">
        <div class="score-track"><div class="score-fill" style="width:50%;background:#ef4444;"></div></div>
        <div class="score-labels"><span>300</span><span>600</span><span>850</span></div>
      </div>
      <div class="metric-row">
        <span class="metric-label">Segmento detectado</span>
        <span class="metric-value" style="color:#991b1b;">Riesgo Alto</span>
      </div>
      <div class="metric-row">
        <span class="metric-label">Resultado real</span>
        <span class="metric-value" style="color:#166534;">Buen pagador</span>
      </div>
    </div>
  </div>
  <div class="verdict">
    <div class="verdict-icon icon-fail">✗</div>
    <span><strong>ERROR:</strong> El modelo sobreponderó la utilización de crédito como señal de riesgo, sin capturar otros factores compensatorios, generando un falso negativo.</span>
  </div>
</div>
</div>

Los casos presentados anteriormente ilustran tres lecciones clave para interpretar los resultados de nuestro scorecard:

**1.** El caso de María**(593 pts)** evidencia la capacidad del modelo para identificar perfiles con señales claras de vulnerabilidad financiera. A pesar de no presentar valores extremos en una sola variable, la combinación de un **DTI elevado (20.64%)** y una utilización de crédito relativamente alta generó una clasificación de **Riesgo Medio**. Este resultado fue consistente con la realidad (mal pagador), lo que constituye un **acierto del modelo**. Este caso demuestra que el scorecard no depende únicamente de outliers, sino que puede capturar riesgos acumulativos derivados de múltiples factores moderados.

**2.** El caso de Carlos**(631 pts)** demuestra la importancia de la estabilidad en las variables clave. Con una **tasa de interés baja (11.49%)**, una utilización controlada (35%) y un nivel de endeudamiento manejable (**DTI: 16.14%**), el modelo clasificó correctamente al cliente como **Bajo Riesgo**. Este **acierto** refleja cómo la ausencia de señales de estrés financiero permite al scorecard identificar perfiles confiables. En este contexto, no se trata de valores excepcionalmente bajos, sino de un equilibrio general que reduce la probabilidad de incumplimiento.

**3.** El caso de Alejandra**(576 pts)** pone en evidencia una limitación del modelo frente a variables altamente penalizadas. La **utilización de crédito extremadamente alta (90.20%)** y una tasa elevada (22.20%) llevaron a clasificar al cliente como **Riesgo Alto**. Sin embargo, en la realidad se trata de un buen pagador, lo que constituye un **error (falso negativo)**. Este comportamiento sugiere que el modelo tiende a sobreponderar la utilización de crédito como señal de riesgo, sin capturar adecuadamente factores compensatorios como la disciplina de pago o la gestión efectiva de la deuda.

## Referencias 

**Experian. (2024).** What is a Good Credit Score? Experian Information Solutions, Inc. https://www.experian.com/blogs/ask-experian/credit-education/score-basics/what-is-a-good-credit-score/

**Fair Isaac Corporation. (2023).** What is a FICO Score? FICO. https://www.fico.com/en/products/fico-score
