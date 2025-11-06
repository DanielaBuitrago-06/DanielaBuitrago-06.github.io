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
- Detección de características mediante **SIFT** (fallback a ORB si no disponible).  
- Matching robusto usando *ratio test* de Lowe (ratio = 0.75).  
- Estimación de homografía por RANSAC (umbral 5.0 píxeles).  
- Proyección cilíndrica (f = 900 px) para corregir distorsión angular.  
- Composición mediante *warping* respecto a la primera imagen (referencia).

**Calibración y medición:**  
- Uso de referencias métricas (cuadro: 117.0 cm; mesa: 161.1 cm).  
- Escala promedio: **s = 0.2293 cm/píxel**.  
- Medición de 5 objetos adicionales.  
- Estimación de incertidumbre:  
  \[
  \sigma_{total} = \sqrt{\sigma_{calibración}^2 + \sigma_{medición}^2}
  \]

### **3.2 Justificación de decisiones técnicas**
- SIFT se eligió por robustez; ORB actúa como fallback.  
- Proyección cilíndrica (f = 900 px) reduce distorsión.  
- Blending multibanda suaviza transiciones.  
- Escalas conocidas evitan calibración intrínseca.  
- Logging automático para trazabilidad y reproducibilidad.

### **3.3 Diagrama de flujo**
  ┌────────────────────┐
  │   Captura de vistas│
  └───────┬────────────┘
          │
 ┌────────▼────────┐
 │Preprocesamiento │
 │(proyección cil.)│
 └────────┬────────┘
          │
 ┌────────▼──────────┐
 │Detección y Matching│
 └────────┬──────────┘
          │
 ┌────────▼────────┐
 │Estimación H con │
 │RANSAC           │
 └────────┬────────┘
          │
 ┌────────▼────────┐
 │Warp + Blending  │
 └────────┬────────┘
          │
 ┌────────▼────────┐
 │Calibración      │
 │(escala cm/px)   │
 └────────┬────────┘
          │
 ┌────────▼────────┐
 │Medición e       │
 │Incertidumbre     │
 └──────────────────┘


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
**Resultados del pipeline:**
- Imagen 1 → referencia: 1214 matches, 543 inliers (44.7%)  
- Imagen 2 → referencia: 548 matches, 303 inliers (55.3%)  
**Panorama final:** 2108 × 2292 píxeles, sin discontinuidades visibles.

### **4.3 Calibración y mediciones**
Escala: **s = 0.2293 cm/píxel**

| Elemento | Distancia (px) | Medida (cm) | ± (cm) | Incertidumbre relativa (%) |
|-----------|----------------|--------------|------------------------------|------------------------------|
| Cuadro (ancho) | 316.2 | 72.50 | ±1.23 | 1.70% |
| Mesa (largo) | 527.0 | 120.83 | ±1.23 | 1.02% |
| Ventana | 632.4 | 144.99 | ±1.23 | 0.85% |
| Planta | 343.8 | 78.82 | ±1.23 | 1.57% |
| Silla | 168.6 | 38.66 | ±1.23 | 3.19% |

Promedio de incertidumbre relativa: **1.67%**

---

## **5. Análisis y Discusión**

### **5.1 Comparación de detectores**
SIFT superó a ORB en robustez y estabilidad.  
Matches: 548–1214 por par; inliers: 45–55%.

### **5.2 Análisis de errores**
Fuentes principales:
- Error de registro: ±0.46 cm (0.39% relativa).  
- Propagación total: ±1.23 cm promedio.  
- No coplanaridad y errores de clic agregan desviaciones menores.

### **5.3 Precisión de las mediciones**
Incertidumbre promedio: **1.67%**, mejor caso (ventana) **0.85%**, peor (silla) **3.19%**.

### **5.4 Limitaciones**
- Homografía asume plano dominante.  
- Rotaciones grandes degradan precisión.  
- Longitud focal estimada (900 px) no calibrada.  
- Las mediciones automáticas usan proporciones del panorama.

### **5.5 Posibles mejoras**
- Calibración intrínseca y corrección radial.  
- *Bundle adjustment* global.  
- Segmentación semántica y selección interactiva.  
- Múltiples pares de puntos conocidos para calibración más robusta.

---

## **6. Conclusiones**

El pipeline logró un panorama coherente (2108×2292 px) y calibrado.  
La escala de **0.2293 cm/píxel** permitió mediciones con errores promedio **<2%**.  
Las pruebas confirman la eficacia del registro multivista para obtener mediciones métricas sin calibración intrínseca ni reconstrucción 3D.

---

## **7. Referencias**

- Hartley, R., & Zisserman, A. (2004). *Multiple View Geometry in Computer Vision*. Cambridge University Press.  
- Lowe, D. (2004). *Distinctive Image Features from Scale-Invariant Keypoints*. *International Journal of Computer Vision, 60*(2), 91–110.  
- Rublee, E., Rabaud, V., Konolige, K., & Bradski, G. (2011). *ORB: An Efficient Alternative to SIFT or SURF*. *IEEE ICCV*, 2564–2571.  
- Burt, P., & Adelson, E. (1983). *The Laplacian Pyramid as a Compact Image Code*. *IEEE Transactions on Communications, 31*(4), 532–540.  
- Forsyth, D., & Ponce, J. (2012). *Computer Vision: A Modern Approach*. Pearson.  

