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

# Modelado y Aprendizajes del Proceso

## 1. Modelo de Referencia: Regresión Logística

Antes de sumergirnos en la complejidad de las redes neuronales, implementamos una **Regresión Logística** como modelo de referencia (baseline). Este modelo es ampliamente utilizado en la industria financiera para tareas de scoring crediticio debido a su simplicidad, eficiencia y alta interpretabilidad.

### Configuración Técnica

El entrenamiento se configuró con parámetros específicos para garantizar la estabilidad y el manejo del desbalance:

*   **Solver `lbfgs` (Limited-memory Broyden–Fletcher–Goldfarb–Shanno):** Seleccionamos este optimizador porque es altamente eficiente para problemas de clasificación multivariable en conjuntos de datos grandes. Pertenece a la familia de métodos de "cuasi-Newton" y utiliza una cantidad limitada de memoria computacional para aproximar la curvatura de la función de pérdida, lo que permite una convergencia más rápida que el descenso de gradiente estándar.
*   **`max_iter=1000`:** Dado que el dataset tiene 94 variables tras la codificación, aumentamos el límite de iteraciones para asegurar que el algoritmo encuentre el mínimo global de la función de costo y no se detenga prematuramente (evitando errores de "non-convergence").
*   **`class_weight='balanced'`:** Esta es quizás la decisión más crítica. El dataset original es desbalanceado (78% buenos vs 22% malos). Sin este parámetro, el modelo tendería a predecir que "todos son buenos" para maximizar el acierto simple. Al balancear, el algoritmo aplica una **penalización mayor** a los errores cometidos sobre la clase minoritaria (los malos pagadores), forzando al modelo a aprender sus características distintivas.

### Resultados

<div align="center" markdown="1">

*Tabla 1. Métricas de clasificación del modelo base (Regresión Logística)*

| Métrica | Clase / Promedio | Precisión | Recall | F1-Score | Soporte |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Clase 0** | Buen pagador | 0.87 | 0.66 | 0.75 | 41,942 |
| **Clase 1** | Mal pagador | 0.35 | 0.64 | 0.45 | 11,764 |
| **Exactitud** | *Accuracy* | | | **0.65** | 53,706 |
| **Promedio Simple** | *Macro Avg* | 0.61 | 0.65 | 0.60 | 53,706 |
| **Promedio Ponderado** | *Weighted Avg* | 0.75 | 0.65 | 0.68 | 53,706 |

</div>

<br>    

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig5_modelo_base.png" width="700" />
    <p><em>Figura 1.  Evaluación del modelo base (Regresión Logística)</em></p>
</div>

De acuerdo con las métricas obtenidas presentadas en la Tabla 1 y la Figura 1, podemos extraer las siguientes conclusiones sobre el rendimiento del modelo de regresión logística:

*   A primera vista, un acierto del 65% parece bajo. Sin embargo, debemos considerar que si el modelo fuera "perezoso" y predijera a todos los clientes como **buenos pagadores**, obtendría un **78.1% de accuracy** de forma artificial, pero fallaría en detectar al 100% de los morosos. Por lo tanto, nuestro 65% es más valioso porque realmente está discriminando entre grupos.
*   El **AUC-ROC de 70.43%** nos indica que existe un **70.43% de probabilidad** de que el modelo asigne un score de riesgo más alto a un mal pagador elegido al azar que a un buen pagador. Es una medida de la capacidad de separación del modelo, independientemente del umbral usado.
*   El modelo logra detectar satisfactoriamente al **64% de los malos pagadores reales (Recall)**. No obstante, su precisión es baja (35%), lo que significa que muchas personas marcadas como "riesgosas" en realidad terminarían pagando bien. En el contexto bancario, esto indicaría rechazar a algunos buenos pagadores por precaución antes que aprobar a un mal pagador que genere una pérdida total de capital.

## 2. Redes Neuronales Artificiales (ANN)

Tras establecer el modelo base, procedemos a diseñar una Red Neuronal para intentar capturar interacciones no lineales entre las variables financieras que la regresión logística podría haber omitido.

### Diseño de la Red Neuronal Base

Para compensar el desbalance de clases, asignamos un peso distinto a cada tipo de cliente (buenos y malos pagadores). Para determinar estos pesos, se utilizó una técnica de **balanceo de costos** basada en la frecuencia de las clases. El objetivo es que el modelo "sienta" la misma importancia por ambas clases, a pesar de que una sea mucho más frecuente que la otra.

La lógica matemática se basa en la siguiente fórmula estándar (utilizada por librerías como Scikit-learn):


<div align="center" markdown="1">

$$Peso = \frac{N_{total}}{k \cdot n_{clase}}$$

*Ecuación 1. Cálculo de pesos para clases desbalanceadas*
</div>

Donde:
*   **$N_{total}$**: Es el número total de registros en el set de entrenamiento (**214,824**).
*   **$k$**: Es el número de clases (**2**: Bueno y Malo).
*   **$n_{clase}$**: Es la cantidad de registros que hay de esa clase específica.

En nuestro caso, la distribución era:
*   **Buenos pagadores (0):** 167,769 registros.
*   **Malos pagadores (1):** 47,055 registros.

**1. Peso para el Buen Pagador (Clase 0):**

<div align="center" markdown="1">

$$W_0 = \frac{214,824}{2 \cdot 167,769} \approx \mathbf{0.640}$$

*Ecuación 2. Peso para la clase de buenos pagadores*

</div>
Como hay muchos buenos pagadores, su peso es menor a 1. El modelo "suaviza" su importancia para no sesgarse hacia ellos.

**2. Peso para el Mal Pagador (Clase 1):**

<div align="center" markdown="1">

$$W_1 = \frac{214,824}{2 \cdot 47,055} \approx \mathbf{2.283}$$

*Ecuación 3. Peso para la clase de malos pagadores*
</div>

Como hay pocos malos pagadores, su peso es mayor a 1 (específicamente, **3.5 veces más pesado** que el de un buen pagador). Esto le dice a la Red Neuronal: *"Cometer un error con un mal pagador es 3.5 veces más costoso que equivocarse con uno bueno"*.

Si no asignáramos estos pesos, la Red Neuronal intentaría maximizar el **Accuracy** de la forma más fácil: prediciendo que **todos** son buenos pagadores. Si lo hiciera, tendría un 78% de precisión sin aprender nada. 

Al aplicar estos pesos, la función de pérdida (**Binary Crossentropy**) se vuelve "más pesada" cuando el modelo falla en predecir a un mal pagador, obligando a la red a ajustar sus neuronas para detectar esos patrones minoritarios, lo que eleva el **Recall** (la capacidad de encontrar a los morosos).

**Arquitectura de la Red Neuronal:**

Diseñamos una arquitectura de modelo **secuencial** compuesta por una capa de entrada de **128 neuronas**, seguida de dos capas ocultas de **64 y 32 neuronas** respectivamente. Todas las capas son **densas**, lo que significa que cada neurona está conectada con todas las de la capa siguiente para maximizar el flujo de información. Para las capas internas, utilizamos la función de activación **ReLU**, que ayuda a mitigar el problema de desvanecimiento del gradiente, mientras que en la capa de salida empleamos una función **Sigmoide**. Esta última nos entrega un valor entre 0 y 1, interpretado como la probabilidad de incumplimiento, permitiendo graduar la certeza del modelo sobre cada cliente.

Configuramos una **tasa de aprendizaje de 0.001**, un valor equilibrado para asegurar que el modelo converja de forma estable sin colapsar la memoria ni quedar atrapado prematuramente en un mínimo local. Para combatir el **sobreentrenamiento (overfitting)**, implementamos un **ratio de dropout del 30%** en cada capa, "apagando" neuronas aleatoriamente durante el entrenamiento para evitar que el modelo simplemente memorice los datos. Adicionalmente, aplicamos **Batch Normalization** para normalizar las activaciones entre capas, lo que estabiliza el proceso y acelera la convergencia.

### Entrenamiento y Resultados Iniciales

Compilamos la red utilizando el optimizador **Adam** y la función de pérdida **binary_crossentropy**, ideal para nuestro objetivo binario. Implementamos dos **Callbacks** estratégicos:
*   **EarlyStopping:** Detiene el entrenamiento si el AUC de validación no mejora tras 10 épocas, restaurando los mejores pesos encontrados.
*   **ReduceLROnPlateau:** Reduce la tasa de aprendizaje a la mitad si el rendimiento se estanca durante 5 épocas, permitiendo un ajuste más fino al final del proceso.

El entrenamiento se realizó durante un máximo de **100 épocas** con un tamaño de **batch de 512 registros**, utilizando una partición del 80% para aprendizaje y 20% para validación interna.

<div align="center" markdown="1">

*Tabla 2. Métricas de clasificación de la Red Neuronal Base*

| Métrica | Clase / Promedio | Precisión | Recall | F1-Score | Soporte |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Clase 0** | Buen pagador | 0.87 | 0.64 | 0.74 | 41,942 |
| **Clase 1** | Mal pagador | 0.34 | 0.67 | 0.45 | 11,764 |
| **Exactitud** | *Accuracy* | | | **0.65** | 53,706 |
| **Promedio Simple** | *Macro Avg* | 0.61 | 0.65 | 0.60 | 53,706 |
| **Promedio Ponderado** | *Weighted Avg* | 0.76 | 0.65 | 0.68 | 53,706 |

</div>

<br>

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig6_red_neuronal.png" width="700" />
    <p><em>Figura 2.  Evaluación de la Red Neuronal Base</em></p>
</div>

De la Tabla 2 y la Figura 2, podemos destacar los siguientes aprendizajes sobre el rendimiento de la Red Neuronal Base:

*   **AUC-ROC (0.7133):** Logramos una mejora ligera frente a la regresión logística (0.7043), demostrando una mayor capacidad de discriminación entre ambas clases.
*   **Recall del Mal Pagador (67%):** El modelo es capaz de identificar a 2 de cada 3 clientes morosos. Este es un resultado positivo para el negocio, ya que priorizamos la captura de riesgo.
*   **Precisión del Mal Pagador (34%):** Esta es la métrica más baja; indica que, de cada 100 clientes que el modelo marca como "riesgosos", solo 34 realmente terminan incumpliendo. Existe un costo de oportunidad al clasificar a "buenos pagadores" como "malos" (Falsos Positivos).
*   **F1-Score (0.45):** El equilibrio entre precisión y sensibilidad se mantiene en niveles similares al baseline, lo que confirma que el modelo base de la red todavía tiene margen de mejora.
*   **Accuracy General (65%):** Aunque es menor que la proporción de la clase mayoritaria (78%), este valor es esperado dado que el modelo está penalizando fuertemente los errores en la clase minoritaria para no ignorar el riesgo.

## 3. Red Neuronal Optimizada: SMOTE y Búsqueda de Hiperparámetros

Para reducir los sesgos que el peso de clase (class weights) pudiera haber introducido en la red inicial, implementamos la técnica de oversampling **SMOTE (Synthetic Minority Over-sampling Technique)**. A diferencia del sobremuestreo simple, SMOTE no duplica registros, sino que crea nuevos ejemplos sintéticos de "malos pagadores" interpolando entre los ya existentes.

Este proceso transformó nuestro conjunto de datos de la siguiente manera:
*   **Antes de SMOTE:** 167,769 buenos pagadores vs. 47,055 malos pagadores (Fuerte desbalance).
*   **Después de SMOTE:** Igualamos ambas clases a **167,769 registros cada una**, alcanzando un total de **335,538 registros** de entrenamiento.

**¿Por qué hicimos esto?** Al proporcionar a la red una cantidad equitativa de ejemplos de ambas clases, el algoritmo puede aprender de forma más robusta las fronteras de decisión, disminuyendo la tendencia natural del modelo a favorecer a la clase mayoritaria.

Con un dataset balanceado, el siguiente reto fue encontrar la estructura de red ideal. No todas las redes profundas son mejores; a veces, una red más "ancha" captura mejor la información tabular. Probamos **30 combinaciones** distintas evaluando:
1.  **Arquitecturas:** Desde redes "Muy profundas" (5 capas) hasta redes "Anchas" (2 capas con muchas neuronas).
2.  **Dropout:** Probamos valores de 0.3, 0.4 y 0.5 para encontrar el punto exacto donde el modelo deja de memorizar (overfitting) y comienza a generalizar.
3.  **Learning Rate (Tasa de aprendizaje):** Evaluamos 0.001 y 0.0005 para ajustar la velocidad de convergencia.

**Hallazgos de la Optimización:**
La combinación ganadora fue la arquitectura **"Muy ancha"**, compuesta por **dos capas ocultas de 512 y 256 neuronas**. Se determinó que un **dropout del 0.5** era necesario para regularizar una red tan ancha, junto con una **tasa de aprendizaje de 0.001**. Esta configuración alcanzó un **AUC-ROC de 0.7052** y el mejor equilibrio de F1-Score durante las pruebas de validación.

### Entrenamiento del Modelo Optimizado
Utilizando la mejor combinación encontrada, procedimos a entrenar el modelo definitivo ajustando los siguientes **Callbacks**:
*   **Paciencia de 15 épocas** en el EarlyStopping para permitir que la red explorara mejor el espacio de soluciones.
*   **Batch size de 1024**, duplicando el anterior para aprovechar la mayor cantidad de datos generados por SMOTE y estabilizar las estimaciones del gradiente.

### Ajuste del Umbral de Clasificación Óptimo
Por defecto, una red neuronal clasifica a un cliente como "mal pagador" si la probabilidad es mayor a 0.5 (50%). Sin embargo, en riesgo crediticio, este umbral no siempre es el más rentable para el negocio.

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig7a_umbral_optimo.png" width="700" />
    <p><em>Figura 3.  Métricas vs Umbral de clasificación</em></p>
</div>

Realizamos un barrido de umbrales desde el 10% hasta el 90% para encontrar el punto donde maximizamos el **F1-Score** (el equilibrio perfecto entre detectar morosos y no perder clientes buenos). Como se observa en la Figura 3, el **umbral óptimo se situó en 0.51**.

### Resultados de la Red Optimizada con SMOTE

<div align="center" markdown="1">

*Tabla 3. Métricas de clasificación de la Red Neuronal Optimizada con SMOTE*

| Métrica | Clase / Promedio | Precisión | Recall | F1-Score | Soporte |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Clase 0** | Buen pagador | 0.87 | 0.66 | 0.75 | 41,942 |
| **Clase 1** | Mal pagador | 0.35 | 0.65 | 0.46 | 11,764 |
| **Exactitud** | *Accuracy* | | | **0.66** | 53,706 |
| **Promedio Simple** | *Macro Avg* | 0.61 | 0.66 | 0.60 | 53,706 |
| **Promedio Ponderado** | *Weighted Avg* | 0.76 | 0.66 | 0.69 | 53,706 |

</div>

<br>

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig8_red_neuronal_smote.png" width="700" />
    <p><em>Figura 4.  Evaluación de la Red Neuronal optimizada con SMOTE</em></p>
</div>

Los resultados obtenidos con la red optimizada utilizando SMOTE, presentados en la Tabla 3 y la Figura 4, revelan los siguientes aprendizajes:

*   **AUC-ROC (0.7132):** Prácticamente idéntico al modelo base y apenas **0.0089** por encima de la Regresión Logística.
*   **Recall (65%) y Precisión (35%):** Las métricas de clasificación se mantienen estancadas en los mismos niveles que el baseline inicial.
*   **F1-Score (0.46):** No hubo una mejora significativa en el equilibrio entre sensibilidad y precisión.

## 4. Ensamble Híbrido

Dado que las mejoras obtenidas con las redes neuronales convencionales y SMOTE no fueron tan disruptivas como esperábamos (manteniéndose en un rango de AUC de 0.71), decidimos realizar un intento final con un enfoque distinto. Este nuevo experimento integró **ingeniería de características avanzada**, un **modelo de boosting** de última generación y un **ensamble híbrido** diseñado específicamente para maximizar la detección de morosos.

### Ingeniería de Características
Uno de los mayores retos en riesgo crediticio son las variables categóricas con muchas opciones (como el estado de residencia o el propósito del crédito). En lugar de usar One-Hot Encoding (que crea cientos de columnas vacías), aplicamos **Target Encoding con Suavizado Bayesiano**. 

Esta técnica transforma cada categoría en un número que representa la probabilidad promedio de impago para ese grupo, pero aplicando un "suavizado" para evitar que categorías con pocos registros engañen al modelo. Esto con la intención de que el algoritmo capturara patrones geográficos y de comportamiento mucho más precisos.

### Componentes del Ensamble
En lugar de confiar en un solo modelo, combinamos las fortalezas de dos algoritmos muy distintos:

1.  **LightGBM (Gradient Boosting):** Es extremadamente eficiente para encontrar reglas de decisión basadas en umbrales numéricos (por ejemplo: "Si el DTI > 18% y la tasa > 15%, el riesgo sube exponencialmente").
2.  **Red Neuronal de Nueva Generación:** Diseñamos una red más moderna utilizando la función de activación **Swish** (que permite un flujo de gradiente más suave que ReLU) y **Layer Normalization** para asegurar que el modelo sea robusto ante variaciones en los datos de entrada.

Combinamos ambos modelos en una proporción de **85% LightGBM y 15% Red Neuronal**. Esta mezcla permite que el modelo de boosting lleve el peso principal de la decisión, mientras que la red neuronal actúa como un "suavizador" que ayuda a generalizar mejor en casos atípicos.

Finalmente, realizamos una **optimización dirigida por el negocio**. En el sector financiero, dejar pasar a un mal pagador es mucho más costoso que rechazar a uno bueno por error. Por ello, buscamos el umbral de decisión exacto que nos garantizara un **Recall del 70%**. Encontramos que al ajustar nuestro umbral de clasificación a **0.20**, el modelo lograba detectar a 7 de cada 10 morosos.

### Resultados del Ensamble Optimizado

<div align="center" markdown="1">

*Tabla 4. Métricas de clasificación del Ensamble Híbrido Optimizado*

| Métrica | Clase / Promedio | Precisión | Recall | F1-Score | Soporte |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Clase 0** | Buen pagador | 0.88 | 0.62 | 0.73 | 41,942 |
| **Clase 1** | Mal pagador | 0.34 | **0.70** | 0.46 | 11,764 | 
| **Exactitud** | *Accuracy* | | | **0.64** | 53,706 |
| **Promedio Simple** | *Macro Avg* | 0.61 | 0.66 | 0.59 | 53,706 |
| **Promedio Ponderado** | *Weighted Avg* | 0.76 | 0.64 | 0.67 | 53,706 |

</div>

<br>

<div style="text-align: center;">
    <img src="https://raw.githubusercontent.com/jihernandezc/rnaab_riesgo_crediticio/refs/heads/master/output/figs/fig9_ensamble_optimizado.png" width="700" />
    <p><em>Figura 5. Evaluación del Ensamble Optimizado</em></p>
</div>

Los resultados del ensamble híbrido optimizado, presentados en la Tabla 4 y la Figura 5, revelan los siguientes aprendizajes clave:

*   **AUC-ROC Final (0.7204):** Es el valor más alto obtenido en todo el proyecto, superando finalmente la barrera del 0.71.
*   **Recall Final del Mal Pagador (70.11%):** Logramos cumplir con el objetivo de detectar a la gran mayoría de los clientes riesgosos.
*   **Precisión (34%):** Se mantiene estable, lo cual es un éxito considerando que aumentamos el Recall (normalmente, al subir el Recall, la precisión cae drásticamente).
*   **F1-Score (0.46):** Representa el mejor equilibrio logrado entre todas las técnicas probadas.

## 5. Aprendizajes del Proceso de Modelamiento

Tras experimentar con técnicas como el sobremuestreo sintético (**SMOTE**), optimizaciones de arquitectura y ensambles híbridos, el rendimiento del modelo se mantuvo en el rango de **0.70 a 0.72 de AUC-ROC**. Este comportamiento nos lleva a reflexionar sobre la naturaleza de los datos y la complejidad del problema:

* A pesar de que el ensamble híbrido logró el mejor desempeño (AUC: 0.7204), la mejora respecto a la Regresión Logística básica fue de apenas **1.6 puntos porcentuales**. Este fenómeno se explica mediante el ***"Flat-maximum effect"***. Según **Overstreet et al. (1992)**, en el scoring crediticio existe una robustez inherente donde modelos lineales simples suelen rendir casi a la par de modelos altamente complejos. Esto sugiere que las variables financieras tradicionales (DTI, tasa, ingresos) han entregado toda la "señal" posible, y el ruido restante no puede ser capturado solo con algoritmos más potentes.

* Desde una perspectiva de ingeniería, la implementación de arquitecturas con funciones de activación **Swish** y procesos de **Target Encoding Bayesiano** demandó un costo computacional y de mantenimiento significativamente mayor. No obstante, en el sector financiero, incluso una mejora marginal puede traducirse en ahorros millonarios. Aun así, como bien señalan **Lessmann et al. (2015)** en su estudio comparativo de algoritmos de scoring, las mejoras de las técnicas modernas sobre los modelos lineales suelen ser marginales y requieren un esfuerzo de optimización desproporcionado.

* Aunque un AUC de 0.72 pueda parecer modesto en otras áreas de la IA, en el análisis de riesgo crediticio se considera un desempeño **"Aceptable a Bueno"**. De acuerdo con la escala de **Hosmer et al. (2013)**, un valor por encima de 0.70 indica una capacidad de discriminación adecuada para la toma de decisiones institucionales.

El aprendizaje más valioso de este proceso es que hemos alcanzado el límite de rendimiento con el conjunto de información disponible. El modelo es **robusto y confiable** dentro de sus parámetros, pero los resultados confirman que la ventaja competitiva en el futuro no vendrá de una red neuronal más profunda, sino del enriquecimiento del dataset con fuentes de datos alternativas que capturen la señal que hoy se pierde entre el ruido.

## 6. Referencias

**Chawla, N. V., Bowyer, K. W., Hall, L. O., & Kegelmeyer, W. P. (2002).** SMOTE: Synthetic minority over-sampling technique. *Journal of Artificial Intelligence Research*, 16, 321–357. https://doi.org/10.1613/jair.953

**Hosmer, D. W., Lemeshow, S., & Sturdivant, R. X. (2013).** *Applied Logistic Regression* (3ra ed.). John Wiley & Sons.  https://github.com/Drxan/Study/blob/master/Books_Need2Read/David%20W.%20Hosmer%20-%20Applied%20Logistic%20Regression%20-%203rd%20Edition.pdf

**Lessmann, S., Baesens, B., Seow, H. V., & Thomas, L. C. (2015).** Benchmarking state-of-the-art classification algorithms for credit scoring: An update of research. *European Journal of Operational Research*, 247(1), 124–136. https://doi.org/10.1016/j.ejor.2015.05.030

**Overstreet, G. A., Jr., Bradley, E. L., Jr., & Kemp, R. S., Jr. (1992).** The flat-maximum effect and generic linear scoring models: A test. *Journal of the Operational Research Society*, 43(10), 971–981. https://doi.org/10.1057/jors.1992.140


