<div style="position: sticky; top: 0; background-color: white; padding: 10px 0; border-bottom: 1px solid #ddd; z-index: 999; text-align: center; width: 100%;">
  <a href="index.html" style="text-decoration: none;">🏠 <b>Inicio</b></a> | 
  <a href="exploracion.html" style="text-decoration: none;">🔍 <b>Exploración</b></a> |
  <a href="modelos.html" style="text-decoration: none;">🧠 <b>Modelos</b></a> | 
  <a href="casodeuso.html" style="text-decoration: none;">🏦 <b>Caso de Uso</b></a> | 
  <a href="sitio-web.html" style="text-decoration: none;">🌐 <b>Sitio Web</b></a>
</div>

<br>

# 🔍 Exploración y Análisis Descriptivo

El éxito de un modelo de riesgo crediticio no depende únicamente de la complejidad de su algoritmo, sino de la calidad y comprensión de los datos subyacentes. En esta sección, desglosamos el [*Credit Risk Dataset*](https://www.kaggle.com/datasets/ranadeep/credit-risk-dataset/data), validamos nuestras hipótesis iniciales y preparamos el terreno para el modelado con Redes Neuronales.

## 1. Análisis del Diccionario e Inspección de Datos

El conjunto de datos original cuenta con **75 variables** que abarcan información demográfica, financiera y del historial crediticio de los solicitantes. 

### 1.1 Transformación de la Variable Objetivo (`loan_status`)
Siguiendo las mejores prácticas de la industria, transformamos la variable multiclase original en una **variable binaria (Target)**:
*   **0 (Buen pagador):** Incluye "Fully Paid" y "Does not meet the credit policy. Status:Fully Paid".
*   **1 (Mal pagador):** Incluye "Charged off", "Late (31-120 days)", "Default" y "Does not meet the credit policy. Status:Charged Off".
*   **Excluidos:** Registros "Current", "In Grace Period", etc., por su incertidumbre temporal.

### 1.2 Selección y Limpieza
Inicialmente, se eliminaron columnas con más del 50% de valores faltantes. Este umbral se estableció porque:

*   Cuando una variable tiene más del 50% de datos faltantes, intentar "completar" (imputar) esos valores con la media, mediana o moda introduce un sesgo masivo. Estaríamos inventando la mitad de la información, lo que puede llevar al modelo a encontrar patrones inexistentes (overfitting al ruido).
*   Si a la mayoría de los individuos no se les registró esa variable, es probable que no sea un requisito estándar para todos los perfiles crediticios o que el campo sea opcional.

Además, tras inspeccionar el diccionario, se identificó que no todas las variables son aptas para el entrenamiento. A continuación se presentan los criterios de selección y limpieza aplicados:

<div align="center" markdown="1">

### Tabla 1. Variables eliminadas y justificación

| Columna | Justificación para eliminarla |
| :--- | :--- |
| `id`, `member_id` | Son identificadores únicos aleatorios. No tienen relación con el comportamiento financiero del cliente. |
| `url` | Es un enlace web. No aporta información sobre el perfil de riesgo. |
| `desc`, `title` | Son campos de texto libre. Requerirían Procesamiento de Lenguaje Natural (NLP). El campo `purpose` (categórico) ya resume esta información. |
| `zip_code` | Los primeros 3 dígitos del código postal tienen demasiadas categorías (alta cardinalidad) y están correlacionados con `addr_state`. |
| `issue_d` | Es la fecha en que se entregó el dinero. No describe al individuo, sino al momento económico (inflación, tasas de ese mes), lo cual puede sesgar el modelo hacia el pasado. |
| **Fuga de Datos (Leakage)** | **Variables que ocurren después de otorgar el crédito:** |
| `total_pymnt`, `total_pymnt_inv` | Es el total pagado hasta hoy. Un cliente que ya pagó mucho tendrá un score bajo de riesgo por definición. No sirve para predecir *antes* del préstamo. |
| `total_rec_prncp`, `total_rec_int`, `total_inc_late_fee` | Capital, intereses y moras recibidos. Esta información se genera a medida que el cliente paga (o deja de pagar). |
| `recoveries`, `collection_recovery_fee` | Estas variables **solo existen si el cliente ya incumplió**. Si el modelo ve que `recoveries > 0`, sabrá con 100% de certeza que el cliente es "malo", pero esto es trampa (fuga de datos). |
| `last_pymnt_d`, `last_pymnt_amnt` | Información sobre el último pago realizado. Es información del comportamiento posterior a la decisión. |
| `last_credit_pull_d` | La fecha más reciente en que se consultó el buró. Cambia constantemente después de dar el crédito. |
| `next_pymnt_d` | Fecha del próximo pago. Solo relevante para créditos vigentes. |
| `policy_code` | Generalmente tiene un solo valor (1 o 2) para todo el dataset. Si no varía, no ayuda al modelo a distinguir entre buenos y malos. |
| `funded_amnt`, `funded_amnt_inv` | A veces el banco aprueba menos de lo que el cliente pidió (`loan_amnt`). Usamos solo `loan_amnt` porque es el riesgo solicitado inicialmente. |
| `out_prncp`, `out_prncp_inv` | Representan el capital que aún no se ha pagado. En el momento de solicitar el crédito, este valor siempre es igual al monto prestado; si varía, es porque el préstamo ya está en curso, revelando indirectamente si el cliente es "bueno" o "malo" antes de que el modelo prediga. |
| `grade`| Es una versión simplificada de `sub_grade`. Al mantener ambas, introducimos información redundante que no añade poder predictivo a la Red Neuronal y puede complicar la interpretación de los coeficientes en la scorecard final. |

</div>

Adicionalmente, no incluimos **`loan_status`**, ya que es la fuente de nuestra variable `target`. Por lo que si la dejamos dentro de las variables predictoras, el modelo tendrá una precisión del 100% de forma artificial, porque la respuesta está dentro de la pregunta.

### 1.3 Transformación de Variables Categóricas

Para codificar las variables categóricas, primero validamos si una variable es **ordinal** (tiene un orden lógico) o **nominal** (es solo una etiqueta). Esto es importante porque el tipo de codificación que se le aplicará a cada variable dependerá de esta clasificación. A continuación se muestra una tabla con la clasificación de cada variable categórica presente en el Dataframe:

<div align="center" markdown="1">

### Tabla 2. Clasificación de variables categóricas y tratamiento propuesto

| Variable | Tipo de Dato Original | Tratamiento Propuesto | Justificación |
| :--- | :--- | :--- | :--- |
| **`term`** | String (" 36 months") | **Numérico (36, 60)** | El plazo es una magnitud física. Un plazo más largo suele implicar mayor riesgo de impago por exposición al tiempo. |
| **`grade`** | String ("A1", "B2") | **Ordinal (1 a 35)** | Representa el riesgo asignado por el analista. Hay un orden jerárquico claro donde A1 < A2 < ... < G5. |
| **`emp_length`** | String ("10+ years") | **Ordinal (0 a 10)** | Representa estabilidad laboral. "10+" es el valor máximo y "< 1 year" el mínimo. Se mapea a una escala lineal. |
| **`earliest_cr_line`** | Fecha ("Jan-1985") | **Numérico (Meses)** | Una fecha por sí sola no dice nada. Lo que importa es la **antigüedad crediticia**. Calculamos los meses transcurridos desde esa fecha hasta el presente del dataset. |
| **`verification_status`** | String | **Nominal (One-Hot)** | Aunque hay niveles (Verified vs Not), no hay una escala numérica lineal clara de riesgo entre ellos. |
| **`home_ownership`** | String | **Nominal (One-Hot)** | No hay un orden natural. Por ejemplo, "RENT" no es "mejor" que "MORTGAGE" en términos de riesgo. |
| **`purpose`** / **`addr_state`** | String | **Nominal (One-Hot)** | Son categorías. No hay un orden lógico entre ellas. |
| **`pymnt_plan`** / **`initial_list_status`** / **`application_type`** | String | **Nominal (One-Hot)** | Son banderas (flags) binarias o de estado. |

</div>

**Sobre la fecha de referencia para `earliest_cr_line`:**
Dado que este dataset es histórico (finaliza aproximadamente en 2015-2018), usaremos la **fecha máxima detectada en la columna** como "Hoy". Así medimos la antigüedad relativa de cada cliente. Sin embargo, esto asume un punto de corte estático para todos los préstamos. En un escenario real, sería ideal tener la fecha de cada solicitud de crédito para calcular la antigüedad exacta en ese momento. Pero dado el dataset disponible, esta es una aproximación razonable.

## 2. Análisis Descriptivo

### 2.1 Distribución de la Variable Objetivo

Uno de los hallazgos más críticos del análisis descriptivo es la distribución de nuestra variable objetivo.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/main/output/figs/fig1_distribucion_target.png" width="700" />
    <p><em>Figura 1. Distribución de la variable objetivo</em></p>
</div>

> *Nota: Se observa que hay aproximadamente **3.6 veces más buenos pagadores** que malos pagadores. Este desbalance requiere estrategias específicas de evaluación (como F1-Score o AUC-ROC) en lugar de la precisión simple (Accuracy).*

---

### 2.2 Análisis de Variables Numéricas vs Target

A continuación se compara cómo se distribuyen las variables numéricas más importantes entre buenos y malos pagadores usando boxplots. Si una variable tiene distribuciones diferentes entre las dos clases, es una buena señal de que será útil para la elaboración del modelo. Las variables analizadas son:
*   **int_rate — tasa de interés**
*   **dti — relación deuda/ingreso**
*   **annual_inc — ingreso anual**
*   **loan_amnt — monto del crédito**
*   **installment — cuota mensual**
*   **revol_util — utilización del crédito rotativo**

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/main/output/figs/fig2_numericas_vs_target.png" width="700" />
    <p><em>Figura 2. Distribución de variables numéricas por tipo de pagador</em></p>
</div>

De la Figura 2, se puede observar que **int_rate** es la señal más clara de riesgo, ya que las cajas del gráfico correspondiente a esta variable para buenos y malos pagadores es la que más separada está la una de la otra, lo cual suele generar mayor confianza en la predicción de un cliente, lo que nos permite analizar que altas tasas de crédito están asociadas al incumplimiento en el pago de las mismas.

El **dti** también parece separar bien los grupos y sugiere que quienes ya tienen muchas deudas en relación con su ingreso tienen mayor probabilidad de incumplir, pero no es la mejor señal de separación.

El **annual_inc** muestra que los buenos pagadores ganan más que los malos pagadores pero no es la mejor variable para decidir porque sus medias están muy cercanas y no separa a ambos grupos con claridad.

El **loan_amnt** tampoco sirve mucho para el modelo, pero señala que los malos pagadores piden más en sus créditos que los buenos pagadores.

El **installment** tiene un índice y comportamiento muy similar a la variable anterior lo cual puede indicar correlación entre ambas variables.

El **revol_util** señala que los malos pagadores usan más dinero del crédito disponible que quienes son buenos pagadores, aunque la diferencia no es enorme y tampoco servirá mucho como factor diferencial en el modelo.

### 2.3 Análisis de Variables Categóricas vs Target

De la misma manera analizaremos las variables categóricas vs el target, comparando cómo se distribuyen las variables categóricas más importantes entre buenos y malos pagadores usando gráficos de barras. De igual manera, si una variable tiene distribuciones diferentes entre las dos clases, es una buena señal de que será útil para el modelo.

Las variables que analizaremos son:
*   **grade — calificación crediticia del préstamo**
*   **purpose — propósito del crédito**
*   **home_ownership — tipo de vivienda del solicitante**
*   **verification_status — estado de verificación de ingresos**
*   **emp_length — antigüedad laboral**
*   **term — plazo del crédito**

> Acá usamos grade porque no la eliminamos por ser irrelevante, sino porque se codificó con sub_grade, y usar sub_grade acá no es posible porque tiene demasiadas categorías. Sin embargo, el análisis de grade nos da una idea de cómo se relaciona la calificación crediticia con el incumplimiento.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/main/output/figs/fig3_categoricas_vs_target.png" width="700" />
    <p><em>Figura 3. Tasa de incumplimiento por variable categórica</em></p>
</div>

Basándonos en la Figura 3, podemos deducir que **grade** es la variable categórica con mayor poder de separación pues tiene una tendencia marcada de que mientras mayor sea el grado (más cercano a G) mayor será la probabilidad de que el cliente incumpla con el pago de su crédito.

**Purpose** muestra que los créditos para **small bussiness** son los más riesgosos y que para **wedding** y **car** son los más seguros, también aparenta ser una buena variable para separar ambos casos.

El **home_ownership** parece ser más estable por lo que es más riesgoso para nuestro modelo tomar decisiones basadas en dicha variable.

**verification_status** suele tener un comportamiento similar a la anterior variable y parece que la clasificación es contraintuitiva puesto que los clientes con ingresos **verified** presentan más incumplimiento.

**emp_length** Es la variable con menor poder discriminativo de todas debido a la similitud presentada en todos los posibles valores de la misma con respecto a la media.

Y el **term** si puede ser una variable muy útil, ya que nos permite analizar que los créditos a 60 meses presetan mayor índice de incumplimiento frente a los créditos a 32 meses, lo cual suena bastante lógico.

### 2.4 Correlación entre las Variables y el Target

La correlación de Pearson nos permite cuantificar la relación lineal entre cada variable numérica y el target. Esto es crucial para identificar cuáles variables tienen mayor poder predictivo y deben ser priorizadas en la ingeniería de características y en la interpretación del modelo.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/main/output/figs/fig4a_correlaciones_target.png" width="700" />
    <p><em>Figura 4. Correlación de variables numéricas con el target</em></p>
</div>

La Figura 4 permite identificar qué variables tienen una relación lineal directa con la probabilidad de impago. Se pueden extraer tres conclusiones clave:

1.  **Principales Inductores de Riesgo (Barras Rojas):** La **tasa de interés (`int_rate`)** es el predictor individual más fuerte (0.255). Esto sugiere que el mercado ya aplica una prima de riesgo: a mayor riesgo percibido, mayor tasa, lo que a su vez dificulta el pago. Le siguen el **DTI (0.134)** y el **uso de líneas revolventes (0.100)**, confirmando que el sobreendeudamiento es un precursor del incumplimiento.
2.  **Factores de Mitigación (Barras Verdes):** El **ingreso anual (`annual_inc`)** y el **balance total de cuentas (`tot_cur_bal`)** presentan correlaciones negativas. Esto indica que niveles más altos de ingresos y de patrimonio actúan como "escudos" o factores de protección que reducen la probabilidad de caer en mora.
3.  **Señales de Comportamiento:** Variables como el número de consultas en los últimos 6 meses (`inq_last_6mths`) y la morosidad previa (`delinq_2yrs`) muestran una correlación positiva con el riesgo, validando que el comportamiento histórico de búsqueda de crédito y fallos previos son predictores relevantes.

---

### **Sugerencia de redacción para el reporte:**
*“Al combinar ambas gráficas, observamos que el perfil de riesgo está dominado por el costo del dinero (tasas) y la capacidad de pago (DTI e ingresos). La altísima correlación entre el monto del crédito y la cuota mensual sugiere que la carga financiera total es el factor determinante, más allá del capital inicial solicitado.”*


### 2.5 Análisis de Correlación entre Variables

El análisis de correlación entre las variables numéricas nos ayuda a identificar posibles redundancias o multicolinealidades que podrían afectar el rendimiento del modelo. Por ejemplo, si dos variables están altamente correlacionadas entre sí, podrían estar aportando información similar al modelo, lo que podría llevar a un sobreajuste.

Esperaríamos encontrar baja correlación entre las variables, debido al proceso de selección y limpieza que se realizó, pero es importante validar esto con un mapa de calor de correlaciones.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/main/output/figs/fig4b_heatmap_correlaciones.png" width="700" />
    <p><em>Figura 5. Heatmap de correlaciones entre variables relevantes</em></p>
</div>

Al analizar la Figura 5, se pueden extraer las siguientes conclusiones:

1.  La matriz confirma que la tasa de interés (`int_rate`) tiene una relación moderada con la utilización del crédito (`revol_util`, 0.33) y el DTI (0.18). Esto indica que la tasa de interés "captura" parte del riesgo de otras variables, pero aún mantiene un valor predictivo independiente y único de 0.25 frente al target.
2.  El ingreso anual (`annual_inc`) muestra correlaciones bajas o moderadas con el resto de las variables (máximo 0.39 con `tot_cur_bal`), lo que significa que aporta información fresca y distinta que no está presente en el monto del préstamo o las tasas de interés.

Además, en la matriz también se observa una correlación casi perfecta (0.95) entre **`loan_amnt` (monto del préstamo)** e **`installment` (cuota mensual)**. Esto estadísticamente nos advierte que ambas variables aportan información redundante. Sin embargo, desde la perspectiva de riesgo representan conceptos distintos:

1.  **Exposición vs. Capacidad de Pago (Flujo de Caja):**
    *   **`loan_amnt` (Monto del préstamo):** Representa la **exposición total** del banco. Es la cantidad de capital que está en riesgo de pérdida total (*Loss Given Default*).
    *   **`installment` (Cuota mensual):** Representa la **presión sobre el flujo de caja** mensual del cliente. Un cliente puede tener una deuda total alta (`loan_amnt`), pero si el plazo es largo, la cuota (`installment`) puede ser manejable. Inversamente, una deuda pequeña con una cuota muy alta puede causar un impago inmediato.

2.  **Captura Implícita del Plazo (`term`):**
    *   La relación entre el monto y la cuota está determinada por el **plazo** (36 o 60 meses) y la **tasa de interés**. Al mantener ambas, permitimos que la Red Neuronal aprenda la "densidad" del préstamo. Dos préstamos de \$10,000 pueden tener cuotas muy diferentes; esa diferencia es una señal de riesgo que el modelo debe capturar.

3.  **Resiliencia de las Redes Neuronales:**
    *   A diferencia de los modelos lineales clásicos, las **Redes Neuronales Artificiales (ANN)** son excelentes manejando variables altamente correlacionadas. A través de sus capas ocultas y el ajuste de pesos, la red puede extraer interacciones no lineales entre ellas (como la proporción cuota/ingreso de forma interna) sin que la multicolinealidad sesgue los coeficientes como ocurriría en una regresión.

4.  **Valor Predictivo Complementario:**
    *   Al observar la gráfica de correlación con el target, `loan_amnt` tiene 0.071 y `installment` 0.055. Aunque parecidos, no son idénticos. Esos "0.016" puntos de diferencia sugieren que cada variable aporta una pizca de información única que, sumada en las capas de la red, mejora la precisión del score final.

Es por esto que a pesar de la alta correlación detectada entre `loan_amnt` e `installment` (0.95), se tomó la decisión de conservar ambas variables para el entrenamiento de la Red Neuronal. 

## 3. Planteamiento de Hipótesis

Tras el análisis descriptivo, formalizamos las siguientes hipótesis de riesgo basadas en las diferencias observadas entre buenos y malos pagadores. Estas hipótesis guiarán la interpretación de los resultados del modelo y la elaboración de la scorecard final:

<div align="center" markdown="1">

### Tabla 3. Hipótesis de riesgo basadas en el análisis descriptivo

| Variable | Hallazgo | Hipótesis de Riesgo |
| :--- | :--- | :--- |
| `int_rate` | Malos: 16.0% vs Buenos: 13.3% | **Costo del Riesgo:** A mayor tasa de interés, se genera un fenómeno de "selección adversa" y una carga financiera mayor que asfixia al deudor, elevando la probabilidad de impago. |
| `dti` | Malos: 18.6 vs Buenos: 16.1 | **Capacidad de Pago:** Un DTI elevado indica que el cliente tiene poco margen de maniobra ante imprevistos económicos, haciendo que el nuevo crédito sea difícil de sostener. |
| `annual_inc` | Malos: \$66k vs Buenos: \$74k | **Efecto Escudo:** Los ingresos altos actúan como un colchón financiero. A menor ingreso, mayor vulnerabilidad ante la volatilidad económica. |
| `revol_util` | Malos: 59.2% vs Buenos: 53.2% | **Dependencia del Crédito:** Un uso alto de líneas rotativas sugiere que el cliente vive al límite de su capacidad crediticia y utiliza el crédito para gastos corrientes. |
| `term` | 60 meses tiene > mora | **Exposición Temporal:** Los créditos a largo plazo están sujetos a más eventos de vida (despido, enfermedad), aumentando la incertidumbre del pago final. |
| `inq_last_6mths` | Correlación positiva (0.053) | **Búsqueda Desesperada:** Un alto número de consultas recientes indica una necesidad urgente de liquidez, lo cual es una señal de alerta (*red flag*) financiera. |

</div>

## 4. Conclusiones y Aprendizajes del EDA

El proceso de exploración no solo sirvió para limpiar datos, sino para generar aprendizajes estratégicos sobre el fenómeno del riesgo en este portafolio:

1.  **El "Círculo Vicioso" de la Tasa:** El análisis confirma que la tasa de interés es el predictor más fuerte. Existe un riesgo intrínseco donde el banco, al intentar protegerse cobrando más a los perfiles riesgosos, termina aumentando la probabilidad de que estos incumplan debido al costo de la deuda.
2.  **Identificación del Perfil Crítico:** El "Mal Pagador" típico no es necesariamente alguien sin ingresos, sino alguien **sobreendeudado**. La combinación de un DTI > 18% y una utilización de tarjetas (`revol_util`) > 60% es una señal mucho más potente que el nivel de ingresos por sí solo.
3.  **Redundancia con Propósito:** Aprendimos que variables como `loan_amnt` e `installment`, aunque correlacionadas al 95%, deben coexistir en el modelo. La primera mide la severidad de la pérdida potencial para el banco, mientras que la segunda mide la presión mensual sobre el bolsillo del cliente.
4.  **Desafío del Desbalance (78/22):** La población está sesgada hacia los buenos pagadores. Esto nos enseña que un modelo "perezoso" podría predecir que todos son buenos y tener un 78% de precisión, pero sería inútil para el negocio. El reto del modelado será maximizar el *Recall* (detectar a los malos) sin destruir la rentabilidad.
5.  **Variables Contraintuitivas:** Descubrimos que la antigüedad laboral (`emp_length`) tiene poco poder discriminatorio. Esto sugiere que, en el mercado actual, tener muchos años en un empleo no garantiza responsabilidad financiera, lo cual rompe un paradigma tradicional del análisis de crédito.