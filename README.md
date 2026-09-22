# 🚀 Optimización Analítica a Escala Masiva: BigQuery vs Polars

> **Trabajo de Fin de Grado (Ingeniería del Software)**  
> Un análisis empírico sobre los límites de la arquitectura de datos masiva evaluando **Rendimiento, FinOps y GreenOps**. Desarrollado a raíz de un caso de uso real en **MásOrange**.

![Google Cloud](https://img.shields.io/badge/GoogleCloud-%234285F4.svg?style=for-the-badge&logo=google-cloud&logoColor=white)
![BigQuery](https://img.shields.io/badge/BigQuery-%23669DF6.svg?style=for-the-badge&logo=google-cloud&logoColor=white)
![Polars](https://img.shields.io/badge/Polars-%23FF7043.svg?style=for-the-badge&logo=polars&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB.svg?style=for-the-badge&logo=python&logoColor=white)

## 📌 Contexto y Motivación Corporativa

En entornos empresariales de telecomunicaciones como **MásOrange**, la gestión de datos escala a petabytes. Mientras que el anexado vertical de tablas masivas está resuelto mediante operaciones eficientes de metadatos (como `bq cp`), el **cruce horizontal (JOIN 1:1) masivo** sigue siendo un cuello de botella logístico y computacional.

Este proyecto nace para responder a una pregunta arquitectónica crítica:
*¿Es más eficiente delegar un JOIN masivo sacando los datos del Data Warehouse Serverless (BigQuery) hacia un motor externo en memoria RAM ultrapotente (Polars)?*

Para responderla, no solo se midió el tiempo de ejecución, sino el **impacto financiero (FinOps)** y la **huella de carbono (GreenOps)**.

---

## 🏗️ Metodología y Entorno de Pruebas

Se diseñó un entorno de pruebas escalonado simulando cargas de producción, utilizando la clave de cliente (`id_baja`) optimizada criptográficamente (reducción del 78% del tamaño original usando `FARM_FINGERPRINT`).

*   **Datasets escalonados:** 2 GB, 10 GB, 50 GB y 100 GB.
*   **Entorno Nativo:** Google BigQuery (Motor Dremel, almacenamiento columnar Capacitor).
*   **Entorno Desacoplado:** Google Vertex AI (Instancias `n2-highmem` escalando hasta 864 GB de RAM) + Polars (Rust).
*   **Rigor estadístico:** Medias de 5 ejecuciones para datasets pequeños y 3 para masivos (mitigación de ruidos de red).

---

## 🔬 Desarrollo: La Hipótesis Polars y el Pivotaje Técnico

La premisa inicial era utilizar la función `hstack` de Polars, la cual promete una concatenación columnar física $O(1)$ sin evaluar el cruce comparándolo con diferentes técnicas de optimización que existen en BigQuery como el particionamiento o el clustering, por separado y combinándolas, para buscar la estrategia definitiva de cara a estas uniones 1:1 masivas. 

**El Descubrimiento:** El análisis reveló que `hstack` compromete la integridad relacional si los datos no están perfectamente pre-ordenados. El coste computacional de esta ordenación previa ($O(N \log N)$) anulaba por completo la ventaja temporal. 
**El Pivotaje:** Se descartó el atajo algorítmico en favor de un Hash JOIN estricto en memoria RAM para competir en igualdad de condiciones de integridad contra BigQuery.
---

## 📊 Resultados: La Triple Restricción

### 1. Rendimiento (Tiempo End-to-End)
Aislando estrictamente el cálculo matemático en RAM, Polars es hasta **4.7 veces más rápido** que la infraestructura distribuida de Google. Sin embargo, en una operación completa, la latencia de red (Extraer $\rightarrow$ Cargar $\rightarrow$ Ingestar) colapsa el rendimiento. Además, no existen técnicas de optimización dentro del entorno BigQuery que beneficien este tipo de operaciones en ningun aspecto.

<img width="2361" height="1461" alt="grafico_A_escalabilidad" src="https://github.com/user-attachments/assets/ba69047a-e35f-4d12-baa8-71784314cd65" />

A pesar de la clara victoria de BigQuery, merece la pena indagar más en el por qué de este resultado para encontrar lo que realmente consume la gran mayoria de los recursos en la estrategia de procesamiento externo: el movimiento de datos:

<img width="2936" height="1761" alt="grafico_B_cuellobotella" src="https://github.com/user-attachments/assets/4286380d-6cea-4b40-a9e2-87687565966d" />
<img width="2661" height="1461" alt="grafico_E_computopuro" src="https://github.com/user-attachments/assets/cfa8a079-2be1-4638-8f66-a45f612fe1e1" />


### 2. Impacto Financiero (FinOps)
El "Impuesto de inactividad". Pagar instancias gigantes (ej. `n2-highmem-128`) que permanecen ociosas el 90% del tiempo esperando la transferencia de red, sumado al coste volumétrico de extracción de datos, encarece el proceso de forma drástica.

<img width="2361" height="1460" alt="grafico_D_finops" src="https://github.com/user-attachments/assets/a2421dc4-3ffb-4b7e-b8db-3e33e50a3642" />
<img width="2661" height="1460" alt="grafico_F_impuesto_io" src="https://github.com/user-attachments/assets/a32dab1f-44b6-4c42-af60-2425c3889a74" />


### 3. Sostenibilidad (GreenOps)
Auditado mediante **CodeCarbon**. El movimiento masivo de paquetes de red TCP y las escrituras temporales en SSD generan una penalización termodinámica masiva, muy superior al procesamiento *serverless* ultra-optimizado (PUE 1.09) de los centros de datos de Google.

<img width="2661" height="1460" alt="grafico_C_sostenibilidad" src="https://github.com/user-attachments/assets/b5ab3cf2-e7e9-4857-900a-73a3ee8449d8" />

<img width="2661" height="1460" alt="grafico_I_eficiencia_energetica_pura" src="https://github.com/user-attachments/assets/3ae2c51c-a027-460f-ac38-6588228ae086" />


---

## 🪐 Conclusión Arquitectónica: "Data Gravity"

Este proyecto demuestra empíricamente el principio de la **Gravedad de los Datos (Data Gravity)**. 

A escala Big Data, el coste temporal, económico y energético de mover la información por la red siempre superará la brillantez algorítmica de un motor local. La eficiencia pura en memoria RAM no justifica la logística de extracción.

**Veredicto corporativo:** Es infinitamente más eficiente llevar el código a donde residen los datos, que extraer los datos para alimentar el código. Se valida el uso in-situ de BigQuery como solución óptima para uniones masivas 1:1.

---

## 🛠️ Stack Tecnológico

*   **Data Warehouse & Cloud:** Google Cloud Platform, BigQuery, Vertex AI, Cloud Storage.
*   **Procesamiento de Datos:** Polars, SQL (Dremel).
*   **Auditoría y Visualización:** CodeCarbon (GreenOps), Matplotlib, Seaborn, Python 3.
*   **Arquitectura:** Diseño de flujos ETL masivos, optimización de tipos de datos, evaluación FinOps/GreenOps.

---

Ingeniero orientado a la optimización de arquitecturas de datos masivas. Apasionado por resolver problemas complejos de infraestructura no solo desde la velocidad de ejecución, sino maximizando el impacto financiero (FinOps) y reduciendo la huella de carbono digital (GreenOps). Buscando retos en Data Engineering, Cloud Architecture y Backend a gran escala.
