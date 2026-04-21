<div style="position: sticky; top: 0; background-color: white; padding: 10px 0; border-bottom: 1px solid #ddd; z-index: 999; text-align: center; width: 100%;">
  <a href="index.html" style="text-decoration: none;">🏠 <b>Inicio</b></a> | 
  <a href="exploracion.html" style="text-decoration: none;">🔍 <b>Exploración</b></a> |
  <a href="modelos.html" style="text-decoration: none;">🧠 <b>Modelos</b></a> | 
  <a href="casodeuso.html" style="text-decoration: none;">🏦 <b>Caso de Uso</b></a> | 
  <a href="sitio-web.html" style="text-decoration: none;">🌐 <b>Sitio Web</b></a>
</div>

<br>

# Exploración y Análisis Descriptivo

## 1. Análisis del Diccionario e Inspección de Datos

El conjunto de datos original [*Credit Risk Dataset*](https://www.kaggle.com/datasets/ranadeep/credit-risk-dataset/data) cuenta con **74 variables** que abarcan información demográfica, financiera y del historial crediticio de los solicitantes. 

### 1.1 Transformación de la Variable Objetivo (`loan_status`)
Siguiendo las mejores prácticas de la industria, transformamos la variable multiclase original en una **variable binaria (Target)**:
*   **0 (Buen pagador):** Incluye "Fully Paid" y "Does not meet the credit policy. Status:Fully Paid".
*   **1 (Mal pagador):** Incluye "Charged off", "Late (31-120 days)", "Default" y "Does not meet the credit policy. Status:Charged Off".
*   **Excluidos:** Registros "Current", "In Grace Period", etc., por su incertidumbre temporal.

### 1.2 Selección y Limpieza
Inicialmente, se eliminaron columnas con más del 50% de valores faltantes. Este umbral se estableció porque:

*   Cuando una variable tiene más del 50% de datos faltantes, intentar "completar" (imputar) esos valores con la media, mediana o moda introduce un sesgo masivo. Estaríamos inventando la mitad de la información, lo que puede llevar al modelo a encontrar patrones inexistentes (overfitting al ruido).
*   Si a la mayoría de los individuos no se les registró esa variable, es probable que no sea un requisito estándar para todos los perfiles crediticios o que el campo sea opcional.

Además, tras inspeccionar el diccionario, se identificó que no todas las variables son aptas para el entrenamiento. En la Tabla 1 se presentan los criterios de selección y limpieza aplicados:

<div align="center" markdown="1">

### Tabla 1. Variables eliminadas y justificación

| Categoría | Columnas (Variables) | Justificación Técnica de Eliminación |
| :--- | :--- | :--- |
| **Identificadores y Ruido** | `id`, `member_id`, `url`, `emp_title`, `desc`, `title` | Son datos únicos o de texto libre. No tienen valor estadístico y requerirían NLP complejo. El campo `purpose` ya resume la intención del crédito. |
| **Alta Cardinalidad / Sesgo** | `zip_code`, `addr_state` | El código postal y el estado tienen demasiadas categorías. Pueden causar **overfitting** (que el modelo aprenda que un estado es "bueno" solo por azar). |
| **Fuga de Datos (Leakage) - Pagos** | `total_pymnt`, `total_pymnt_inv`, `total_rec_prncp`, `total_rec_int`, `total_rec_late_fee`, `out_prncp`, `out_prncp_inv` | **Crítico:** Representan dinero ya pagado o saldos pendientes. Esta información solo existe *después* de que el crédito se otorgó. Si se incluyen, el modelo "adivina" el futuro en lugar de predecir el riesgo. |
| **Fuga de Datos (Leakage) - Incumplimiento** | `recoveries`, `collection_recovery_fee` | Estas variables solo se activan cuando el cliente ya falló. Incluirlas es "hacer trampa", ya que el modelo sabría la respuesta antes de procesar el perfil. |
| **Fuga de Datos (Leakage) - Fechas** | `issue_d`, `last_pymnt_d`, `last_pymnt_amnt`, `last_credit_pull_d`, `next_pymnt_d`, `earliest_cr_line` | Fechas de eventos que ocurren durante o después del préstamo. Pueden sesgar el modelo hacia periodos económicos específicos (inflación, crisis pasadas) que no se repetirán igual. |
| **Redundancia Matemática** | `installment`, `funded_amnt`, `funded_amnt_inv` | La cuota (`installment`) es una combinación de monto, tasa y plazo. Usamos `loan_amnt` y `int_rate` por separado para evitar que el modelo se vuelva perezoso con una fórmula fija. |
| **Sesgo Institucional (Proxy)** | `grade`, `sub_grade` | Son calificaciones que el banco ya asignó. Si las dejamos, el modelo solo aprenderá a "copiar" el criterio del analista humano en lugar de encontrar nuevos patrones en los datos crudos. |
| **Baja Variabilidad / Rareza** | `policy_code`, `collections_12_mths_ex_med`, `acc_now_delinq`, `tot_coll_amt` | Tienen casi el mismo valor para todos los usuarios (0 o 1). No ayudan a la Red Neuronal a distinguir entre un buen y un mal pagador. |
| **Complejidad innecesaria** | `tot_cur_bal`, `total_rev_hi_lim`, `revol_bal` | Para una aplicación web sencilla, estas variables añaden dificultad al usuario para responder y pueden estar correlacionadas con el nivel de ingresos y el DTI. |

</div>

Adicionalmente, no incluimos **`loan_status`**, ya que es la fuente de nuestra variable `target`. Por lo que si la dejamos dentro de las variables predictoras, el modelo tendrá una precisión del 100% de forma artificial, porque la respuesta está dentro de la pregunta.

### 1.3 Transformación de Variables Categóricas

Para codificar las variables categóricas, primero validamos si una variable es **ordinal** (tiene un orden lógico) o **nominal** (es solo una etiqueta). Esto es importante porque el tipo de codificación que se le aplicará a cada variable dependerá de esta clasificación. A continuación se muestra una tabla con la clasificación de cada variable categórica presente en el Dataframe:

<div align="center" markdown="1">

### Tabla 2. Clasificación de variables categóricas y tratamiento propuesto

| Variable | Tipo de Dato Original | Tratamiento Propuesto | Justificación |
| :--- | :--- | :--- | :--- |
| **`term`** | String (" 36 months") | **Numérico (36, 60)** | El plazo es una magnitud física. Un plazo más largo suele implicar mayor riesgo de impago por exposición al tiempo. |
| **`emp_length`** | String ("10+ years") | **Ordinal (0 a 10)** | Representa estabilidad laboral. "10+" es el valor máximo y "< 1 year" el mínimo. Se mapea a una escala lineal. |
| **`verification_status`** | String | **Nominal (One-Hot)** | Aunque hay niveles (Verified vs Not), no hay una escala numérica lineal clara de riesgo entre ellos. |
| **`home_ownership`** | String | **Nominal (One-Hot)** | No hay un orden natural. Por ejemplo, "RENT" no es "mejor" que "MORTGAGE" en términos de riesgo. |
| **`purpose`** / **`addr_state`** | String | **Nominal (One-Hot)** | Son categorías. No hay un orden lógico entre ellas. |
| **`pymnt_plan`** / **`initial_list_status`** / **`application_type`** | String | **Nominal (One-Hot)** | Son banderas (flags) binarias o de estado. |

</div>

Con base en esta clasificación, se aplicó **One-Hot Encoding** a las variables nominales y una transformación lineal a las variables ordinales. Esto permitió convertir todas las variables en un formato numérico adecuado para el entrenamiento de la Red Neuronal, sin introducir sesgos indebidos ni perder información relevante.

## 2. Análisis Descriptivo

### 2.1 Distribución de la Variable Objetivo

Uno de los hallazgos más críticos del análisis descriptivo es la distribución de nuestra variable objetivo.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig1_distribucion_target.png" width="700" />
    <p><em>Figura 1. Distribución de la variable objetivo</em></p>
</div>

De la Figura 1 se observa que hay aproximadamente **3.6 veces más buenos pagadores** que malos pagadores. Este desbalance requiere estrategias específicas de evaluación (como F1-Score o AUC-ROC) en lugar de la precisión simple (Accuracy).*

### 2.2 Análisis de Variables Numéricas vs Target

A continuación, se analiza la distribución de las variables clave mediante gráficos de violín, los cuales superan las limitaciones del boxplot tradicional al mostrar la densidad de probabilidad de los datos. Esta visualización es fundamental porque permite identificar no solo los cuartiles, sino también las "panzas" o concentraciones donde se agrupan la mayoría de los usuarios. Variables que muestren formas claramente desplazadas o diferentes entre ambos grupos, son los mejores predictores para nuestra Red Neuronal, mientras que aquellas con siluetas idénticas sugieren una baja capacidad de discriminación.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig2_numericas_vs_target.png" width="700" />
    <p><em>Figura 2. Distribución de variables numéricas por tipo de pagador</em></p>
</div>

Al analizar la Figura 2, se pueden sacar las siguientes conclusiones sobre el poder de separación de cada variable:

* **int_rate (Tasa de Interés):** La separación entre los "violines" es la más pronunciada del conjunto. La densidad de los malos pagadores se concentra claramente por encima del 15%, mientras que los buenos pagadores tienen su mayor volumen cerca del 12%. Es una señal limpia y potente: el riesgo percibido por el mercado (reflejado en la tasa) es un predictor excelente del riesgo real.

* **dti (Relación Deuda/Ingreso):** Muestra una separación visualmente significativa. El violín de los malos pagadores es notablemente más "ancho" en la parte superior (entre 20% y 30%) en comparación con los buenos pagadores. Esto valida que la carga financiera es un motor de incumplimiento, aunque menos drástico que la tasa de interés.

* **annual_inc (Ingreso Anual):** Gracias al zoom al 99%, ahora vemos que aunque los buenos pagadores tienen una "panza" más prominente en rangos de ingresos altos, la superposición de los cuerpos de ambos violines es masiva. Esto confirma que **el ingreso por sí solo no garantiza el pago**, lo que lo hace una variable de soporte, pero no de decisión primaria.

* **revol_util (Utilización Revolvente):** Los malos pagadores tienen una distribución mucho más uniforme hacia arriba, mientras que los buenos pagadores están "pesados" en la base (uso bajo). Es una variable que aporta un matiz importante sobre el comportamiento de consumo.

* **inq_last_6mths (Consultas recientes):** El violín de los "Buenos pagadores" es extremadamente delgado en la parte superior, concentrándose casi totalmente en 0 consultas. El de los "Malos pagadores" tiene mucha más densidad en 1 y 2 consultas. **Es una señal de alerta temprana excelente.**

Por otro lado, según la evidencia visual de estos gráficos, estas variables aportan poco o generan ruido:

1.  **open_acc (Cuentas abiertas) y total_acc (Total de cuentas):** Los violines son casi **gemelos idénticos**. Las medias ($\mu: 10.9$ vs $11.2$) y las formas de distribución se solapan casi por completo. Podrían descartarse. Tener muchas o pocas cuentas no parece distinguir en absoluto si alguien pagará o no. Mantener ambas solo añade complejidad innecesaria al modelo (colinealidad).

2.  **pub_rec (Registros públicos negativos):** La distribución es casi una línea plana en cero para ambos grupos. Aunque un registro público es malo, hay tan poca variabilidad en los datos (la inmensa mayoría tiene 0) que la Red Neuronal tendrá dificultades para extraer un patrón generalizable.

3.  **delinq_2yrs (Moras en 2 años):** Similar a `pub_rec`, el solapamiento es casi total. La diferencia entre una media de 0.2 y 0.3 es estadísticamente despreciable para un modelo de Deep Learning en este volumen de datos. Podría descartarse sin afectar significativamente la precisión del modelo.

### 2.3 Análisis de Variables Categóricas vs Target

De la misma manera analizaremos las variables categóricas vs el target, comparando cómo se distribuyen las variables categóricas más importantes entre buenos y malos pagadores usando gráficos de barras. De igual manera, si una variable tiene distribuciones diferentes entre las dos clases, es una buena señal de que será útil para el modelo.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig3_categoricas_vs_target.png" width="700" />
    <p><em>Figura 3. Tasa de incumplimiento por variable categórica</em></p>
</div>

Al analizar la Figura 3, se observa que la variable **term** es el principal diferenciador: los créditos a **60 meses** presentan una tasa de incumplimiento cercana al 34%, casi duplicando a los de **36 meses** (18%), lo que confirma que el plazo largo es un factor crítico. Por su parte, la variable **purpose** muestra una segmentación muy clara; los préstamos para **small_business** son los más riesgosos (32%), mientras que **car** y **wedding** se sitúan por debajo del 15%, demostrando ser los destinos de fondos más seguros.

Respecto a **verification_status**, se observa nuevamente un comportamiento **contraintuitivo**, donde los perfiles **Verified** tienen mayor mora que los **Not Verified** (24% vs 17%), sugiriendo que el proceso de verificación se aplica con mayor rigor a perfiles que ya se perciben como riesgosos. En **home_ownership**, se valida que la estabilidad patrimonial influye: quienes viven en **RENT** superan el promedio de mora, mientras que aquellos con **MORTGAGE** presentan un mejor comportamiento de pago.

La variable **initial_list_status** muestra una diferencia sutil pero marcada, donde el estado **"w"** es más propenso al impago que el **"f"**. Finalmente, **emp_length** destaca por ser la variable con **nulo poder discriminativo**, ya que la tasa de mora se mantiene estancada alrededor de la media (21.9%) sin importar si el cliente tiene menos de un año o más de diez de antigüedad.

De lo anterior, se podría considerar elinar estas variables por las siguientes razones:

*   **pymnt_plan:** La categoría **"y"** tiene una tasa de mora altísima (66%), pero solo cuenta con **6 observaciones (n=6)** frente a más de 268,000 de la otra categoría. No es estadísticamente significativa.
*   **application_type:** La categoría **JOINT** tiene apenas **3 observaciones (n=3)**. Su alto porcentaje de mora es anecdótico y no sirve para generalizar un modelo.
*   En **home_ownership**, las categorías **ANY** (n=1) y **NONE** (n=48) deben eliminarse o agruparse en "OTHER", ya que su volumen es insuficiente para el análisis.
*   **emp_length:** Aunque tiene muchos datos, no aporta valor para separar "buenos" de "malos" pagadores. Podría eliminarse para simplificar el modelo sin perder precisión.

### 2.4 Correlación entre las Variables y el Target

La correlación de Pearson nos permite cuantificar la relación lineal entre cada variable numérica y el target. Esto es crucial para identificar cuáles variables tienen mayor poder predictivo y deben ser priorizadas en la ingeniería de características y en la interpretación del modelo.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig4a_correlaciones_target.png" width="700" />
    <p><em>Figura 4. Correlación de variables numéricas con el target</em></p>
</div>

De la figura 4 podemos notar que la **tasa de interés (`int_rate`)** lidera con 0.255, confirmando que el costo del crédito es la variable que tiene mayor impacto en el riesgo. Le siguen el **DTI (0.134)** y la **utilización del crédito (`revol_util`, 0.100)**; este bloque valida que el estrés financiero por sobreendeudamiento es el principal disparador del impago.

Por otro lado, el **ingreso anual (`annual_inc`)** y el **total de cuentas (`total_acc`)** muestran correlaciones negativas. Esto sugiere que la estabilidad financiera y una trayectoria amplia en el sistema actúan como amortiguadores, reduciendo la probabilidad de mora.

### 2.5 Análisis de Correlación entre Variables

El análisis de correlación entre las variables numéricas nos ayuda a identificar posibles redundancias o multicolinealidades que podrían afectar el rendimiento del modelo. Por ejemplo, si dos variables están altamente correlacionadas entre sí, podrían estar aportando información similar al modelo, lo que podría llevar a un sobreajuste.

Esperaríamos encontrar baja correlación entre las variables, debido al proceso de selección y limpieza que se realizó, pero es importante validar esto con un mapa de calor de correlaciones.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig4b_heatmap_correlaciones.png" width="700" />
    <p><em>Figura 5. Heatmap de correlaciones entre variables relevantes</em></p>
</div>

El mapa de calor de la Figura 5 revela que la mayoría de las variables tienen correlaciones bajas entre sí (cercanas a 0). Esto es un resultado excelente, ya que indica que no hay **multicolinealidad** grave. 

* **`revol_util` vs `int_rate` (0.33):** Es la correlación más alta de la matriz. Tiene sentido financiero: los clientes que usan al límite sus tarjetas suelen recibir tasas de interés más altas. Sin embargo, un valor de 0.33 es lo suficientemente bajo para confirmar que ambas variables deben coexistir; la tasa mide el costo del mercado y la utilización mide el comportamiento de consumo.

* **`annual_inc` vs `loan_amnt` (0.34):** Existe una relación lógica donde personas con mayores ingresos solicitan montos más altos. Al ser una correlación moderada, la red puede distinguir perfectamente entre un "crédito grande para alguien rico" (bajo riesgo) y un "crédito grande para alguien de ingresos medios" (alto riesgo).

### 2.6 Eliminación de Variables tras el Análisis Descriptivo y de Correlación

Tras el análisis descriptivo y de correlación, se definieron las variables y categorías específicas que serán eliminadas para optimizar el entrenamiento de la Red Neuronal, evitando el ruido estadístico y problemas de multicolinealidad.

<div align="center" markdown="1">

### Tabla 3. Variables eliminadas tras el análisis descriptivo y justificación técnica

| Categoría | Variables / Casos Específicos | Justificación Técnica Post-EDA |
| :--- | :--- | :--- |
| **Baja Variabilidad** | `pub_rec`, `delinq_2yrs` | La inmensa mayoría de los registros están en cero. La diferencia en las medias es estadísticamente despreciable para separar grupos con claridad. |
| **Redundancia / Colinealidad** | `open_acc`, `total_acc` | Sus distribuciones (violines) son prácticamente idénticas. Mantener ambas aporta información redundante que no mejora la discriminación del riesgo. |
| **Insignificancia Estadística** | `pymnt_plan`, `application_type` | Presentan un desbalance extremo (n < 10 en categorías minoritarias). El modelo podría sobreajustarse a casos anecdóticos en lugar de aprender patrones reales. |
| **Categorías con n bajo** | `home_ownership` (**ANY**, **NONE**) | Categorías con volumen insuficiente (**ANY** n=1, **NONE** n=48) que no permiten una generalización confiable. Se eliminan para limpiar el ruido en las variables categóricas. |
| **Nulo Poder Predictivo** | `emp_length` | El análisis de tasas de incumplimiento demostró que la mora es constante (~21.9%) independientemente de la antigüedad laboral. No ayuda a distinguir perfiles. |
| **Ruido Administrativo** | `initial_list_status` | Aunque presenta una diferencia sutil, responde más a procesos internos de asignación de fondos que a una característica de riesgo del solicitante. |

</div>

## 3. Planteamiento de Hipótesis

Tras el análisis descriptivo, formalizamos las siguientes hipótesis de riesgo basadas en las diferencias observadas entre buenos y malos pagadores. Estas hipótesis guiarán la interpretación de los resultados del modelo y la elaboración de la scorecard final:

<div align="center" markdown="1">

### Tabla 4. Hipótesis de riesgo basadas en el análisis descriptivo

| Variable | Hallazgo | Hipótesis de Riesgo |
| :--- | :--- | :--- |
| `int_rate` | Malos $\mu: 16.0\%$ vs Buenos $\mu: 13.3\%$. Correlación: **0.255**. | El mercado ya identifica el riesgo; tasas altas asfixian al deudor, elevando la probabilidad de impago por carga financiera excesiva. |
| `dti` | Malos presentan mayor "ancho" entre 20% y 30%. Correlación: **0.134**. | Un DTI elevado indica poco margen de maniobra ante imprevistos, haciendo que el nuevo crédito sea difícil de sostener frente al ingreso. |
| `inq_last_6mths`|Buenos pagadores se concentran casi totalmente en 0 consultas. Correlación: **0.053**. | Consultas recientes indican una necesidad urgente de liquidez, actuando como una señal de alerta de inestabilidad financiera. |
| `revol_util` | Malos usan sus líneas de crédito de forma agresiva y constante. | Un uso alto de líneas rotativas sugiere que el cliente vive al límite de su capacidad y usa el crédito para gastos corrientes. |
| `annual_inc` | Buenos pagadores tienen una densidad más prominente en rangos altos ($\mu: \$74k$). | Ingresos altos actúan como colchón. A menor ingreso, mayor vulnerabilidad ante la volatilidad económica. |
| `term` | 60 meses duplica la tasa de mora vs 36 meses (34% vs 18%). | Créditos a largo plazo están sujetos a más eventos de vida (despido, enfermedad), aumentando el riesgo de incumplimiento. |

</div>

## 4. Conclusiones y Aprendizajes del EDA

El proceso de exploración no solo sirvió para limpiar datos, sino para generar aprendizajes estratégicos sobre el fenómeno del riesgo para tener en cuenta durante el modelado:

1.  El análisis confirma que `int_rate` es el predictor más fuerte. Existe un "círculo vicioso" donde el costo de la deuda, impuesto por el riesgo percibido, termina siendo el principal disparador del incumplimiento real.
2.  El perfil del "Mal Pagador" no se define solo por ganar menos, sino por estar **sobreendeudado**. La combinación de un DTI elevado y una utilización de tarjetas (`revol_util`) alta es una señal más potente que el nivel de ingresos por sí solo.
3.  Aprendimos que variables tradicionales como la antigüedad laboral (`emp_length`) y el conteo de cuentas (`open_acc`, `total_acc`) tienen un poder discriminatorio nulo, presentando distribuciones casi idénticas entre grupos. Su eliminación simplifica la arquitectura de la red y mejora la experiencia de usuario en la App sin sacrificar precisión.
4.  El mapa de calor confirmó que el set de variables final es robusto y carece de multicolinealidad grave. Cada variable seleccionada aporta una dimensión única (capacidad, comportamiento o costo), facilitando la convergencia del modelo de Deep Learning.
5.  Con una población sesgada hacia los buenos pagadores, el reto no será la precisión global, sino la capacidad del modelo para detectar a los "malos" (*Recall*) sin generar un exceso de falsos rechazos que afecten la colocación de créditos.

## 5. Referencias

**Gopi, S. (2020, 20 de septiembre).** *How to Prepare Data for Credit Risk Modeling*. Towards Data Science. https://towardsdatascience.com/how-to-prepare-data-for-credit-risk-modeling-5523641882f2/

**Listendata. (2019, agosto).** *A Complete Guide to Credit Risk Modelling*. ListenData. https://www.listendata.com/2019/08/credit-risk-modelling.html

**R.G. (2021).** *Credit Risk Dataset* [Conjunto de datos]. Kaggle. https://www.kaggle.com/datasets/ranadeep/credit-risk-dataset/data
