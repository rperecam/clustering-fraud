# Análisis Integral: K-Means en la Detección de Fraude

Esta es una profundización detallada que toca el punto neurálgico del análisis de clústeres en detección de anomalías: la tensión entre la naturaleza binaria del problema (fraude/no fraude) y la naturaleza geométrica de los datos.

A diferencia de DBSCAN, que busca densidades, K-Means intenta dividir el espacio en $k$ grupos esféricos. El objetivo aquí es verificar si los fraudes se agrupan en una región específica del hiperespacio o si quedan diluidos entre las transacciones normales, conectando la perspectiva operativa con los resultados matemáticos.

---

### 1. La Paradoja de los Clústeres: Selección del $k$ y Geometría
Es natural pensar: *"Si busco dos clases (Fraude y Normal), debería pedirle al algoritmo 2 grupos ($k=2$)"*. Sin embargo, esta intuición falla en problemas de altísimo desbalance (0.17% fraudes vs 99.83% normales).

#### **La Explicación Geométrica**
K-Means no "sabe" qué es fraude. Solo sabe de distancias y varianza.
* **Si forzamos $k=2$:** El algoritmo dividirá el espacio basándose en la varianza más grande de los datos (por ejemplo, transacciones de alto valor vs. bajo valor). Como los fraudes son tan pocos y pesan tan poco matemáticamente, quedan "diluidos" e invisibles dentro de uno de los dos grupos masivos.
* **Al usar $k=9$ (Sub-segmentación táctica):** Permitimos que el algoritmo rompa los grandes grupos monolíticos en piezas más pequeñas. Al hacer el "grano más fino", permitimos que un pequeño grupo de fraudes, que tienen características muy similares entre sí (mismo monto, misma hora, mismo patrón), se separe y forme su propio "islote" o clúster independiente.

#### **Resultados Experimentales por $k$**
Analizando el rendimiento con diferentes valores, observamos este comportamiento:
* **$k=4$ (Sub-segmentación insuficiente):** El modelo es demasiado general. Los fraudes quedan ocultos dentro de grandes grupos normales (Recall: ~6%, F1: 0.01).
* **$k=13$ (Sobre-segmentación):** Aunque el Recall sube al 78%, la precisión se desploma al 1.2%. El modelo empieza a romper grupos normales y mezclarlos con ruido, perdiendo utilidad práctica.
* **$k=9$ (El "Sweet Spot"):** Aquí ocurre el fenómeno más interesante. Usamos $k=9$ no porque haya 9 tipos de fraude, sino porque necesitamos 9 divisiones del espacio para lograr "recortar" con precisión la pequeña región donde se esconde el fraude sin arrastrar miles de datos normales.

---

### 2. Profiling del "Clúster de Oro" (ID 4)
La tabla de distribución de clústeres nos ofrece el hallazgo más valioso de este modelo: la identificación del **Clúster ID 4**.

#### **Estadísticas del Clúster 4**
* Este grupo contiene solo **48 transacciones**.
* De ellas, **43 son fraudes reales**.
* **Tasa de Fraude (Pureza): 89.58%**.
* **Falsos Positivos:** Solo 5 transacciones legítimas cayeron en este grupo de alto riesgo.

#### **Interpretación de Negocio**
K-Means ha encontrado un patrón de fraude muy específico y repetitivo (probablemente ataques automatizados o un mismo modus operandi) que se separa geométricamente del resto.
El hecho de que la tasa sea del 90% implica un **Lift (Elevación)** astronómico:

$$\text{Lift} = \frac{89.58\%}{0.17\%} \approx 526$$

Significa que una transacción que cae en el Clúster 4 tiene **526 veces más probabilidad** de ser fraude que una transacción aleatoria.

#### **El Fraude "Diluido"**
El resto de los fraudes (aproximadamente 55 casos) están dispersos en los clústeres gigantes (ID 1, 2 y 7). En estos grupos, la tasa de fraude es insignificante (<0.2%). Esto confirma lo visto en UMAP: hay fraudes "fáciles" (aislados, detectados por el Clúster 4) y fraudes "difíciles" (camuflados entre la normalidad).

---

### 3. El Umbral de Decisión y Acción Operativa
¿Cuándo un Clúster es "Culpable"? En tu código, la decisión no es arbitraria, se basa en un Umbral Dinámico.

#### **La Fórmula de la Alerta**
Para etiquetar el Clúster 4 como "Fraude", el algoritmo evaluó internamente:
$$\text{Tasa del Clúster} > \text{Tasa Global} \times \text{Multiplicador}$$

* **Tasa Global ($P_{global}$):** ~0.17%.
* **Tasa del Clúster ($P_{cluster}$):** $43 / 48 = 0.8958$ (89.58%).
* **La Decisión:** El umbral dinámico (con multiplicador 2.0) sería $0.17\% \times 2 = 0.34\%$. Como $89.58\% \gg 0.34\%$, se activa la ALERTA.

#### **Conclusión Operativa**
K-Means con $k=9$ no sirve para detectar *todo* el fraude, pero es excelente para detectar un *tipo específico* con altísima confianza.
* **Acción:** Las transacciones que caigan en el **Clúster 4 pueden ser bloqueadas automáticamente** (debido a la certeza del 90%).
* **Estrategia Híbrida:** K-Means y DBSCAN son complementarios. K-Means atrapa los casos obvios automáticamente, y DBSCAN filtra los casos raros para investigación manual.

---

### 4. Comparativa de Modelos

#### **Vs. DBSCAN**
* **Precisión:** DBSCAN tenía una precisión del 19%, mientras que el Clúster 4 de K-Means tiene una precisión del **~90%**.
* **Recall:** K-Means (43.8%) detecta menos de la mitad del fraude total, pero lo que detecta es casi seguro fraude. DBSCAN ofrece un enfoque diferente basado en densidad y ruido.

#### **Vs. Método de Ward**
Mencionas que Ward arrojó resultados similares pero con matices interesantes:
* **K-Means:** Recall 44% | Precisión 90%
* **Ward:** Recall 62.5% | Precisión 71.4%

**¿Por qué la diferencia?**
Tanto K-Means como Ward intentan minimizar la varianza intra-clúster. Sin embargo:
* **K-Means (El Francotirador):** Fue más estricto. Aisló el núcleo más puro del fraude (Clúster 4), logrando una precisión casi perfecta pero dejando fuera a los fraudes periféricos.
* **Ward (La Red de Pesca):** Al ser jerárquico, fusiona puntos cercanos. Probablemente, el clúster de fraude de Ward incluyó el "núcleo puro" que encontró K-Means más una capa externa de puntos vecinos que contenía más fraudes (subiendo el Recall al 62%) pero también más transacciones legítimas "ruidosas" (bajando la precisión al 71%).

Ward sugiere que podríamos ser un poco más flexibles; podríamos expandir los límites del Clúster 4 para atrapar más fraudes a cambio de aceptar un poco más de falsos positivos, lo cual sigue siendo operativamente muy aceptable.

---

### 5. Perspectiva a Futuro: ¿Qué ocurre con nuevos datos?
Si desplegamos este modelo ($k=9$) en producción, ocurren dos fenómenos que debemos gestionar:

1.  **La Trampa del "Overfitting" Geométrico:** El Clúster 4 ha detectado un modus operandi específico. Si los atacantes cambian su patrón, esos nuevos fraudes no caerán en el Clúster 4, sino que se dispersarán en los clústeres "limpios" (1, 2 o 7). La precisión se mantendrá alta, pero el Recall bajará.
2.  **Validación de Nuevos Clústeres:** Con el tiempo, podría surgir un "Nuevo Clúster 10" o el "Clúster 7" podría empezar a contaminarse.

**Solución:** Se requiere un monitoreo continuo de la Tasa de Fraude por Clúster. Si el Clúster 7 (que hoy es seguro) sube su tasa de fraude del 0.2% al 5%, debe reclasificarse como "Sospechoso" y enviarse a revisión manual.