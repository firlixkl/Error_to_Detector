# El espejismo del 99 %: por qué un accuracy altísimo no dice que un detector funcione

> **Serie «Del error al detector» · Capítulo 1**
> Pregunta: *¿por qué un 99 % de accuracy no implica que un detector de intrusiones funcione? ¿Qué cuentan los falsos positivos, los falsos negativos, el recall y la precisión que el accuracy esconde?*

> **Estado.** La hipótesis, las reglas C1–C3 y las predicciones (secciones 5.5 y 5.6) se fijaron **antes de ejecutar** con los datos reales. El cuaderno ya se ejecutó una vez con UNSW-NB15: C1, C2 y C3 se cumplieron. Al revisar *por qué*, se encontraron dos debilidades de diseño en C2 y C3; se añadió una **revisión exploratoria** (sección 5.8), que **aún no se ha ejecutado con los datos reales**. Las secciones 6, 7 y 8 siguen siendo un esqueleto a rellenar.

---

## Índice

1. [Resumen ejecutivo](#1-resumen-ejecutivo)
2. [Conceptos necesarios](#2-conceptos-necesarios)
3. [El dataset: UNSW-NB15](#3-el-dataset-unsw-nb15)
4. [Preparación de los datos](#4-preparación-de-los-datos)
5. [Diseño experimental](#5-diseño-experimental)
6. [Resultados](#6-resultados)
7. [Diagnósticos: ¿por qué pasa esto?](#7-diagnósticos-por-qué-pasa-esto)
8. [Interpretación](#8-interpretación)
9. [Errores cometidos y cómo se corrigieron](#9-errores-cometidos-y-cómo-se-corrigieron)
10. [Limitaciones y amenazas a la validez](#10-limitaciones-y-amenazas-a-la-validez)
11. [Próximos pasos](#11-próximos-pasos)
12. [Reproducción](#12-reproducción)
13. [Glosario](#13-glosario)
14. [Referencias](#14-referencias)

---

## 1. Resumen ejecutivo

| | |
|---|---|
| **Hipótesis (fijada antes de medir)** | Un accuracy ≥ 99 % es compatible con un detector inútil, o peor que no hacer nada, y no distingue entre detectores que se comportan de forma muy distinta |
| **Cómo se mide** | Tres reglas (C1, C2, C3) sobre un mismo modelo, variando la **prevalencia de ataques** y el **umbral** (sección 5.5) |
| **Modelo** | Regresión logística (umbral 0,5 y barrido de umbrales), más tres detectores «tontos» de referencia: todo normal, todo ataque y al azar |
| **Datos** | UNSW-NB15, los mismos que en [Azar vs. tiempo](../Azar-Tiempo/): 2.540.047 conexiones, 12,65 % de ataques |
| **Escenarios** | Reparto por tiempo (entrenar con la captura A, probar con la B) y reparto al azar con los mismos tamaños |
| **Predicción** | A prevalencia del 1 %, el modelo empata con «todo normal» con reparto al azar y **pierde** contra él con reparto por tiempo (sección 5.6) |
| **Veredicto pre-registrado** | C1 y C2 se cumplen → hipótesis confirmada según las reglas. **Con reservas:** C2 es casi trivial y C3 no aísla lo que dice (sección 5.8) |
| **Revisión exploratoria (C2', C3')** | Pendiente de ejecutar con los datos reales |

### Las tres ideas que se ponen a prueba

1. **El detector que no detecta nada también puede sacar un 99 %.** Su accuracy es `1 − p`, siendo `p` la proporción de ataques. Con `p = 1 %`, un 99 %.
2. **El accuracy cuenta errores distintos como si valieran lo mismo.** Un ataque que se escapa (falso negativo) y una falsa alarma (falso positivo) suman igual, aunque en un SOC no cuesten igual.
3. **Un modelo supera al detector que no hace nada solo si detecta más ataques que falsas alarmas lanza.** Es decir, solo si su precisión supera el 50 % (sección 2.3). Por debajo de esa **prevalencia crítica**, el accuracy premia al silencio.

---

## 2. Conceptos necesarios

### 2.1 La matriz de confusión

Todo lo que sigue sale de cuatro números. Para un detector de intrusiones:

| | **Alerta** (predice ataque) | **Sin alerta** (predice normal) |
|---|---|---|
| **Ataque real** | **TP**: ataque detectado | **FN**: ataque que se escapa |
| **Tráfico normal** | **FP**: falsa alarma | **TN**: normal correctamente ignorado |

### 2.2 Las métricas

Sea `p` la prevalencia (proporción de ataques), `TPR = TP / (TP + FN)` y `FPR = FP / (FP + TN)`.

| Métrica | Fórmula | Pregunta que responde |
|---|---|---|
| **Accuracy** | (TP + TN) / total | ¿Qué fracción acierto en total? |
| **Recall** (sensibilidad, TPR) | TP / (TP + FN) | De los ataques reales, ¿cuántos capturé? |
| **Tasa de falsos negativos (FNR)** | FN / (TP + FN) = 1 − recall | De los ataques reales, ¿cuántos se me escapan? |
| **Precisión** | TP / (TP + FP) | De mis alertas, ¿cuántas eran ataques? |
| **Tasa de falsos positivos (FPR)** | FP / (FP + TN) | De las conexiones normales, ¿cuántas disparan alarma? |
| **F1** | 2·TP / (2·TP + FP + FN) | Media armónica de precisión y recall |
| **Accuracy balanceado** | (recall + (1 − FPR)) / 2 | Media del acierto en cada clase, sin importar su tamaño |
| **MCC** (coeficiente de Matthews) | (TP·TN − FP·FN) / √((TP+FP)(TP+FN)(TN+FP)(TN+FN)) | Una sola nota que usa las cuatro casillas |

**Convención de la serie:** si una división es 0/0, la métrica está **sin definir** (`NaN`), nunca 0. Aquí importa especialmente: un detector que no lanza ninguna alerta **no tiene precisión**, porque no hay alertas de las que preguntar cuántas eran ciertas.

### 2.3 Por qué el 99 % no dice nada

**Razón 1: existe una línea base trivial.** El detector «todo normal» no lanza ninguna alerta y tiene `accuracy = 1 − p`. Con 100 ataques en 10.000 conexiones saca un 99 % de accuracy y un recall del 0 %.

**Razón 2: el accuracy depende de la prevalencia, no solo del modelo.** Con TPR y FPR fijos:

```
accuracy = TPR · p + (1 − FPR) · (1 − p)
```

El mismo detector, con las mismas virtudes y defectos, obtiene notas distintas según cuántos ataques haya en los datos. Lo mismo le pasa a la precisión, que además se hunde cuando `p` baja:

```
precisión = (TPR · p) / (TPR · p + FPR · (1 − p))
```

El recall y la FPR, en cambio, **no dependen de `p`**: solo miran una clase cada uno.

**Razón 3: el accuracy compara contra el silencio de una forma muy concreta.** Restando las dos fórmulas anteriores, en fracciones del total:

```
accuracy del modelo − accuracy de «todo normal» = TP − FP
```

El modelo solo supera en accuracy al detector que no hace nada si **detecta más ataques que falsas alarmas lanza**. Eso equivale a precisión > 50 %, y ocurre a partir de la **prevalencia crítica**:

```
p* = FPR / (TPR + FPR)
```

Por debajo de `p*`, el detector que no hace nada puntúa mejor en accuracy que el modelo. Y la ganancia máxima posible, con un modelo perfecto, es `p`: con `p = 1 %`, un punto.

### 2.4 Un ejemplo numérico (ilustrativo, no medido)

Un millón de conexiones con un 1 % de ataques (10.000 ataques):

| Detector | TP | FN | FP | TN | Accuracy | Recall | Precisión |
|---|---|---|---|---|---|---|---|
| A: todo normal | 0 | 10.000 | 0 | 990.000 | **99,00 %** | 0 % | sin definir |
| B: recall 90 %, FPR 0,5 % | 9.000 | 1.000 | 4.950 | 985.050 | **99,41 %** | 90 % | 64,5 % |
| C: recall 99 %, FPR 2 % | 9.900 | 100 | 19.800 | 970.200 | **98,01 %** | 99 % | 33,3 % |

Por accuracy, el orden es B > A > C. El detector C **encuentra casi todos los ataques**, pero queda por debajo del que no detecta ninguno. Ninguna de estas cifras sale de un experimento: es aritmética para fijar la idea. Lo que se pregunta es si pasa algo parecido con datos reales.

---

## 3. El dataset: UNSW-NB15

Se usa el mismo dataset que en el subcapítulo anterior. La descripción completa (origen, columnas, atajo de las IP, dos capturas) está en el [informe de Azar vs. tiempo](../Azar-Tiempo/README.md#3-el-dataset-unsw-nb15). Aquí solo se repite lo que importa para este experimento.

### 3.1 Composición

| Concepto | Valor |
|---|---|
| Conexiones | **2.540.047** |
| Ataques | **321.283 (12,65 %)** |
| Variables usadas | **41** (38 numéricas + 3 categóricas) |

| Captura | Conexiones | Ataques | % ataques | Accuracy de «todo normal» |
|---|---|---|---|---|
| **A** | 1.087.203 | 22.215 | 2,04 % | 97,96 % |
| **B** | 1.452.844 | 299.068 | 20,59 % | 79,41 % |
| **Total** | 2.540.047 | 321.283 | 12,65 % | 87,35 % |

*(Cifras del informe anterior; la última columna es `100 % − % de ataques`.)*

### 3.2 El problema: en estos datos el 99 % no aparece «de forma natural»

En ninguna parte del dataset el detector que no hace nada llega al 99 %. Para que lo hiciera, la prevalencia tendría que ser del 1 %, y aquí es del 2 % como mínimo (captura A) y del 12,65 % en total. Esto es una propiedad del banco de pruebas: los ataques están **sobrerrepresentados** frente a una red real, donde suelen ser una fracción muy pequeña del tráfico.

Por eso el experimento **simula** prevalencias más realistas (sección 5.3) en lugar de esperar a que el 99 % aparezca solo.

---

## 4. Preparación de los datos

Se reutiliza **exactamente** el preprocesado del subcapítulo anterior, para que los resultados sean comparables:

| Paso | Qué se hace |
|---|---|
| Limpieza y etiquetas | Conversión numérica, categorías sin espacios, `attack_cat` vacío → `Normal` |
| Orden | Por `Stime` (orden estable) |
| Fuera del modelo | IP, puertos, marcas de tiempo y etiquetas |
| Imputación y transformaciones | Mediana, `log1p`, estandarización, one-hot con frecuencia mínima 100 |
| Ajuste | Todo dentro de un `Pipeline`, ajustado **solo con el entrenamiento** de cada escenario |

**Modelo.** `LogisticRegression(max_iter=1000)` con valores por defecto, sin pesos de clase. Umbral de referencia **0,5**.

**Lo que cambia respecto al subcapítulo anterior:** aquí no hay bloques temporales ni pliegues. Solo se usa la frontera entre las capturas A y B (sección 5.2).

---

## 5. Diseño experimental

### 5.1 Qué se varía y qué se mantiene

En el subcapítulo anterior se variaba el reparto y se medían recall y precisión. Aquí **el modelo se entrena una vez por escenario** y se varían dos cosas sobre sus predicciones:

- la **prevalencia de ataques** de la prueba (sección 5.3),
- el **umbral de decisión** (secciones 5.5, C2 y C3).

### 5.2 Escenarios

Son el «caso duro» del subcapítulo anterior, para poder compararlos.

| Escenario | Entrenamiento | Prueba | Repeticiones |
|---|---|---|---|
| **tiempo (A → B)** | toda la captura A (1.087.203 filas) | toda la captura B (1.452.844 filas) | 1 (determinista) |
| **azar** | 1.087.203 filas al azar de todo el dataset | las 1.452.844 restantes | 5 (semillas `SEED + r`) |

De cada escenario se guardan las curvas TPR y FPR para una rejilla de 1.004 umbrales, que incluye un umbral por encima de 1 (el detector que **nunca alerta**).

### 5.3 Prevalencia simulada

TPR y FPR no dependen de la prevalencia, así que las métricas para cualquier `p` se calculan **exactamente** a partir de ellas (fórmulas de la sección 2.3). Como comprobación independiente, el cuaderno también construye conjuntos de prueba reales con cada prevalencia (submuestreando ataques o tráfico normal, 10 repeticiones) y compara.

| Prevalencia | Para qué sirve |
|---|---|
| 50 %, 20 %, 10 %, 5 % | De «balanceado» a «parecido al dataset» |
| **1 %** (`P_MAIN`) | La prevalencia en la que «todo normal» saca exactamente un 99 % |
| 0,1 % | Un caso más realista de red real, para ver qué pasa con la precisión |

### 5.4 Detectores de referencia

| Detector | TPR | FPR | Accuracy a prevalencia `p` |
|---|---|---|---|
| Todo normal | 0 | 0 | `1 − p` |
| Todo ataque | 1 | 1 | `p` |
| Al azar (con la prevalencia del entrenamiento `q`) | `q` | `q` | `1 − p − q + 2pq` |

### 5.5 Reglas de decisión fijadas de antemano

Evaluadas a prevalencia `P_MAIN = 1 %`. Con 2 escenarios y `MIN_FRAC_AGREE = 75 %`, cada regla debe cumplirse en **los 2**.

| Regla | Qué afirma | Parámetro | Valor |
|---|---|---|---|
| **C1** (principal) | El accuracy no distingue al modelo del detector que no hace nada | `ACC_TIE` | **0,5 puntos** |
| | | `MIN_RECALL_USEFUL` | 0,5 |
| **C2** (principal) | Mismo accuracy ≥ 99 %, detectores muy distintos | `ACC_MIRAGE` | 0,99 |
| | | `RECALL_RANGE_BIG` | **0,30** |
| **C3** (secundaria) | Optimizar el accuracy empuja hacia el silencio | `ACC_OPT_DROP` | **0,05** |

**C1.** Se calcula `Δ = accuracy del modelo (umbral 0,5) − accuracy de «todo normal»`:
- `|Δ| < 0,5 puntos` → **EMPATE**; `Δ ≤ −0,5` → **EL TONTO GANA**; `Δ ≥ +0,5` → **EL MODELO GANA CLARO**.
- Apoya la hipótesis si el modelo es útil (recall ≥ 0,5) y el resultado es EMPATE o EL TONTO GANA.
- *Por qué 0,5 puntos y no 1:* a `p = 1 %` la ganancia máxima posible es 1 punto. Con un umbral de 1 punto, la regla casi no podría fallar. Con 0,5, «empate» significa que el modelo **no elimina ni la mitad de los errores** del detector que no hace nada.

**C2.** Entre los umbrales del mismo modelo con `accuracy ≥ 99 %` **y recall > 0**, el recall debe variar 30 puntos o más. Se excluye el detector que nunca alerta para que la regla no sea trivial.

**C3.** El umbral que maximiza el accuracy debe tener al menos 5 puntos menos de recall que el umbral 0,5.

**Veredicto global:** **confirmada** si se cumplen C1 y C2; **confirmada solo en parte** si se cumple una; **refutada** si no se cumple ninguna. C3 se informa aparte y no decide.

> **Nota de método.** Estas reglas se fijaron **después de probar el cuaderno con datos sintéticos** (no con UNSW-NB15). Esa prueba reveló que el umbral de empate inicial de 1 punto era demasiado fácil de cumplir y se corrigió a 0,5 antes de tocar los datos reales. Se documenta en la sección 9.

**Descriptivas planificadas (sin regla de decisión):** tablas por prevalencia, matriz de confusión por millón de conexiones, prevalencia crítica `p*`, MCC y accuracy balanceado.

### 5.6 Predicciones antes de medir

Derivadas **solo** de las métricas redondeadas del informe anterior (caso duro, umbral 0,5) y de las fórmulas de la sección 2.3. El modelo se reentrena aquí, así que las cifras pueden variar ligeramente.

| Escenario | Recall | FPR | Accuracy a 1 % | Δ frente a «todo normal» | Precisión a 1 % | Prevalencia crítica `p*` |
|---|---|---|---|---|---|---|
| azar | 0,9710 | 0,0067 | ≈ 99,31 % | **+0,31 puntos** | ≈ 59 % | ≈ 0,69 % |
| tiempo (A → B) | 0,9932 | 0,0225 | ≈ 97,77 % | **−1,23 puntos** | ≈ 31 % | ≈ 2,2 % |

**Predicción para C1:** azar → EMPATE; tiempo → **EL TONTO GANA**. Se cumpliría en 2 de 2.
**Para C2 y C3 no hay predicción cuantitativa:** depende de cómo caiga el recall al subir el umbral, y es una incertidumbre real del experimento.

### 5.7 Diagnósticos planificados

1. **Re-muestreo real frente a cálculo exacto** (sección 7.1).
2. **Prevalencia crítica `p*`** y comprobación numérica de que en `p*` la precisión es 0,5 y el accuracy del modelo iguala al de «todo normal» (sección 7.2).
3. **Matriz de confusión por millón de conexiones** a `P_MAIN` (sección 7.3).

### 5.8 Revisión exploratoria de C2 y C3 (añadida tras ver los datos)

> **Todo lo de esta sección se decidió después de ver los resultados de la primera ejecución.** Por la regla de la serie, es **exploratorio**: sirve para entender cuánto depende la conclusión del diseño de las reglas, no para confirmar nada. **El veredicto pre-registrado (C1 y C2) no se cambia.**

**Debilidad de C2.** Se excluyó el detector que *nunca* alerta (recall = 0), pero no los que alertan *casi nunca*. En la primera ejecución el rango de recall salió de **0,008** a 0,948 (tiempo) y de 0,019 a 0,986 (azar). A prevalencia 1 %, cualquier umbral cercano a 1 tiene accuracy ≈ 99 % con recall minúsculo, así que la regla se cumple casi por construcción.

**Una identidad que lo explica.** A prevalencia `p`, `accuracy ≥ 1 − p` equivale a `TP ≥ FP`, es decir, a **precisión ≥ 50 %**. Por eso la precisión de los umbrales con accuracy ≥ 99 % salió con mínimo 0,500 en ambos escenarios: no es un hallazgo, es una consecuencia matemática. El cuaderno lo comprueba con un `assert`.

**Debilidad de C3.** Se comparó el umbral que maximiza accuracy contra el umbral **0,5**. Pero 0,5 es mal punto de operación a prevalencia 1 % (el modelo se entrenó con 2 % o 12,7 %), así que cualquier umbral mejor pierde recall frente a él, sea de accuracy o de F1. En la primera ejecución, además, el umbral óptimo de accuracy coincidió con el de F1 en el escenario por tiempo (0,996) y casi en el de azar (0,934 frente a 0,874). La regla no aislaba el efecto del accuracy.

**Reglas revisadas:**

| Regla | Qué cambia | Parámetro | Valor |
|---|---|---|---|
| **C2'** | Entre los umbrales con accuracy ≥ 99 %, solo los que detectan de verdad (recall ≥ suelo). El rango de recall debe seguir siendo ≥ 0,30. Se añade una **tabla de sensibilidad** con varios suelos, porque el suelo es arbitrario | `C2B_RECALL_FLOOR` | 0,5 |
| | | `C2B_FLOORS_SENS` | 0; 0,1; 0,3; 0,5; 0,7; 0,9 |
| **C3'** | Se compara el umbral que maximiza accuracy con el que maximiza **F1** (ambos «buenos»), a `P_MAIN`. Se muestra cómo evoluciona la diferencia al bajar la prevalencia | `ACC_OPT_DROP_VS_F1` | 0,05 |

**Descriptiva añadida (5.1.3 del cuaderno).** Con el mejor umbral de accuracy, ¿qué fracción de los errores de «todo normal» elimina el modelo a cada prevalencia? Separa dos cosas que C1 mezcla: que el **accuracy** sea mala métrica y que el **umbral 0,5** esté mal puesto para esa prevalencia.

**Lo que ya se sabe de la primera ejecución** (antes de ejecutar la revisión): con el mejor umbral de accuracy el modelo sacó 0,9979 (tiempo) y 0,9983 (azar) frente a 0,99 de «todo normal», es decir, **sí supera claramente al detector que no hace nada**. Los veredictos de C1 se deben en buena parte al umbral 0,5, no solo al accuracy como métrica. Esta es la matización más importante del experimento.

**Cómo se verificó el cuaderno revisado.** Con tres conjuntos sintéticos (fácil, intermedio y difícil) para comprobar que las reglas nuevas **pueden aprobar y suspender**. Resultado: en el fácil todo se refuta; en el intermedio C1 y C2 se cumplen pero C2' falla y C3' se cumple; en el difícil C2' no es evaluable. Esto demuestra que no son vacías, **no** dice nada sobre UNSW-NB15.

---

## 6. Resultados

> **Pendiente de ejecutar.** Las tablas son el esqueleto que genera el cuaderno. No hay datos.

### 6.1 El mismo modelo con distinta prevalencia (umbral 0,5)

| Escenario | Prevalencia | Accuracy modelo | Accuracy «todo normal» | Ganancia | Recall | Precisión | FPR | FP por millón | FN por millón |
|---|---|---|---|---|---|---|---|---|---|
| tiempo | 50 % … 0,1 % | … | … | … | … | … | … | … | … |
| azar | 50 % … 0,1 % | … | … | … | … | … | … | … | … |

### 6.2 Detectores de referencia a prevalencia 1 %

| Escenario | Detector | Accuracy | Recall | Precisión | FPR | TP | FN | FP |
|---|---|---|---|---|---|---|---|---|
| … | modelo / todo normal / todo ataque / al azar | … | … | … | … | … | … | … |

### 6.3 Veredicto de C1, C2 y C3

| Regla | tiempo (A → B) | azar | ¿Se cumple (2 de 2)? |
|---|---|---|---|
| C1 | … | … | … |
| C2 | … | … | … |
| C3 | … | … | … |

**Hipótesis global:** *pendiente*.

### 6.4 Umbrales con accuracy ≥ 99 % (C2)

| Escenario | Nº de umbrales | Recall (mín – máx) | Precisión (mín – máx) |
|---|---|---|---|
| … | … | … | … |

### 6.5 Optimizar el accuracy (C3)

| Escenario | Criterio | Umbral | Accuracy | Recall | Precisión | FP por millón | FN por millón |
|---|---|---|---|---|---|---|---|
| … | umbral 0,5 / máx. accuracy / máx. F1 | … | … | … | … | … | … |

### 6.6 Revisión exploratoria (C2', C3' y umbral razonable)

*Pendiente de ejecutar. Todo lo de este apartado es **exploratorio**.*

| Escenario | Suelo de recall | Nº umbrales | Recall (mín – máx) | Rango | FN por millón (en recall mín → en recall máx) |
|---|---|---|---|---|---|
| … | 0 / 0,1 / 0,3 / 0,5 / 0,7 / 0,9 | … | … | … | … |

| Escenario | Prevalencia | Umbral (accuracy) | Umbral (F1) | Recall (accuracy) | Recall (F1) | Pérdida de recall |
|---|---|---|---|---|---|---|
| … | 50 % … 0,1 % | … | … | … | … | … |

| Escenario | Prevalencia | Umbral | Accuracy modelo | Accuracy «todo normal» | Errores eliminados | Recall | Precisión |
|---|---|---|---|---|---|---|---|
| … | 50 % … 0,1 % | … | … | … | … | … | … |

### 6.7 Figuras

El cuaderno guarda tres figuras en `figuras/`:
- `fig1_accuracy_vs_prevalencia.png`: accuracy del modelo y de «todo normal» según la prevalencia.
- `fig2_precision_recall_vs_prevalencia.png`: la precisión se hunde con `p` y el recall no se entera.
- `fig3_accuracy_vs_recall_umbrales.png`: misma línea de accuracy, recall muy distinto.

<!-- Tras ejecutar, enlázalas aquí con ![descripción](figuras/nombre.png) -->

---

## 7. Diagnósticos: ¿por qué pasa esto?

### 7.1 Re-muestreo real frente a cálculo exacto

| Escenario | Prevalencia | Accuracy (re-muestreo) | Accuracy (exacto) | Diferencia máxima en accuracy, precisión y recall |
|---|---|---|---|---|
| … | … | … | … | … |

*Pendiente.* Si la diferencia máxima es pequeña, el cálculo exacto de la sección 6 es fiable.

### 7.2 Prevalencia crítica

| Escenario | TPR | FPR | `p*` | Prevalencia del dataset en la prueba |
|---|---|---|---|---|
| … | … | … | … | … |

*Pendiente.* Compara `p*` con la prevalencia real del dataset y con el 1 % simulado.

### 7.3 Matriz de confusión por millón de conexiones (prevalencia 1 %, umbral 0,5)

| | Alerta | Sin alerta |
|---|---|---|
| **Ataque real** | … | … |
| **Tráfico normal** | … | … |

*Pendiente, una por escenario.*

---

## 8. Interpretación

> **Se escribe después de ejecutar.** Preguntas guía, para separar lo planeado de lo descubierto:

1. ¿Se cumplió cada predicción de la sección 5.6? ¿Cuál no, y por qué?
2. ¿Dónde está `p*` frente a la prevalencia de una red real? Si está por encima, el modelo pierde contra el silencio en accuracy aunque detecte casi todo.
3. ¿Qué detector habría elegido un analista guiándose solo por el accuracy, y qué habría elegido mirando recall, precisión y FPR?
4. ¿Qué cambia del reparto al azar al reparto por tiempo? Conecta con la subida de falsas alarmas del subcapítulo anterior.
5. **Lo añadido tras ver los datos**, marcado como **exploratorio**.

---

## 9. Errores cometidos y cómo se corrigieron

| Versión | Error | Cómo se detectó | Corrección |
|---|---|---|---|
| Cuaderno v1 (antes de usar datos reales) | La regla C1 tenía un umbral de empate de **1 punto**, igual a la prevalencia principal (1 %). Como la ganancia máxima posible sobre «todo normal» es 1 punto, «EMPATE» saldría casi siempre, incluso con un modelo casi perfecto | Prueba del cuaderno con datos sintéticos: un modelo casi perfecto era clasificado como EMPATE | `ACC_TIE = 0,5` puntos y explicación de por qué en 5.5 |
| Cuaderno v1 (tras la 1.ª ejecución real) | **C2 casi trivial**: excluía el detector que nunca alerta, pero no los que alertan casi nunca (recall mín. 0,008 y 0,019) | Al revisar la salida: el rango de recall salía enorme por los umbrales cercanos a 1 | C2' con suelo de recall y tabla de sensibilidad (5.8). **C2 original se conserva** y su veredicto no se cambia |
| Cuaderno v1 (tras la 1.ª ejecución real) | **C3 confundido**: comparaba el umbral óptimo de accuracy contra el umbral 0,5, mal punto de operación a 1 %; la «pérdida de recall» venía sobre todo de ahí | El umbral óptimo de accuracy coincidía (o casi) con el de F1 | C3' compara contra el umbral de F1 (5.8). **C3 original se conserva** |
| Cuaderno v1 (prueba sintética) | `assert` de la prevalencia crítica fallaba si el modelo nunca alertaba (TPR y FPR ≈ 0, `p*` indefinida) | Prueba con datos sintéticos difíciles | Se salta la comprobación si `p*` no está en (0, 1) y se avisa |
| | *(se completa durante la ejecución de la revisión con los datos reales)* | | |

---

## 10. Limitaciones y amenazas a la validez

- **Un solo dataset**, generado en laboratorio con ataques sintéticos. Los resultados pueden no trasladarse al tráfico real.
- **Un solo modelo**, lineal. Un modelo más flexible tendría otra curva TPR/FPR y otra prevalencia crítica.
- **La prevalencia se simula.** Se asume que TPR y FPR no cambian con la prevalencia, es decir, que los ataques y el tráfico normal se parecen a sí mismos al reducir la proporción de ataques. La verificación por re-muestreo (7.1) lo comprueba para este dataset, pero **en una red real la prevalencia cambia con el tiempo y también la forma del tráfico**.
- **El modelo se entrena con la prevalencia original** (2 % en la captura A, 12,65 % en el reparto al azar), no con la simulada. Entrenar con un 1 % de ataques cambiaría el modelo y el significado del umbral 0,5.
- **Los umbrales de C2 y C3 se eligen mirando las etiquetas de prueba.** Son una ilustración de puntos de operación, no una forma legítima de seleccionar un modelo: seleccionarlo así sería filtrar información de la prueba.
- **Solo 2 escenarios**, así que cada regla exige 2 de 2. No es una prueba estadística formal.
- **El escenario por tiempo es una única ejecución determinista**, sin intervalo de variabilidad; solo el reparto al azar se repite.
- **Prevalencia y cambio temporal están confundidos** en el escenario A → B (heredado del subcapítulo anterior): entrenar con A (2 %) y probar con B (20,6 %) mezcla el efecto del tiempo con el de la prevalencia.
- **Las reglas son operacionalizaciones de «el 99 % no dice nada».** Los números (0,5 puntos, 30 puntos, 5 puntos) son decisiones del autor, fijadas antes de medir pero no derivadas de ninguna teoría.
- **C2' y C3' son exploratorias**: se diseñaron viendo los resultados de C2 y C3. Si salen «bien» no confirman nada; si salen «mal» tampoco refutan nada.
- **El suelo de recall de C2' (0,5) es una elección arbitraria.** La tabla de sensibilidad existe precisamente para ver cuánto depende la conclusión de ella.
- **Con el mejor umbral de accuracy el modelo supera a «todo normal»**, pero ese umbral se elige con las etiquetas de prueba: es una cota optimista, no el rendimiento esperable en producción.
- **El re-muestreo real de 6.1.1 tiene ruido** cuando hay pocos ataques o pocas alertas (umbrales altos); el cálculo exacto no.
- **El coste de los errores no se modela.** Se muestra cuántos FN y FP hay, no cuánto cuesta cada uno (capítulo 6).

---

## 11. Próximos pasos

1. **Repetir con modelos que no sean lineales** (árbol, bosque), para ver si cambia la prevalencia crítica.
2. **Función de coste y umbral basado en coste** (capítulo 6): ponerle precio a un FN y a un FP, y elegir el umbral con eso en lugar de con el accuracy.
3. **PR-AUC frente a ROC-AUC** a prevalencias bajas: cuál de las dos se entera del desbalanceo.
4. **Entrenar con la prevalencia simulada** (submuestrear ataques en el entrenamiento) y ver cómo se mueve el umbral efectivo.
5. **Un segundo dataset** con prevalencia natural más baja.
6. **Aplicar la prevalencia simulada a los pliegues** del subcapítulo anterior, no solo al caso A → B.

---

## 12. Reproducción

### Estructura del repositorio

```
.
├── README.md
└── Subcapitulos/
    └── Espejismo-99/
        ├── README.md                           # este informe
        ├── laboratorio_espejismo_99.ipynb
        ├── figuras/                            # se genera al ejecutar el cuaderno
        ├── resultados/                         # tablas en CSV, se genera al ejecutar
        └── UNSW-NB15/                          # NO se sube: se descarga aparte
            ├── UNSW-NB15_1.csv
            ├── UNSW-NB15_2.csv
            ├── UNSW-NB15_3.csv
            └── UNSW-NB15_4.csv
```

Añade `UNSW-NB15/` al `.gitignore`.

### Requisitos

- Python 3.9 o superior
- `pandas`, `numpy`, `scikit-learn >= 1.2`, `matplotlib`, `jupyter`

```bash
pip install pandas numpy "scikit-learn>=1.2" matplotlib jupyter
```

### Pasos

1. Descarga los cuatro CSV en bruto de UNSW-NB15 (los mismos del subcapítulo anterior) y colócalos en `UNSW-NB15/` junto al cuaderno. Los datos **no** se redistribuyen en este repositorio.
2. Abre el cuaderno desde esa carpeta:
   ```bash
   jupyter lab
   ```
3. Revisa la configuración de la primera celda **antes de ejecutar**:

| Parámetro | Valor | Significado |
|---|---|---|
| `SAMPLE_FRAC` | `1.0` | Fracción de datos (usa `0.25` para una prueba rápida) |
| `N_RANDOM_REPEATS` | `5` | Repeticiones del reparto al azar |
| `THRESHOLD` | `0.5` | Umbral de referencia |
| `PREVALENCES` | `[0.5, 0.2, 0.1, 0.05, 0.01, 0.001]` | Prevalencias simuladas |
| `P_MAIN` | `0.01` | Prevalencia principal |
| `N_RESAMPLES` | `10` | Re-muestreos reales de verificación |
| `ACC_MIRAGE` | `0.99` | Accuracy que se considera «el 99 %» |
| `ACC_TIE` | `0.005` | C1: umbral de empate (0,5 puntos) |
| `MIN_RECALL_USEFUL` | `0.5` | C1: recall mínimo para que el modelo sea «útil» |
| `RECALL_RANGE_BIG` | `0.30` | C2: rango mínimo de recall |
| `ACC_OPT_DROP` | `0.05` | C3: pérdida mínima de recall |
| `MIN_FRAC_AGREE` | `0.75` | Fracción de escenarios exigida |
| `SEED` | `42` | Semilla |
| `C2B_RECALL_FLOOR` | `0.5` | C2' (exploratoria): suelo de recall |
| `C2B_FLOORS_SENS` | `[0, .1, .3, .5, .7, .9]` | C2': suelos para la tabla de sensibilidad |
| `ACC_OPT_DROP_VS_F1` | `0.05` | C3' (exploratoria): pérdida mínima de recall frente a F1 |
| `RES_DIR` | `resultados` | Carpeta donde se exportan las tablas en CSV |

4. Ejecuta todas las celdas. Se entrena un modelo por escenario y repetición (6 con la configuración por defecto), así que con `SAMPLE_FRAC = 1.0` tarda **varios minutos o decenas de minutos**.

### Estructura del cuaderno

| Sección | Contenido |
|---|---|
| 0 | Configuración y reglas de decisión |
| 1 / 1.1 | Carga, limpieza, las dos capturas y la línea base «todo normal» |
| 2 | Piezas: modelo, métricas con «sin definir», curvas TPR/FPR, prevalencia exacta y re-muestreo |
| 3 / 3.1 / 3.2 | Entrenamiento en los dos escenarios, tablas por prevalencia, detectores de referencia y regla C1 |
| 4 | Regla C2: mismo accuracy, distinto recall |
| 5 | Regla C3: optimizar el accuracy, y veredicto global |
| 5.1 | **Revisión exploratoria:** C2', C3', tabla de sensibilidad, umbral razonable y resumen pre-registro frente a revisión |
| 6 / 6.1.1 | Diagnósticos: re-muestreo real (umbral 0,5 y otros umbrales), prevalencia crítica y matriz por millón |
| 7 / 7.1 | Figuras y exportación de tablas a CSV |
| 8 | Guía para rellenar las conclusiones |

---

## 13. Glosario

| Término | Significado |
|---|---|
| **Accuracy** | Fracción total de aciertos; engaña cuando casi todo es de una sola clase |
| **Prevalencia** | Proporción de ataques en un conjunto de datos |
| **Falso positivo (FP)** | Conexión normal que dispara una alerta (falsa alarma) |
| **Falso negativo (FN)** | Ataque que no dispara ninguna alerta |
| **Recall / TPR** | De los ataques reales, fracción detectada |
| **FNR** | De los ataques reales, fracción que se escapa (`1 − recall`) |
| **Precisión** | De las alertas lanzadas, fracción que eran ataques |
| **FPR** | De las conexiones normales, fracción que dispara alerta |
| **Línea base** | Resultado de un detector trivial contra el que se compara el modelo |
| **Prevalencia crítica (`p*`)** | Prevalencia por debajo de la cual el detector que no hace nada saca mejor accuracy que el modelo: `FPR / (TPR + FPR)` |
| **Punto de operación** | Combinación concreta de recall y FPR que resulta de un umbral |
| **MCC** | Coeficiente de Matthews: una nota única que usa TP, TN, FP y FN |
| **Accuracy balanceado** | Media del recall y de `1 − FPR`: acierto medio por clase |
| **Alert fatigue** | Saturación del analista por exceso de falsas alarmas |
| **SOC** | Centro de operaciones de seguridad |
| **Falacia de la tasa base** | Ignorar la prevalencia al interpretar una alerta o una métrica |

---

## 14. Referencias

- Moustafa, N. y Slay, J. (2015). *UNSW-NB15: a comprehensive data set for network intrusion detection systems (UNSW-NB15 network data set)*. Military Communications and Information Systems Conference (MilCIS), IEEE.
- Axelsson, S. (2000). *The base-rate fallacy and the difficulty of intrusion detection*. ACM Transactions on Information and System Security, 3(3), 186-205.
- Saito, T. y Rehmsmeier, M. (2015). *The precision-recall plot is more informative than the ROC plot when evaluating binary classifiers on imbalanced datasets*. PLoS ONE, 10(3).
- Chicco, D. y Jurman, G. (2020). *The advantages of the Matthews correlation coefficient (MCC) over F1 score and accuracy in binary classification evaluation*. BMC Genomics, 21(1).
- Documentación de scikit-learn: `LogisticRegression`, `Pipeline`, `ColumnTransformer`.

> Las referencias de Axelsson, Saito y Rehmsmeier, y Chicco y Jurman están citadas de memoria: **comprueba los datos bibliográficos antes de publicar**.

---

*Serie «Del error al detector». Este laboratorio está en fase de pre-registro: la hipótesis puede salir refutada, y eso también forma parte del método.*
