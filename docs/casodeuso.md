<div style="position: sticky; top: 0; background-color: white; padding: 10px 0; border-bottom: 1px solid #ddd; z-index: 999; text-align: center; width: 100%;">
  <a href="index.html" style="text-decoration: none;">🏠 <b>Inicio</b></a> | 
  <a href="exploracion.html" style="text-decoration: none;">🔍 <b>Exploración</b></a> |
  <a href="modelos.html" style="text-decoration: none;">🧠 <b>Modelos</b></a> | 
  <a href="casodeuso.html" style="text-decoration: none;">🏦 <b>Caso de Uso</b></a> | 
  <a href="sitio-web.html" style="text-decoration: none;">🌐 <b>Sitio Web</b></a>
</div>

<br>

# Implementación de la Scorecard Crediticia

En el sector financiero, las probabilidades puras (0 a 1) resultan difíciles de interpretar para los tomadores de decisiones o el cliente final. Por ello, transformamos la salida de nuestro **Ensamble Híbrido** en una escala de puntaje estandarizada mediante la metodología de **Log-Odds**.

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

> **Nota técnica:** En el código, se aplica un truncamiento (*clip*) al resultado final para asegurar que ningún puntaje exceda los límites operativos de $[300, 850]$, manteniendo la consistencia con los estándares de la industria.

## Análisis del Score en la Población

Tras aplicar el modelo al conjunto de prueba, observamos lo siguiente:

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig8_distribucion_score.png" width="900" />
    <p><em>Figura 1. Distribución del Score Crediticio</em></p>
</div>

### Estadísticas Clave del Modelo:
*   **Score Mínimo:** 549.9
*   **Score Máximo:** 767.7
*   **Media General:** 644.1
*   **Mediana:** 643.4

Como se observa en los boxplots de la **Figura 1**, aunque existe una diferencia entre las medias, esta no es tan pronunciada:
*   **Buen pagador (Promedio):** 648.7 puntos.
*   **Mal pagador (Promedio):** 627.6 puntos.

Lo anterior nos permite entender por qué cuesta tanto separar a los buenos pagadores de los malos pagadores y es que sus distribuciones están solapadas en gran parte del tramo de los scores, lo cual influye en el resultado de las predicciones del modelo.

## Segmentación de Riesgo y Estrategia de Negocio

Para que el puntaje sea útil en el día a día, dividimos a los clientes en 5 niveles de riesgo. Es importante mencionar que aunque usamos la escala tradicional de 300 a 850 puntos (FICO), los cortes los calibramos según cómo se comportaron realmente los clientes en este dataset, no por estándares externos.

<div align="center" markdown="1">

*Tabla 1. Segmentación de Riesgo*

| Segmento | Score | % Población | Mora Real | Acción Recomendada |
| :--- | :---: | :---: | :---: | :--- |
| **Muy bajo riesgo** | > 700 | 3.1\% | **2.8\%** | Aprobación automática y mejores tasas. |
| **Bajo riesgo** | 660 - 700 | 24.4\% | **8.3\%** | Aprobación preferencial. |
| **Riesgo medio** | 630 - 660 | 40.4\% | **18.6\%** | Evaluación manual o pedir garantías. |
| **Riesgo alto** | 550 - 630 | 32.1\% | **38.3\%** | Rechazo o tasas muy altas (castigo). |
| **Muy alto riesgo** | < 550 | 0.0\%^* | **100.0\%** | Rechazo automático inmediato. |

</div>

> **Nota sobre el riesgo extremo:** El modelo detectó con 100% de puntería al grupo de "Muy alto riesgo", pero ojo: es una muestra muy pequeña (un solo caso en el test). Es una buena señal de precisión, pero lo tomamos como un indicador preliminar.

Si se compara con un score tradicional, por ejemplo un score de **640**, podría considerarse "aceptable". Pero en nuestro modelo, ese cliente ya está en **Riesgo Medio** con casi un 19% de probabilidad de impago. Esto pasa por dos razones clave:

1.  Los datos vienen de LendingClub (*peer-to-peer lending*), donde el riesgo suele ser mayor que en un crédito hipotecario tradicional, entonces el modelo aprendió a ser más precavido.
2.  Notamos que al bajar de los 630 puntos, la mora se duplica (pasa de 18% a 38%). Ahí es donde el modelo marca una frontera de seguridad clara: por debajo de eso, el riesgo deja de ser manejable.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig8_segmentacion_riesgo.png" width="900" />
    <p><em>Figura 2. Segmentación por Nivel de Riesgo</em></p>
</div>

Como se observa en la **Figura 2a**, el modelo logra una distribución balanceada. La mayoría de los clientes se sitúan en la zona media y baja de riesgo, permitiendo un flujo de caja saludable, mientras que la **Figura 2b** valida la potencia del modelo: a medida que el score baja, la columna roja de incumplimiento crece de forma exponencial, confirmando que el algoritmo realmente sabe quién va a fallar.

## ¿Qué mueve el Score?

Utilizando los pesos del modelo, identificamos qué variables tienen mayor impacto en el movimiento del puntaje de un cliente:

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig10_importancia_variables.png" width="900" />
    <p><em>Figura 3. Impacto de Variables en el Score</em></p>
</div>

De la **Figura 3** se desprende que el **DTI (Relación Deuda/Ingreso)** es el factor más crítico para determinar el riesgo crediticio. Clientes con un DTI alto son vistos como sobreendeudados, lo que asfixia su capacidad de pago y reduce drásticamente su score. En contraste, variables como la **Antigüedad Crediticia** y el **Ingreso Anual** actúan como colateral natural, elevando el puntaje y reduciendo la probabilidad de incumplimiento.

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
    <div class="avatar" style="background:#dbeafe;color:#1d4ed8;">ML</div>
    <div class="case-meta">
      <p class="case-name">María López <span style="font-size: 0.8em; color: #666;">(ID: 227402)</span></p>
      <p class="case-role">Consolidación de deuda · 1 año antig. · DTI: 8.14%</p>
    </div>
    <span class="badge badge-low">Bajo riesgo</span>
  </div>
  <div class="case-body">
    <div class="case-context">
      <p>María solicitó un crédito para <strong>consolidar sus deudas</strong>. A pesar de llevar solo un año en su empleo actual, su baja relación deuda/ingreso (DTI) del 8.14% fue una señal positiva de salud financiera.</p>
      <p>Con un score sólido de <strong>686 puntos</strong>, el modelo la clasificó como un perfil confiable dentro del segmento de bajo riesgo, permitiendo una aprobación ágil.</p>
    </div>
    <div class="case-metrics">
      <div class="metric-row">
        <span class="metric-label">Score crediticio</span>
        <span class="metric-value">686 pts</span>
      </div>
      <div class="score-bar-wrap">
        <div class="score-track"><div class="score-fill" style="width:70%;background:#3b82f6;"></div></div>
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
    <span><strong>ACIERTO:</strong> El scorecard interpretó correctamente la estabilidad de María a pesar de su corta antigüedad laboral, validando que un DTI bajo es un fuerte mitigador de riesgo.</span>
  </div>
</div>

<!-- CASO 2: ACIERTO -->
<div class="case-card">
  <div class="case-header">
    <div class="avatar" style="background:#fef3c7;color:#92400e;">CR</div>
    <div class="case-meta">
      <p class="case-name">Carlos Ríos <span style="font-size: 0.8em; color: #666;">(ID: 33913)</span></p>
      <p class="case-role">Mejoras del hogar · 10 años antig. · DTI: 13.10%</p>
    </div>
    <span class="badge badge-low">Bajo riesgo</span>
  </div>
  <div class="case-body">
    <div class="case-context">
      <p>Carlos buscaba financiamiento para <strong>remodelar su casa</strong>. Su perfil destaca por una excelente estabilidad laboral (10 años en la empresa) y un endeudamiento controlado.</p>
      <p>El sistema le asignó <strong>661 puntos</strong>. Aunque está cerca del límite del segmento, su trayectoria profesional pesó positivamente en la decisión final del algoritmo.</p>
    </div>
    <div class="case-metrics">
      <div class="metric-row">
        <span class="metric-label">Score crediticio</span>
        <span class="metric-value">661 pts</span>
      </div>
      <div class="score-bar-wrap">
        <div class="score-track"><div class="score-fill" style="width:65%;background:#f59e0b;"></div></div>
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
    <span><strong>ACIERTO:</strong> El modelo premió la antigüedad de Carlos. La predicción fue certera, demostrando que la estabilidad laboral sigue siendo uno de los pilares de la solvencia.</span>
  </div>
</div>

<!-- CASO 3: ERROR -->
<div class="case-card">
  <div class="case-header">
    <div class="avatar" style="background:#fee2e2;color:#991b1b;">AP</div>
    <div class="case-meta">
      <p class="case-name">Alejandra Paredes <span style="font-size: 0.8em; color: #666;">(ID: 723670)</span></p>
      <p class="case-role">Pequeño negocio · 4 años antig. · DTI: 12.68%</p>
    </div>
    <span class="badge badge-high">Riesgo alto</span>
  </div>
  <div class="case-body">
    <div class="case-context">
      <p>Alejandra solicitó un crédito para su <strong>negocio propio</strong>. A pesar de tener un DTI moderado y 4 años de experiencia, el propósito del crédito (Small Business) es históricamente uno de los más riesgosos en el dataset.</p>
      <p>El score cayó a <strong>613 puntos</strong>, situándola en Riesgo Alto. Lamentablemente, Alejandra no pudo cumplir con sus obligaciones financieras.</p>
    </div>
    <div class="case-metrics">
      <div class="metric-row">
        <span class="metric-label">Score crediticio</span>
        <span class="metric-value">613 pts</span>
      </div>
      <div class="score-bar-wrap">
        <div class="score-track"><div class="score-fill" style="width:56%;background:#ef4444;"></div></div>
        <div class="score-labels"><span>300</span><span>600</span><span>850</span></div>
      </div>
      <div class="metric-row">
        <span class="metric-label">Segmento detectado</span>
        <span class="metric-value" style="color:#991b1b;">Riesgo Alto</span>
      </div>
      <div class="metric-row">
        <span class="metric-label">Resultado real</span>
        <span class="metric-value" style="color:#991b1b;">Mal pagador</span>
      </div>
    </div>
  </div>
  <div class="verdict">
    <div class="verdict-icon icon-fail">✗</div>
    <span><strong>ALERTA DE RIESGO:</strong> Aunque el scorecard identificó el peligro (Riesgo Alto), este caso se marca como "Error" en la predicción binaria si el umbral de decisión no fue lo suficientemente estricto para evitar el otorgamiento.</span>
  </div>
</div>

</div>

Los casos presentados ilustran tres lecciones clave para interpretar los resultados de nuestro scorecard:

**1.** Un puntaje alto como el de **María (686 pts)** indica una probabilidad estadística muy alta de éxito basada en su bajo endeudamiento. El modelo permite que la entidad "apueste" con confianza por perfiles donde los datos muestran una ruta clara hacia el cumplimiento, optimizando los tiempos de respuesta.

**2.** El caso de **Carlos (661 pts)** demuestra que el modelo no se fija en una sola variable. Aunque su relación de deuda (DTI) era más alta que la de María, su **estabilidad laboral de 10 años** actuó como un contrapeso positivo. El scorecard balancea estos factores de forma objetiva para no excluir a clientes que, aunque no son perfectos, son financieramente estables.

**3.** El caso de **Alejandra (613 pts)** subraya la importancia de detectar variables de alto impacto como el "Propósito del Crédito". Al identificar que los préstamos para *Small Business* tienen una mayor tasa de mora, el modelo asignó un **Riesgo Alto** de forma preventiva. Esto permite a la entidad proteger su capital o exigir garantías adicionales en segmentos donde la probabilidad de impago es real y medible. Aunque el modelo acertó al clasificar a Alejandra como de alto riesgo, si la entidad decide otorgar el crédito sin medidas de mitigación, el resultado real de incumplimiento se convierte en un "error" desde la perspectiva de la predicción binaria. Esto resalta la necesidad de que las decisiones comerciales estén alineadas con los insights del modelo para maximizar su efectividad.

## Referencias 

**Experian. (2024).** What is a Good Credit Score? Experian Information Solutions, Inc. https://www.experian.com/blogs/ask-experian/credit-education/score-basics/what-is-a-good-credit-score/

**Fair Isaac Corporation. (2023).** What is a FICO Score? FICO. https://www.fico.com/en/products/fico-score
