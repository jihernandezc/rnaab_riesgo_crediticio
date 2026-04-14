<p align="center">
  <img src="https://cdiac.manizales.unal.edu.co/imagenes/LogosMini/un.png" width="250">
  <br>
  <strong>Universidad Nacional de Colombia</strong><br>
  Facultad de Minas - Sede Medellín
</p>

**Curso:** Introducción a Redes Neuronales y Algoritmos Bioinspirados  
**Semestre:** 2026-01  

**Profesor:** Juan David Ospina  
**Monitor:** Andrés Mauricio Zapata  

**Integrantes:**
*   Tomás Acevedo Roldán
*   Santiago Cardona Franco
*   Jimena Hernández Castillo

# **Predicción de Incumplimiento del Crédito con Redes Neuronales**

## 1. Delimitación del Problema

El riesgo de crédito es, en esencia, la probabilidad de que un cliente no cumpla con sus obligaciones financieras. Para una entidad bancaria, no identificar a un "mal pagador" a tiempo se traduce en pérdidas directas de capital, mientras que rechazar a un "buen pagador" representa un costo de oportunidad significativo.

El propósito de este proyecto, es utilizar el [*Credit Risk Dataset* de Kaggle](https://www.kaggle.com/datasets/ranadeep/credit-risk-dataset/data) para construir un sistema que no solo prediga la probabilidad de que un individuo incumpla con sus obligaciones financieras, sino que esa predicción sea interpretable para el negocio. No buscamos una "caja negra", sino una herramienta accionable que asigne un puntaje (Scorecard) basado en el comportamiento histórico y características sociodemográficas.

## 2. Metodología del Proyecto

### Fase I: Exploración y Entendimiento del Dominio
Revisión exhaustiva del diccionario de variables proporcionado para comprender el significado de cada indicador financiero y validación inicial de la consistencia del dataset.

### Fase II: Ingeniería de Datos y Preprocesamiento
1.  **Limpieza y Filtrado:** Elimiar columnas que no aportan información relevante. Selección de registros basada en la lógica de negocio (filtrado de NA según el estatus del préstamo).
2.  **Codificación Target:** Transformación de la variable `loan_status` a binaria (0: Bueno, 1: Malo).
3.  **Tratamiento de Variables:** Manejo de valores faltantes, codificación de variables categóricas a valores numéricos.

### Fase III: Análisis Descriptivo y Elaboración de Hipótesis
Identificación de patrones, valores atípicos (outliers) y relaciones entre variables que nos permitan generar hipótesis sobre qué hace que un cliente sea riesgoso.

### Fase IV: Modelamiento Predictivo y Optimización
1.  **Modelo de Referencia (Baseline):** Implementación de un modelo de baja complejidad (Regresión Logística) para contrastar el valor agregado de la red neuronal.
2.  **Arquitectura de Redes Neuronales:** Diseño y optimización de una Red Neuronal medinate diferentes técnicas.
3.  **Evaluación Multicriterio:** Uso de métricas como AUC-ROC (capacidad de discriminación), Recall (captura de morosos) y F1-Score (equilibrio de precisión).

### Fase V: Construcción de la Scorecard
1.  **Traducción de Probabilidades:** Conversión de la salida sigmoide de la red neuronal en una escala de puntos (Score).
2.  **Análisis de Atributos:** Identificación de las variables con mayor impacto en el riesgo mediante técnicas de importancia de características (SHAP o pesos del modelo).

### Fase VI: Desarrollo del Sitio Web
1.  **Backend:** Integración del modelo entrenado en un entorno de ejecución.
2.  **Frontend:** Diseño de una interfaz intuitiva donde el usuario ingresa sus datos y recibe:
    * Su puntaje crediticio (Score).
    * Su posición relativa frente a la población general.
3.  **Visualización:** Gráficas comparativas para mejorar la experiencia del usuario (UX).

### Fase VII: Comunicación y Difusión
1.  **Reporte Técnico:** Documentación de cada fase del proyecto, resultados obtenidos y análisis de desempeño.
2.  **Video Promocional:** Creación de un video que resuma el proyecto, destacando su relevancia y aplicabilidad en el sector financiero. Además de mencionar las contribuciones individuales de cada integrante.

## **Navegación del Proyecto**

Para explorar los diferentes componentes de este trabajo, utilice los siguientes enlaces:

🔍 [**Exploración y Análisis Descriptivo**](exploracion.md) 

* *Análisis del diccionario, inspección de datos y validación de hipótesis iniciales.*

🧠 [**Modelos y Aprendizajes del Proceso**](modelos.md)  
* *Arquitectura de la red neuronal, optimización de hiperparámetros y métricas de desempeño.*
* *Hallazgos clave y lecciones aprendidas durante el entrenamiento y ajuste de los modelos.*

🏦 [**Caso de Uso y Scorecard**](casodeuso.md)  
* *Implementación del sistema de puntaje y aplicación práctica del modelo.*

🌐 [**Sitio Web y Video Promocional**](sitio-web.md)  
* *Acceso a la aplicación interactiva y presentación de la solución final.*
* *Detalle de los aportes específicos de cada integrante del equipo.*

<br>
<div align="center">
  <a href="https://github.com/jihernandezc/rnaab_riesgo_crediticio" target="_blank">
    <img src="https://img.shields.io/badge/GitHub-Ver_Repositorio_Oficial-181717?style=for-the-badge&logo=github" alt="Ver en GitHub">
  </a>
</div>