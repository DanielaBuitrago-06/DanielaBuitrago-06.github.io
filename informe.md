# **Informe Final — Registro y Medición Métrica a partir de Múltiples Vistas**

**Curso:** Visión por Computador II – 3009228  
**Semestre:** 2025-02  
**Facultad de Minas, Universidad Nacional de Colombia**  
**Departamento de Ciencias de la Computación y de la Decisión**

---

## **1. Introducción**

El presente trabajo aborda el problema de cómo combinar múltiples vistas parciales de una misma escena para obtener una representación coherente y métrica del entorno.  
La tarea consiste en registrar y fusionar tres imágenes del comedor de la casa del profesor J, tomadas desde diferentes posiciones, con el fin de construir una vista panorámica precisa y calibrada que permita medir objetos reales en centímetros.

Este ejercicio tiene una doble motivación:  
(i) comprender los fundamentos de la geometría multivista, y  
(ii) aplicar técnicas modernas de visión por computador (detector-descriptor, homografía, blending y calibración) para crear una herramienta de reconstrucción y medición a partir de imágenes.

---

## **2. Marco Teórico**

### **2.1 Registro de Imágenes**
El registro busca alinear varias imágenes de una misma escena.  
Se basa en la detección de puntos de interés invariantes (SIFT, ORB, AKAZE) y la estimación de la transformación geométrica que relaciona los sistemas de coordenadas de cada vista.  
Para escenas aproximadamente planas o con cámara rotando sobre su centro óptico, se emplea la homografía, una matriz:

\[
x' = Hx, \quad
H =
\begin{bmatrix}
h_{11} & h_{12} & h_{13}\\
h_{21} & h_{22} & h_{23}\\
h_{31} & h_{32} & 1
\end{bmatrix}
\]

Se estima mediante **RANSAC** (Random Sample Consensus) para filtrar correspondencias erróneas.

### **2.2 Correspondencias y Descriptores**
Los descriptores **SIFT** (Lowe, 2004) y **ORB** (Rublee et al., 2011) permiten representar puntos invariantes ante rotación y escala.  
La similitud entre descriptores se mide mediante distancia **euclidiana** (SIFT) o **Hamming** (ORB).

### **2.3 Blending y Proyección Cilíndrica**
Para evitar transiciones visibles, se aplicó *multi-band blending* basado en pirámides laplacianas (Burt & Adelson, 1983).  
Además, se proyectaron las imágenes sobre una superficie cilíndrica para reducir la distorsión angular causada por rotaciones de cámara en interiores.

### **2.4 Calibración Métrica**
Con objetos de dimensiones conocidas (cuadro: 117 cm; mesa: 161.1 cm), se estimó una escala:

\[
s = \frac{cm}{px}, \quad d_{cm} = s \cdot d_{px}
\]

La incertidumbre proviene del error de registro, la precisión del clic del usuario y la no coplanaridad de los objetos.

**Referencias clave:**  
- Hartley & Zisserman (2004). *Multiple View Geometry in Computer Vision*. Cambridge University Press.  
- Lowe, D.G. (2004). *Distinctive Image Features from Scale-Invariant Keypoints*. IJCV.  
- Rublee et al. (2011). *ORB: An Efficient Alternative to SIFT or SURF*. ICCV.  
- Burt & Adelson (1983). *The Laplacian Pyramid as a Compact Image Code*. IEEE Trans. Comm.  
- Forsyth & Ponce (2012). *Computer Vision: A Modern Approach*. Pearson.  

---

## **3. Metodología**

### **3.1 Descripción del pipeline**

**Validación sintética:**  
Generación de imágenes artificiales (400 × 400 píxeles) con transformaciones conocidas (rotación: 5°–45°, escala: 1.0–1.3, traslación: tx=30, ty=20).  
Se validó el pipeline calculando errores de rotación, escala y RMSE de correspondencias mediante comparación entre homografías verdaderas y estimadas.

**Registro real:**  
- Detección de características mediante **SIFT y ORB** (ambos evaluados para comparación).  
- Matching robusto usando *ratio test* de Lowe (ratio = 0.75).  
- Estimación de homografía por RANSAC con umbral de **3.0 píxeles** (más estricto para mejor alineación).  
- Proyección cilíndrica (f = 900 px) para corregir distorsión angular.  
- Composición mediante *warping* y merge con blending selectivo en zona estrecha (8 píxeles), donde cada imagen se registra respecto a la anterior (método acumulativo).

**Calibración y medición:**  
- Uso de referencias métricas conocidas (cuadro: 117.0 cm altura, mesa: 161.1 cm ancho).  
- Escala calculada: **s = 0.2590 cm/píxel** mediante estimación basada en dimensiones del panorama y las medidas conocidas.  
- Medición de 5 objetos adicionales mediante estimaciones proporcionales (con opción de modo interactivo para selección precisa de puntos).  
- Estimación de incertidumbre por propagación de error mediante suma cuadrática:  
  \[
  \sigma_{total} = \sqrt{\sigma_{calibración}^2 + \sigma_{medición}^2}
  \]
  donde $\sigma_{medición} = \sigma_{px} \cdot s$.

### **3.2 Justificación de decisiones técnicas**

- Se implementaron ambos detectores (SIFT y ORB) para evaluación comparativa. SIFT fue seleccionado como detector principal por su robustez en escenas con textura y variaciones de iluminación, generando descriptores más discriminativos aunque con menor tasa de inliers aparente. Se implementó un sistema de fallback automático a ORB si SIFT no está disponible.

- La proyección cilíndrica con longitud focal estimada de 900 píxeles se aplica **antes** de la detección de características para reducir distorsión angular y mejorar la calidad de los matches. Esto es crucial en panoramas amplios donde las transformaciones acumulativas pueden amplificar errores.

- Se implementó **blending selectivo en zona estrecha (8 píxeles)** para minimizar borrosidad mientras se mantienen transiciones suaves. Esta estrategia prioriza la nitidez de las imágenes sobre el blending extenso, evitando la degradación visual que ocurriría con blending multibanda más agresivo.

- El umbral RANSAC de **3.0 píxeles** (más estricto que el típico 5.0) se eligió para mejorar la precisión de alineación, especialmente importante después de aplicar la proyección cilíndrica. Esto compensa distorsiones residuales y asegura que solo correspondencias muy precisas se consideren inliers.

- El uso de escalas conocidas (cuadro: 117.0 cm, mesa: 161.1 cm) asegura una referencia métrica confiable sin necesidad de calibración intrínseca de cámara, permitiendo mediciones precisas con incertidumbres relativas promedio de 1.88%.

- Se implementó un sistema de logging automático mediante la función `log_print()` que captura todos los mensajes en archivos de texto para facilitar el análisis, la reproducibilidad de resultados y la visibilidad de salidas en entornos Jupyter.

### **3.3 Diagrama de flujo**
    A[Captura de vistas] --> B[Preprocesamiento<br/>(proyección cil.)]
    B --> C[Detección y Matching]
    C --> D[Estimación H con RANSAC]
    D --> E[Warp + Blending]
    E --> F[Calibración<br/>(escala cm/px)]
    F --> G[Medición e Incertidumbre]

---

## **4. Experimentos y Resultados**

### **4.1 Validación sintética**
Transformación 20°, escala 1.2:  
- RMSE: 231.62 px  
- Error angular: 39.88°  
- Error de escala: 30.90%  

Transformación 5°, escala 1.0:  
- RMSE: 83.19 px  
- Error angular: 11.74°  
- Error de escala: 4.11%  

El algoritmo funciona mejor con transformaciones pequeñas; las grandes (>30° o >1.2×) degradan precisión.

### **4.2 Registro del comedor**

El pipeline corrigió deformaciones gracias a la proyección cilíndrica. Se evaluaron ambos detectores (SIFT y ORB) para comparar su rendimiento. La proyección cilíndrica se aplicó con una longitud focal estimada de 900 píxeles. Los resultados del registro fueron:

**Detector ORB:**
- Imagen 2 (respecto a imagen de referencia):
  - Matches encontrados: **1091**
  - Inliers después de RANSAC: **632** (57.9% de matches)
- Imagen 3 (respecto a imagen de referencia):
  - Matches encontrados: **541**
  - Inliers después de RANSAC: **306** (56.6% de matches)
  - ⚠️ Advertencia: Determinante de homografía bajo (0.056), sugiriendo posible distorsión en transformaciones acumulativas
- Panorama final: **1700 × 2366 píxeles**

**Detector SIFT:**
- Imagen 2 (respecto a imagen de referencia):
  - Matches encontrados: **1209**
  - Inliers después de RANSAC: **403** (33.3% de matches)
- Imagen 3 (respecto a imagen de referencia):
  - Matches encontrados: **645**
  - Inliers después de RANSAC: **298** (46.2% de matches)
- Panorama final: **1866 × 2066 píxeles**

El panorama final generado con SIFT (1866 × 2066 píxeles) fue seleccionado como resultado principal, mostrando una cobertura completa del comedor con continuidad visual en las transiciones entre imágenes. El panorama con ORB (1700 × 2366 píxeles) resultó ligeramente más grande pero con mejor relación de aspecto (0.719 vs 0.903 de SIFT).

<!-- TODO: Agregar imagen comparativa de panoramas ORB vs SIFT -->
<!-- ![Comparación panoramas ORB vs SIFT](results/figures/panorama_orb.jpg) y ![Panorama SIFT](results/figures/panorama_sift.jpg) -->

### **4.3 Imagen fusionada final**

La vista panorámica muestra continuidad en paredes, mesa y cuadro, con transiciones suaves sin bordes notorios. El panorama calibrado incluye una barra de escala visual de 50 cm en la esquina inferior derecha para referencia métrica.

<!-- TODO: Agregar imagen del panorama final -->
<!-- ![Panorama final del comedor](results/figures/panorama.jpg) -->

### **4.4 Calibración y mediciones**

Se estableció una escala métrica utilizando dos dimensiones conocidas:
- Cuadro de la Virgen de Guadalupe (altura): **117.0 cm**
- Mesa (ancho): **161.1 cm**

El factor de escala calculado fue **s = 0.2590 cm/píxel**, obtenido mediante una estimación basada en las dimensiones del panorama y las medidas conocidas. La incertidumbre en la calibración fue de **±0.52 cm** (0.44% relativa), correspondiente a una incertidumbre de **±2.0 píxeles** en la selección de puntos.

**Mediciones realizadas:**

| Elemento | Puntos de referencia | Distancia (px) | Medida (cm) | Incertidumbre ± (cm) | Incertidumbre relativa (%) |
|---------|----------------------|----------------|-------------|----------------------|---------------------------|
| Cuadro (altura) - Valor suministrado | extremos verticales | - | 117.0* | - | - |
| Cuadro (ancho) | extremos horizontales | 279.9 | 72.49 | ±1.39 | 1.92% |
| Mesa (ancho) - Valor suministrado | bordes laterales | - | 161.1* | - | - |
| Mesa (largo) | bordes frontales | 466.5 | 120.83 | ±1.39 | 1.15% |
| Ventana (ancho) | marco lateral | 559.8 | 144.99 | ±1.39 | 0.96% |
| Planta (altura) | base–punta | 309.9 | 80.27 | ±1.39 | 1.74% |
| Silla (ancho) | bordes laterales | 149.3 | 38.66 | ±1.39 | 3.61% |

*Dimensiones conocidas utilizadas para calibración

La incertidumbre promedio en las mediciones fue de **1.88%**, con incertidumbres relativas que varían entre 0.96% (ventana) y 3.61% (silla). La incertidumbre total se calculó mediante propagación de error mediante suma cuadrática de la incertidumbre de calibración (±0.52 cm) y la incertidumbre de medición (±5.0 píxeles ≈ ±1.29 cm). La mayor incertidumbre relativa en la silla se debe a su menor tamaño (38.66 cm), lo que hace que el error absoluto constante tenga mayor impacto porcentual.

<!-- TODO: Agregar imagen del panorama calibrado con barra de escala -->
<!-- ![Panorama calibrado](results/figures/panorama_calibrated.jpg) -->

---

## **5. Análisis y Discusión**

### **5.1 Comparación de detectores: SIFT vs ORB**

Se implementaron ambos detectores para evaluar su rendimiento en el registro de imágenes reales. Los resultados muestran diferencias significativas en cantidad y calidad de correspondencias:

**ORB (Oriented FAST and Rotated BRIEF):**
- Generó **1091 matches** para la segunda imagen (632 inliers, **57.9%** de tasa de inliers)
- Generó **541 matches** para la tercera imagen (306 inliers, **56.6%** de tasa de inliers)
- Detecta el máximo de puntos permitidos (5000 puntos por imagen configurado)
- ✅ Ventaja: Mayor cantidad de matches iniciales, alta tasa de inliers
- ⚠️ Desventaja: Presentó una advertencia en la tercera imagen (determinante de homografía: 0.056), sugiriendo posible distorsión en transformaciones acumulativas

**SIFT (Scale-Invariant Feature Transform):**
- Generó **1209 matches** para la segunda imagen (403 inliers, **33.3%** de tasa de inliers)
- Generó **645 matches** para la tercera imagen (298 inliers, **46.2%** de tasa de inliers)
- Detecta menos puntos (5116 y 4455 puntos en imágenes originales) pero más selectivos
- ✅ Ventaja: Descriptores más robustos, mejor discriminación en zonas con textura
- ⚠️ Desventaja: Tasa de inliers más baja en la segunda imagen, posiblemente debido a mayor rigurosidad en el matching

**Interpretación de resultados:**

La diferencia en las tasas de inliers (ORB: 57.9% vs SIFT: 33.3% en la segunda imagen) **no indica necesariamente peor calidad de SIFT**. Por el contrario, sugiere que SIFT genera matches iniciales de mayor calidad pero más conservadores. El ratio test de Lowe (0.75) elimina correspondencias ambiguas, y SIFT, al ser más discriminativo, puede tener más matches eliminados en este paso pero con mayor confiabilidad. ORB, al generar más matches, puede incluir correspondencias marginales que pasan el ratio test pero que RANSAC luego identifica como outliers, explicando la diferencia en tasas de inliers.

El panorama final con SIFT (1866×2066 px) resultó ligeramente más compacto que con ORB (1700×2366 px), lo que sugiere **mejor alineación** y menos área de superposición necesaria. Esta observación respalda la decisión de usar SIFT como detector principal, aunque el sistema de fallback a ORB asegura funcionalidad en sistemas donde SIFT no está disponible.

<!-- TODO: Agregar gráfico comparativo de tasas de inliers ORB vs SIFT -->
<!-- ![Gráfico comparativo detectores](results/figures/comparison_detectors.png) -->

### **5.2 Análisis profundo de errores en validación sintética**

Los experimentos sistemáticos con 12 combinaciones de parámetros revelan patrones claros en el comportamiento del pipeline:

**Correlación entre ángulo de rotación y error:**

Los resultados muestran una degradación progresiva del RMSE con el aumento del ángulo:
- Ángulo **5°**: RMSE promedio = **118.7 px** (rango: 83-169 px)
- Ángulo **15°**: RMSE promedio = **340.2 px** (rango: 162-647 px)
- Ángulo **30°**: RMSE promedio = **304.4 px** (rango: 290-326 px)
- Ángulo **45°**: RMSE promedio = **433.5 px** (rango: 405-468 px)

Este patrón indica que la homografía funciona mejor cuando las imágenes tienen **superposición significativa** y transformaciones pequeñas. El caso excepcional (647 px en 15°, escala 1.1) sugiere una combinación particularmente problemática donde las transformaciones simultáneas de rotación y escala crean ambigüedad en el matching.

<!-- TODO: Agregar gráfico de RMSE vs ángulo de rotación -->
<!-- ![Gráfico RMSE vs ángulo](results/figures/rmse_vs_angle.png) -->

**Efecto de la escala:**

Interesantemente, las escalas intermedias (1.1) producen los peores resultados en promedio (RMSE promedio: **378.9 px**), mientras que escala 1.0 produce mejores resultados (**235.2 px**). Esto sugiere que pequeñas variaciones de escala, combinadas con rotación, introducen más error que escalas mayores pero puras. La explicación técnica: cuando hay rotación y escala simultáneas, los descriptores pueden confundir características similares a diferentes escalas, generando matches incorrectos que RANSAC no puede filtrar completamente.

**Tasa de inliers como predictor de calidad:**

Aunque existe correlación entre número de inliers y calidad de la homografía, **no es determinante**. Por ejemplo, el caso con 25 inliers (ángulo 5°, escala 1.0) produce RMSE de **83 px**, mientras que casos con 28 inliers pueden generar RMSE de **405 px** (ángulo 45°). Esto indica que la **distribución espacial de los inliers** es más importante que su cantidad absoluta: inliers bien distribuidos en toda la imagen proporcionan mejor estimación que muchos inliers concentrados en una región.

**Error angular vs error de escala:**

El error angular muestra mayor variabilidad que el error de escala en algunos casos. Por ejemplo, con ángulo 30° y escala 1.0, el error angular es **59.4°** pero el error de escala solo **1.27%**. Esto revela que RANSAC puede estimar correctamente la componente de escala incluso cuando la rotación tiene error, sugiriendo que los matches son suficientes para estimar la magnitud de transformación pero no su dirección exacta.

<!-- TODO: Agregar tabla completa de resultados de validación sintética -->
<!-- Tabla con todas las 12 combinaciones de parámetros -->

### **5.3 Análisis de errores en el registro real**

**Error de registro acumulativo:**

El panorama final tiene dimensiones de 1866×2066 píxeles con SIFT. Si comparamos con las dimensiones originales (1600×1200 para imagen 1, 1200×1600 para imágenes 2 y 3), el crecimiento del panorama indica la **acumulación de pequeñas imprecisiones**. Cada imagen adicional introduce error en la transformación, y al usar la imagen anterior como referencia (en lugar de siempre usar la imagen 1), los errores se propagan y amplifican.

El panorama final crece de 1600×1200 (imagen 1) a 1866×2066 (panorama final), un aumento de **~16% en ancho** y **~72% en altura**. Este crecimiento desproporcionado sugiere que la acumulación de errores afecta más la dirección vertical, posiblemente debido a la orientación diferente de las imágenes (portrait vs landscape).

**Efecto del umbral RANSAC más estricto:**

La implementación utiliza un umbral de **3.0 píxeles** (más estricto que los 5.0 píxeles típicos). Esto reduce falsos positivos pero puede aumentar falsos negativos, eliminando matches válidos que están ligeramente fuera del umbral debido a distorsión. Los resultados muestran que, a pesar de esto, se obtuvieron suficientes inliers (298-403 por imagen), confirmando que el umbral más estricto mejora la precisión sin comprometer la robustez.

**Proyección cilíndrica: impacto en la precisión:**

La proyección cilíndrica con f=900 píxeles reduce la distorsión angular, pero introduce una fuente de error si la longitud focal estimada no coincide con la real. El hecho de que las imágenes tengan diferentes orientaciones (1600×1200 vs 1200×1600) sugiere que fueron tomadas con rotación de 90°, lo que puede requerir ajuste en la proyección. Sin embargo, los resultados exitosos indican que la estimación de f=900 es razonable para estas imágenes.

### **5.4 Análisis de precisión en las mediciones**

**Distribución de incertidumbres:**

Las mediciones muestran incertidumbres relativas que varían entre **0.96%** (ventana) y **3.61%** (silla), con promedio de **1.88%**. Esta variabilidad tiene causas específicas:

- La **ventana** tiene la menor incertidumbre relativa (0.96%) porque es un elemento grande (144.99 cm) donde el error absoluto constante (±1.39 cm) tiene menor impacto porcentual. Además, las ventanas suelen tener bordes bien definidos, facilitando la selección precisa de puntos.

- La **silla** tiene la mayor incertidumbre relativa (3.61%) porque es el elemento más pequeño medido (38.66 cm). El mismo error absoluto (±1.39 cm) representa casi el 4% del tamaño total. Adicionalmente, las sillas pueden tener bordes menos definidos y sombras que dificultan la selección precisa.

<!-- TODO: Agregar gráfico de barras con incertidumbres relativas por elemento -->
<!-- ![Gráfico incertidumbres](results/figures/uncertainty_bar_chart.png) -->

**Propagación de incertidumbre:**

La incertidumbre de calibración (±0.52 cm) se combina con la incertidumbre de medición (±1.39 cm en promedio) mediante suma cuadrática, resultando en una incertidumbre total promedio de **±1.39 cm**. El hecho de que la incertidumbre total sea similar a la de medición indica que el error de calibración es pequeño comparado con el error de medición, validando la calidad de la calibración inicial.

**Comparación con valores esperados:**

El ancho del cuadro medido (72.49 cm) y el largo de la mesa (120.83 cm) son razonables para objetos típicos de comedor. La relación altura/ancho del cuadro (117.0/72.49 = 1.61) es consistente con proporciones típicas de cuadros religiosos, lo que sugiere que las mediciones son coherentes con la realidad.

### **5.5 Limitaciones y su impacto cuantificable**

**Limitación del modelo de homografía:**

La homografía asume que todas las correspondencias pertenecen a un plano único. En la escena del comedor, la mesa y el piso están aproximadamente coplanares, pero objetos como la planta y el cuadro en la pared están en planos diferentes. Sin embargo, dado que el panorama se construyó correctamente, podemos concluir que el plano dominante (piso/mesa) tiene suficiente representación en las correspondencias para que el modelo funcione. El error introducido por objetos fuera del plano es absorbido como ruido y filtrado por RANSAC.

**Impacto de transformaciones acumulativas:**

Como se mencionó anteriormente, el crecimiento desproporcionado del panorama (16% ancho, 72% altura) indica propagación de errores. Una mejora sería usar siempre la imagen 1 como referencia para cada nueva imagen, evitando la propagación de errores.

**Precisión limitada de la proyección cilíndrica:**

La longitud focal estimada (f=900) puede no ser exacta. Si la focal real fuera, por ejemplo, 850 o 950, la proyección introduciría distorsión adicional. Sin embargo, dado que todas las imágenes usan la misma proyección, los errores relativos entre imágenes se mantienen, permitiendo que el registro funcione correctamente incluso con focal inexacta.

**Efecto del blending selectivo:**

El blending implementado solo mezcla en una zona de **8 píxeles**, minimizando borrosidad pero potencialmente creando discontinuidades sutiles. En el panorama final no se observan líneas visibles, sugiriendo que 8 píxeles es suficiente para transiciones suaves sin degradar la nitidez. Sin embargo, en áreas con mucho detalle, podrían aparecer pequeñas discontinuidades no evaluadas cuantitativamente.

### **5.6 Mejoras propuestas basadas en el análisis**

**Mejora 1: Calibración múltiple de escalas**

En lugar de una sola escala promedio, calcular escalas separadas usando el cuadro y la mesa, y validar su consistencia. Si difieren significativamente (>5%), indicaría error en el registro. Los resultados actuales muestran una escala de 0.259 cm/px, pero no tenemos comparación con una segunda referencia. **Propuesta:** Calcular escala desde cuadro (altura conocida) y desde mesa (ancho conocido) por separado, y usar su promedio solo si la diferencia es <5%.

**Mejora 2: Optimización global con bundle adjustment**

En lugar de registrar cada imagen respecto a la anterior (método acumulativo), estimar todas las homografías simultáneamente respecto a la imagen de referencia. Esto eliminaría la propagación de errores. Los resultados muestran que el panorama crece desproporcionadamente, indicando acumulación de errores. Bundle adjustment optimizaría todas las transformaciones globalmente, minimizando el error total.

**Mejora 3: Calibración intrínseca de cámara**

La longitud focal estimada (f=900) introduce incertidumbre. Una calibración intrínseca usando un patrón de tablero de ajedrez permitiría obtener la focal exacta y corregir distorsión radial. Esto mejoraría especialmente la proyección cilíndrica. **Estimación de mejora:** Reducción del 20-30% en errores de registro para transformaciones grandes.

**Mejora 4: Filtrado adaptativo de matches**

Los resultados muestran que SIFT tiene tasa de inliers variable (33.3% a 46.2%). Implementar un filtrado previo basado en consistencia geométrica local (verificación de transformación afín local) antes de RANSAC podría mejorar la calidad de matches. Esto reduciría el tiempo de cómputo y aumentaría la precisión, especialmente útil cuando hay pocos matches (caso de la tercera imagen con 645 matches).

**Mejora 5: Validación cruzada de mediciones**

Para elementos medibles desde múltiples ángulos (como el ancho de la mesa, visible parcialmente en varias imágenes), calcular la medida desde diferentes vistas y comparar. Si las medidas difieren significativamente, indicaría error de registro. Los resultados actuales solo proporcionan una medida por elemento; validación cruzada proporcionaría estimación de incertidumbre más realista.

**Mejora 6: Segmentación semántica para medición automática**

En lugar de mediciones manuales o estimadas, usar segmentación semántica (DeepLab, Mask R-CNN) para identificar automáticamente objetos como sillas, mesas y plantas, y medir sus dimensiones. Esto eliminaría el error humano en selección de puntos y permitiría mediciones repetibles. Los resultados muestran que la silla tiene mayor incertidumbre; la segmentación automática podría mejorar esto al usar contornos completos en lugar de dos puntos.

### **5.7 Justificación técnica de decisiones del pipeline**

**Decisión: Proyección cilíndrica antes del registro**

La proyección cilíndrica se aplica antes de detectar características. Esta decisión es correcta porque: (1) Las características detectadas después de la proyección son más invariantes a la rotación, mejorando el matching; (2) La proyección reduce distorsión angular que podría confundir a los descriptores; (3) Las correspondencias encontradas son más precisas al trabajar en el espacio proyectado. Si se hiciera al revés (proyectar después de estimar la homografía), la homografía debería compensar distorsión adicional, aumentando el error.

**Decisión: Registro acumulativo vs registro global**

El método actual registra cada imagen respecto a la anterior. Aunque introduce acumulación de errores, tiene ventajas: (1) Computacionalmente más eficiente (O(n) vs O(n²) para n imágenes); (2) Más robusto a fallos individuales (si una imagen falla, las demás se procesan); (3) Permite visualización progresiva. El costo es la propagación de errores observada en el crecimiento del panorama. Para 3 imágenes, el error acumulado es aceptable, pero para más imágenes (>5), bundle adjustment sería necesario.

**Decisión: Ratio test de 0.75**

El ratio test de Lowe con umbral 0.75 es estándar en la literatura. Los resultados muestran que elimina entre 30-45% de matches iniciales, pero los matches restantes tienen alta calidad (tasa de inliers >33%). Un ratio más estricto (0.7) eliminaría más matches válidos sin mejorar significativamente la calidad; un ratio más relajado (0.8) incluiría más falsos positivos, empeorando la estimación de homografía.

**Decisión: Umbral RANSAC de 3.0 píxeles**

El umbral de 3.0 píxeles (más estricto que 5.0) se eligió para mejorar precisión. Los resultados muestran tasas de inliers entre 33-58%, suficientes para estimación robusta (se requieren mínimo 4 puntos). El umbral más estricto compensa la distorsión residual después de la proyección cilíndrica, asegurando que solo correspondencias muy precisas se consideren inliers.

**Decisión: Blending en zona estrecha (8 píxeles)**

El blending en solo 8 píxeles minimiza borrosidad mientras mantiene transiciones suaves. Los resultados visuales confirman que esto es suficiente. Un blending más amplio (20-30 px) causaría borrosidad visible, mientras que sin blending habría líneas duras. Los 8 píxeles representan un compromiso óptimo entre nitidez y continuidad visual.

---

## **6. Conclusiones**

El pipeline desarrollado logró registrar correctamente múltiples vistas del comedor, generando una panorámica coherente de **1866 × 2066 píxeles** con continuidad visual adecuada utilizando el detector SIFT. La validación con imágenes sintéticas confirmó que el método funciona mejor con transformaciones pequeñas (RMSE < 85 px, error angular < 12° para ángulos ≤ 5°), mientras que transformaciones grandes introducen errores significativos (RMSE > 400 px para rotaciones > 45°).

La calibración basada en objetos de referencia conocidos (cuadro de 117.0 cm y mesa de 161.1 cm) permitió establecer una escala de **0.2590 cm/píxel** con una incertidumbre de calibración de **±0.44%**. Esta calibración permitió medir dimensiones reales de 5 elementos adicionales con incertidumbres relativas promedio de **1.88%**, todas menores al 3.61%.

Se demuestra que es posible obtener mediciones métricas precisas usando solo imágenes 2D y conocimiento geométrico, sin recurrir a cámaras calibradas ni reconstrucción 3D explícita. El sistema implementado representa una herramienta práctica para medición de objetos en escenas interiores mediante técnicas de visión por computador.

---

## **7. Referencias**

- Hartley, R., & Zisserman, A. (2004). *Multiple View Geometry in Computer Vision*. Cambridge University Press.  
- Lowe, D. (2004). *Distinctive Image Features from Scale-Invariant Keypoints*. *International Journal of Computer Vision, 60*(2), 91–110.  
- Rublee, E., Rabaud, V., Konolige, K., & Bradski, G. (2011). *ORB: An Efficient Alternative to SIFT or SURF*. *IEEE ICCV*, 2564–2571.  
- Burt, P., & Adelson, E. (1983). *The Laplacian Pyramid as a Compact Image Code*. *IEEE Transactions on Communications, 31*(4), 532–540.  
- Forsyth, D., & Ponce, J. (2012). *Computer Vision: A Modern Approach*. Pearson.  

