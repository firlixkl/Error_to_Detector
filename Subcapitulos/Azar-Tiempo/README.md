# ¿Reparto al azar o por tiempo? Un laboratorio sobre evaluación de detectores de intrusiones

> **Serie «Del error al detector» · Capítulo 1**
> Pregunta: *¿cuánto cambian el recall y la precisión de una regresión logística en detección de intrusiones cuando el reparto entre entrenamiento y prueba es por tiempo en lugar de al azar?*

Este repositorio contiene un cuaderno de Jupyter reproducible y la documentación completa de un experimento que **refutó la hipótesis inicial** y, de paso, mostró que la métrica que no se estaba mirando (las falsas alarmas) era la que más cambiaba.

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
| **Hipótesis (fijada antes de medir)** | Recall y precisión **bajan** con reparto por tiempo |
| **Modelo** | Regresión logística (umbral 0,5), sin ajuste de hiperparámetros |
| **Datos** | UNSW-NB15: 2.540.047 conexiones, 321.283 ataques (12,65 %), 9 tipos |
| **Veredicto sobre la hipótesis** | **No confirmada.** En los pliegues válidos el recall sube 1,5 puntos de media y la precisión baja 1,1; ninguno supera el umbral de 5 puntos fijado de antemano |
| **Hallazgo secundario (exploratorio)** | La **tasa de falsos positivos se multiplica por ≈3,4** en el caso más duro (0,67 % → 2,25 %) |
| **Por qué la precisión no lo muestra** | La proporción de ataques cambia de 12,6 % a 20,6 % y sostiene la precisión |
| **Posible explicación** | El tráfico normal de la captura B es distinguible del de la captura A (AUC = 0,82) |
| **Advertencia principal** | La regresión logística **no memoriza**; no es el mejor modelo para ver el efecto de la fuga de datos. El siguiente experimento usará un árbol de decisión |

---

## 2. Conceptos necesarios

### 2.1 Qué es un detector de intrusiones y cómo se evalúa

Un detector de intrusiones (IDS) clasifica cada conexión de red como *normal* o *ataque*. Para evaluarlo se separan los datos etiquetados en dos grupos: **entrenamiento** (con el que aprende) y **prueba** (con el que se le examina).

### 2.2 Las métricas

Sea TP los ataques detectados, FN los ataques no detectados, FP las conexiones normales marcadas como ataque y TN las normales correctamente ignoradas.

| Métrica | Fórmula | Pregunta que responde |
|---|---|---|
| **Recall** (sensibilidad) | TP / (TP + FN) | De los ataques reales, ¿cuántos capturé? |
| **Precisión** | TP / (TP + FP) | De mis alertas, ¿cuántas eran ataques? |
| **Tasa de falsos positivos (FPR)** | FP / (FP + TN) | De las conexiones normales, ¿cuántas disparan alarma? |
| **F1** | 2·TP / (2·TP + FP + FN) | Media armónica de precisión y recall |
| **Accuracy** | (TP + TN) / total | ¿Qué fracción acierto en total? |
| **PR-AUC** (average precision) | Área bajo la curva precisión-recall | ¿Ordena bien lo sospechoso, **sin depender del umbral**? |

**Por qué el accuracy engaña con datos desbalanceados.** Con 100 conexiones (99 normales, 1 ataque), un modelo que dice «todo normal» tiene un 99 % de accuracy y un recall del 0 %. En este dataset, el modelo «todo normal» sacaría entre un **79 % y un 87 %** de accuracy según la captura, un 98 % en la captura A.

**Por qué hace falta mirar recall y precisión a la vez.** Marcarlo todo como ataque da recall = 100 %, pero la precisión se hunde y el equipo del SOC (Security Operations Center) queda saturado de falsas alarmas (*alert fatigue*).

**Por qué la precisión no basta: depende de la prevalencia.** Con recall (TPR) y FPR fijos, la precisión cambia con la proporción de ataques *p*:

```
precisión = (TPR · p) / (TPR · p + FPR · (1 − p))
```

La FPR, en cambio, solo usa tráfico normal y **no depende de cuántos ataques haya**. Por eso es la métrica adecuada para comparar situaciones con proporciones de ataques distintas.

### 2.3 Qué significa «repartir por tiempo» y por qué importa

- **Reparto al azar:** se mezclan todas las conexiones y se separa una fracción para la prueba.
- **Reparto por tiempo:** se entrena con el pasado y se prueba con lo posterior, que es lo que ocurre al desplegar un detector.

Un ataque genera miles de conexiones casi idénticas. Al mezclar al azar, el examen contiene «gemelas» de lo estudiado (**fuga de datos**, *data leakage*) y la nota puede salir inflada. Un modelo también puede aprender **atajos** (por ejemplo, memorizar la IP del atacante) que no se sostienen en producción.

---

## 3. El dataset: UNSW-NB15

### 3.1 Qué es

UNSW-NB15 es un conjunto de datos de detección de intrusiones creado en el laboratorio Cyber Range de UNSW Canberra (Moustafa y Slay, 2015). Combina tráfico normal con comportamientos de ataque generados sintéticamente en un entorno de laboratorio. Cada fila es una **conexión** (flujo de red) descrita por 49 columnas, incluidas las etiquetas.

Se han usado los **cuatro CSV en bruto** (`UNSW-NB15_1.csv` ... `UNSW-NB15_4.csv`), que no traen cabecera; los nombres de columna se definen en el cuaderno.

### 3.2 Por qué este dataset

| Criterio | Motivo |
|---|---|
| **Marcas de tiempo fiables** | Trae `Stime` y `Ltime` como tiempo Unix. Sin una marca temporal fiable no se puede repartir por tiempo, que es el núcleo del experimento |
| **Incluye IP de origen y destino** | Permite comprobar explícitamente si el modelo puede hacer trampa con la IP |
| **Tamaño** | 2,5 millones de filas: suficiente para bloques temporales con miles de ataques y para medir variabilidad |
| **Varios tipos de ataque** | Nueve categorías, útiles para ver qué tipos se pierden en cada reparto |
| **Se evita un problema conocido de alternativas** | Otros datasets populares (como CIC-IDS2017) tienen problemas documentados en sus marcas de tiempo; aquí no era un riesgo |

### 3.3 Composición

| Concepto | Valor |
|---|---|
| Conexiones | **2.540.047** |
| Ataques | **321.283 (12,65 %)** |
| Tipos de ataque | Generic, Exploits, Fuzzers, DoS, Reconnaissance, Analysis, Backdoor, Shellcode, Worms |
| Variables usadas | **41** (38 numéricas + 3 categóricas: `proto`, `state`, `service`) |

### 3.4 Dos capturas, no una

Esta es la característica más importante del dataset para este experimento. Las marcas de tiempo muestran **dos capturas separadas por 624 horas (26 días)**:

| Captura | Desde | Hasta | Duración | Conexiones | Ataques | % ataques |
|---|---|---|---|---|---|---|
| **A** | 2015-01-22 11:49 | 2015-01-23 00:25 | 12,6 h | 1.087.203 | 22.215 | **2,04 %** |
| **B** | 2015-02-18 00:23 | 2015-02-18 12:21 | 11,96 h | 1.452.844 | 299.068 | **20,59 %** |

Consecuencias directas:

1. Un reparto por filas habría mezclado las dos capturas en un mismo bloque.
2. La proporción de ataques es 10 veces mayor en B. Eso afecta a la precisión (ver 2.2).
3. Todos los ataques de A están en las primeras 4,2 horas; el resto de A es tráfico 100 % normal. Esto limita cuántos pliegues son evaluables (ver 5.4).

### 3.5 El atajo de las IP

| IP de origen de ataque | % de los ataques |
|---|---|
| 175.45.176.1 | 36,7 % |
| 175.45.176.3 | 33,1 % |
| 175.45.176.0 | 23,1 % |
| 175.45.176.2 | 7,1 % |

**Solo 4 direcciones de origen generan el 100 % de los ataques.** Las mismas 4 IP también aparecen en tráfico normal (junto a 43 IP de origen distintas en total en tráfico normal). Es un artefacto de cómo se montó el banco de pruebas, no una propiedad de los ataques reales, donde los atacantes cambian de IP.

---

## 4. Preparación de los datos

Cada decisión tiene una razón.

| Paso | Qué se hace | Por qué |
|---|---|---|
| 1. Limpieza de tipos | Conversión numérica de `sport`, `dsport`, `ct_ftp_cmd`, `ct_flw_http_mthd`, `is_ftp_login`; eliminación de espacios en categorías | Estas columnas traen valores mezclados (hexadecimales, vacíos, `-`) |
| 2. Etiquetas | `attack_cat` vacío → `Normal`; `Backdoors` → `Backdoor` | Unifica categorías duplicadas por erratas |
| 3. **Orden por `Stime`** | Orden estable por marca de tiempo | Todo el experimento depende de que el orden sea el real |
| 4. **Se elimina la IP** (`srcip`, `dstip`) | Fuera del modelo base | Con 4 IP concentrando todos los ataques, un modelo que las vea aprende *quién ataca*, no *cómo es un ataque*. Un modelo **solo con IP** ya obtiene F1 ≈ 0,94 (sección 7.5) |
| 5. **Se eliminan los puertos** (`sport`, `dsport`) | Fuera del modelo base | Son identificadores de la misma familia que la IP |
| 6. **Se elimina el tiempo como variable** (`Stime`, `Ltime`) | Solo se usa para ordenar y repartir | Si fuera variable, el modelo podría aprender «los ataques ocurren en febrero» |
| 7. Etiquetas fuera | `Label`, `attack_cat` | Son lo que se quiere predecir |
| 8. Imputación | Mediana (solo con datos de entrenamiento) | Hay valores vacíos en algunas columnas |
| 9. Transformación `log1p` | `log(1 + max(x, 0))` en las numéricas | Las variables de red (bytes, paquetes) tienen colas muy largas |
| 10. Estandarización | Media 0, desviación 1 | La regresión logística es sensible a la escala |
| 11. One-hot | Para `proto`, `state`, `service`, con frecuencia mínima de 100 | Las categorías raras se agrupan en «infrecuente» |

El preprocesado se ajusta **solo con el entrenamiento** de cada reparto (todo va dentro de un `Pipeline` de scikit-learn), para no introducir otra fuga de datos.

> **Nota sobre las variables `ct_*`.** Varias columnas del dataset son estadísticas contextuales calculadas por los autores sobre ventanas de conexiones recientes. Se mantienen porque forman parte del conjunto estándar de variables.

**Modelo.** `LogisticRegression(max_iter=1000)` con valores por defecto (regularización L2, `C = 1`, sin pesos de clase). Umbral de decisión fijo en **0,5**.

---

## 5. Diseño experimental

### 5.1 Comparación pareada

Para aislar el efecto del reparto, **solo cambia el reparto**:

- mismo conjunto de datos,
- mismos tamaños de entrenamiento y de prueba,
- mismas variables, mismo modelo, mismo umbral.

### 5.2 Bloques por tiempo real

Cada captura se divide en **3 bloques de igual duración** (no de igual número de filas) y **ningún bloque cruza el hueco entre capturas**.

| Bloque | Captura | Duración | Conexiones | Ataques | % ataques |
|---|---|---|---|---|---|
| 0 | A | 4,20 h | 362.361 | 22.215 | 6,13 % |
| 1 | A | 4,20 h | 375.817 | **0** | 0,00 % |
| 2 | A | 4,20 h | 349.025 | **0** | 0,00 % |
| 3 | B | 3,99 h | 421.253 | 82.310 | 19,54 % |
| 4 | B | 3,99 h | 522.118 | 117.375 | 22,48 % |
| 5 | B | 3,99 h | 509.473 | 99.383 | 19,51 % |

### 5.3 Pliegues de origen móvil (*rolling-origin*)

Para cada pliegue *k* se entrena con los bloques `0..k` y se prueba con el bloque `k+1`:

| Pliegue | Entrenamiento | Filas de entrenamiento | Ataques en entrenamiento | Prueba | Filas de prueba | Ataques en prueba | ¿Válido? |
|---|---|---|---|---|---|---|---|
| 0 | bloque 0 | 362.361 | 22.215 | bloque 1 | 375.817 | 0 | No |
| 1 | bloques 0-1 | 738.178 | 22.215 | bloque 2 | 349.025 | 0 | No |
| **2** | bloques 0-2 (toda A) | 1.087.203 | 22.215 | bloque 3 (primero de B) | 421.253 | 82.310 | **Sí** |
| **3** | bloques 0-3 | 1.508.456 | 104.525 | bloque 4 | 522.118 | 117.375 | **Sí** |
| **4** | bloques 0-4 | 2.030.574 | 221.900 | bloque 5 | 509.473 | 99.383 | **Sí** |

Para cada pliegue, el reparto **al azar** usa las mismas filas (las primeras `test_end`) y los mismos tamaños, y se **repite 10 veces** con semillas distintas (`SEED + 1000·k + r`). Eso da una medida de cuánto varía el resultado solo por azar.

### 5.4 Reglas de decisión fijadas de antemano

| Regla | Valor |
|---|---|
| «Diferencia grande» en recall o precisión | **5 puntos porcentuales** (`BIG_DIFF = 0.05`) |
| Fracción de pliegues válidos que debe cumplirla | **75 %** (`MIN_FRAC_AGREE`) → con 3 pliegues válidos, hacen falta 3 de 3 |
| Pliegue válido | Al menos **100 ataques** en prueba **y** en entrenamiento (`MIN_TEST_ATTACKS`) |
| Métricas sin definir | Si no hay ataques en la prueba, el recall es 0/0: se marca `NaN` y el pliegue se **excluye**, no se cuenta como 0 |

**Regla exploratoria (añadida después de ver la primera ejecución):** la FPR se considera «grande» si se multiplica por 2 o más (`FPR_RATIO_BIG = 2.0`). Al fijarse tras ver datos, **no es una prueba confirmatoria**, sino una pista a verificar.

### 5.5 Caso duro: entrenar con A, probar con B

El salto temporal máximo: entrenamiento = toda la captura A (1.087.203 filas), prueba = toda la captura B (1.452.844 filas). Referencia al azar: 5 repartos aleatorios de **todos** los datos con los mismos tamaños.

### 5.6 Diagnósticos con un reparto 70/30

Un reparto único (70 % entrenamiento, 30 % prueba; 1.778.032 / 762.015 filas) por tiempo y al azar con los mismos tamaños, para las pruebas de gemelas, distancia, tipos de ataque e IP.

---

## 6. Resultados

### 6.1 Pliegues válidos (2, 3 y 4)

Media del reparto al azar (10 repeticiones) frente al valor por tiempo.

| Pliegue | Métrica | Azar (media ± desv.) | Tiempo | Δ (tiempo − azar) |
|---|---|---|---|---|
| 2 | Recall | 0,9654 ± 0,0010 | 0,9928 | **+0,0274** |
| 3 | Recall | 0,9703 ± 0,0007 | 0,9787 | **+0,0084** |
| 4 | Recall | 0,9704 ± 0,0009 | 0,9798 | **+0,0094** |
| 2 | Precisión | 0,9428 ± 0,0010 | 0,9118 | **−0,0310** |
| 3 | Precisión | 0,9539 ± 0,0006 | 0,9595 | **+0,0056** |
| 4 | Precisión | 0,9546 ± 0,0007 | 0,9469 | **−0,0077** |
| 2 | F1 | 0,9540 ± 0,0005 | 0,9506 | −0,0034 |
| 3 | F1 | 0,9620 ± 0,0004 | 0,9690 | +0,0070 |
| 4 | F1 | 0,9624 ± 0,0005 | 0,9631 | +0,0006 |
| 2 | PR-AUC | 0,9935 ± 0,0003 | 0,9924 | −0,0010 |
| 3 | PR-AUC | 0,9955 ± 0,0003 | 0,9970 | +0,0015 |
| 4 | PR-AUC | 0,9958 ± 0,0001 | 0,9958 | 0,0000 |
| 2 | Accuracy | 0,9935 ± 0,0001 | 0,9798 | −0,0137 |
| 3 | Accuracy | 0,9916 ± 0,0001 | 0,9859 | −0,0057 |
| 4 | Accuracy | 0,9904 ± 0,0002 | 0,9853 | −0,0051 |

**Medias de la diferencia (3 pliegues válidos):**

| Métrica | Δ medio | Pliegues con caída > 5 pts | Veredicto |
|---|---|---|---|
| Recall | **+0,015** | 0 / 3 | Sin cambio relevante (si acaso, sube) |
| Precisión | **−0,011** | 0 / 3 | Sin cambio relevante |
| F1 | +0,001 | 0 / 3 | Sin cambio relevante |
| PR-AUC | 0,000 | 0 / 3 | Sin cambio relevante |
| Accuracy | −0,008 | 0 / 3 | Sin cambio relevante (pero tiempo < mínimo del azar en 3/3) |

> **Hipótesis (recall y precisión bajan más de 5 puntos): no confirmada.**
> El cuaderno etiqueta este resultado como «sin cambio claro / inconsistente». La etiqueta es imprecisa: los cambios son **consistentes y pequeños** (ningún pliegue válido supera ±5 puntos en recall ni precisión).

**Variabilidad.** La desviación típica del reparto al azar es de ≈ 0,1 puntos o menos en los pliegues válidos. Las diferencias de varios puntos **no se explican solo por la suerte del reparto aleatorio**. El reparto por tiempo es una única ejecución determinista, así que **no tiene intervalo de variabilidad propio**: diferencias de décimas de punto no son interpretables.

### 6.2 Falsas alarmas (métrica exploratoria)

| Pliegue | ¿Válido? | FPR azar | FPR tiempo | Ratio tiempo/azar |
|---|---|---|---|---|
| 0 | No | 0,0030 | 0,0000 | 0,01 |
| 1 | No | 0,0020 | 0,0000 | 0,00 |
| **2** | Sí | 0,0044 | 0,0233 | **5,35** |
| **3** | Sí | 0,0058 | 0,0120 | **2,08** |
| **4** | Sí | 0,0067 | 0,0133 | **1,99** |

- **Ratio mediano en pliegues válidos: 2,08**; ratio ≥ 2 en **2 de 3** pliegues. Con la regla fijada (3 de 3), el veredicto automático es «sin cambio claro»: el pliegue 4 se queda a una centésima del umbral (1,989).
- La FPR de los pliegues 0 y 1 (prueba dentro de la captura A) es casi nula por tiempo. Esos pliegues se excluyen del veredicto de recall y precisión por no tener ataques, pero **la FPR sí es evaluable** (solo usa tráfico normal) y muestra que el salto aparece al entrar en la captura B.
- Los umbrales de la regla exploratoria se fijaron tras ver datos: son una pista, no una confirmación.

### 6.3 Accuracy frente al modelo «todo normal»

| Pliegue | Prevalencia azar | Prevalencia tiempo | Accuracy «todo normal» (azar) | Accuracy «todo normal» (tiempo) |
|---|---|---|---|---|
| 2 | 6,93 % | 19,54 % | 93,07 % | 80,46 % |
| 3 | 10,92 % | 22,48 % | 89,08 % | 77,52 % |
| 4 | 12,67 % | 19,51 % | 87,33 % | 80,49 % |

La **prevalencia de ataques en la prueba cambia mucho** entre repartos: no es un detalle, afecta a la lectura de precisión y accuracy.

### 6.4 Caso duro: entrenar con A, probar con B

| Métrica | Azar (media de 5 repeticiones) | Desv. azar | Tiempo (A → B) | Δ |
|---|---|---|---|---|
| Accuracy | 0,9905 | 0,0001 | 0,9807 | −0,0098 |
| **Precisión** | 0,9544 | 0,0005 | 0,9195 | **−0,0349** |
| **Recall** | 0,9710 | 0,0003 | 0,9932 | **+0,0222** |
| F1 | 0,9626 | 0,0002 | 0,9549 | −0,0077 |
| **FPR** | 0,0067 | 0,0001 | 0,0225 | **+0,0158 (×3,4)** |
| PR-AUC | 0,9956 | 0,0002 | 0,9943 | −0,0013 |
| Prevalencia en la prueba | 12,65 % | 0,01 % | 20,59 % | +7,94 pts |
| Accuracy «todo normal» | 87,35 % | 0,01 % | 79,41 % | −7,94 pts |

#### Matriz de confusión reconstruida (caso A → B, por tiempo)

> Reconstruida **a partir de las métricas redondeadas** (recall = 0,9932; FPR = 0,0225) y de los totales de la captura B. Son valores aproximados, no salidas directas del cuaderno.

| | Predicho ataque | Predicho normal | Total |
|---|---|---|---|
| **Real: ataque** | TP ≈ 297.034 | FN ≈ 2.034 | 299.068 |
| **Real: normal** | FP ≈ 25.960 | TN ≈ 1.127.816 | 1.153.776 |

*Comprobación de consistencia:* precisión = 297.034 / (297.034 + 25.960) ≈ 0,9196, coincide con el 0,9195 medido.

#### Qué significan las falsas alarmas en operación

| Por cada 1.000.000 de conexiones normales | Falsas alarmas |
|---|---|
| Con el rendimiento del reparto al azar (FPR = 0,67 %) | ≈ **6.700** |
| Con el rendimiento por tiempo (FPR = 2,25 %) | ≈ **22.500** |

*(Ilustración a partir de las tasas; no es una medición sobre un millón de conexiones reales.)*

#### Por qué la precisión no lo muestra

Con la misma proporción de ataques que en el reparto al azar (12,65 %), la precisión del modelo por tiempo sería:

```
precisión = (0,9932 · 0,1265) / (0,9932 · 0,1265 + 0,0225 · 0,8735) ≈ 0,865
```

Es decir, **≈ 86,5 %** frente al 92,0 % medido (con la prevalencia real de B, 20,6 %). El aumento de ataques en la captura B sostiene la precisión. *(Cálculo propio a partir de cifras redondeadas.)*

#### Una observación sobre el PR-AUC

El PR-AUC apenas cambia (0,9956 → 0,9943). Mide si el modelo **ordena** bien las conexiones sospechosas, sin depender del umbral. Una lectura coherente con los datos, que es una **hipótesis y no una conclusión demostrada**, es que el modelo no ha empeorado su capacidad de ordenar, sino que **con el umbral fijo de 0,5 está en otro punto de operación** (más recall, más falsas alarmas). Esto se investigará en el capítulo del umbral de decisión.

---

## 7. Diagnósticos: ¿por qué pasa esto?

### 7.1 Gemelas exactas

Porcentaje de filas de prueba con una fila **idéntica** (en todas las variables del modelo) en el entrenamiento, con el reparto 70/30:

| | Azar | Tiempo |
|---|---|---|
| Filas de prueba con gemela exacta | 22,5 % | 8,3 % |
| **Ataques** de prueba con gemela exacta | **80,6 %** | **17,6 %** |

La fuga de datos existe y se concentra en los **ataques**: un ataque genera muchas conexiones casi idénticas, y al mezclar al azar muchas acaban a ambos lados.

### 7.2 Gemelas aproximadas: distancia al vecino más cercano

Distancia mediana (variables numéricas estandarizadas) de cada fila de prueba a su fila de entrenamiento más cercana (muestra de 5.000 filas de prueba frente a 100.000 de entrenamiento):

| | Todas las filas | Solo ataques |
|---|---|---|
| Azar | 0,259 | **0,034** |
| Tiempo | 0,235 | **0,131** |

Los ataques del reparto al azar están **casi 4 veces más cerca** de un ejemplo conocido que los del reparto por tiempo. En el tráfico normal no hay diferencia apreciable.

### 7.3 ¿Cuánto pesan las gemelas exactas?

Evaluando el modelo del reparto al azar solo con las filas **sin** gemela exacta:

| | Azar (todo) | Azar (sin gemelas) | Tiempo |
|---|---|---|---|
| Precisión | 0,9550 | 0,8122 | 0,9475 |
| Recall | 0,9700 | 0,8626 | 0,9792 |
| F1 | 0,9625 | 0,8366 | 0,9631 |
| PR-AUC | 0,9954 | 0,9219 | 0,9960 |
| Prevalencia | 12,65 % | **3,18 %** | 19,49 % |

> **Precaución.** Esta tabla indica cuánto pesan las gemelas, pero **no es una «nota real»**. Al quitarlas, la proporción de ataques cae al 3,2 % y quedan los casos más raros. Comparar esta columna con las demás no es justo.

### 7.4 Tipos de ataque

**Caso duro A → B.** Ejemplos de cada tipo en A (entrenamiento) y B (prueba) y recall por tipo:

| Tipo | n en A | n en B | Recall por tiempo (A → B) | Recall azar (1ª repetición) |
|---|---|---|---|---|
| Generic | 7.522 | 207.959 | 0,999 | 0,999 |
| Exploits | 5.409 | 39.116 | 0,983 | 0,980 |
| Fuzzers | 5.051 | 19.195 | 0,987 | 0,710 |
| DoS | 1.167 | 15.186 | 0,973 | 0,981 |
| Reconnaissance | 1.759 | 12.228 | 0,986 | 0,976 |
| Analysis | 526 | 2.151 | 0,867 | 0,856 |
| Backdoor | 534 | 1.795 | 0,964 | 0,971 |
| Shellcode | 223 | 1.288 | 0,978 | 0,958 |
| Worms | 24 | 150 | 0,973 | 0,960 |

- **Los 9 tipos aparecen en A.** No hay un tipo de ataque «nuevo» en B que explique el aumento de falsas alarmas. El más raro en A, Worms (24 ejemplos), se detecta al 97 %.
- **Fuzzers es el tipo difícil bajo reparto al azar** (recall 0,710) y no por tiempo (0,987). También en el reparto 70/30 (0,699 al azar frente a 0,761 por tiempo). Es un patrón llamativo cuya causa **no se ha investigado**.

### 7.5 El atajo de la IP

Reparto 70/30, tres modelos:

| Modelo | Precisión (azar / tiempo) | Recall (azar / tiempo) | F1 (azar / tiempo) | FPR (azar / tiempo) |
|---|---|---|---|---|
| Base (sin IP) | 0,9550 / 0,9475 | 0,9700 / 0,9792 | 0,9625 / 0,9631 | 0,0066 / 0,0131 |
| Base + IP | 0,9558 / 0,9455 | 0,9707 / 0,9793 | 0,9632 / 0,9621 | 0,0065 / 0,0137 |
| **Solo IP** | 0,8942 / 0,8969 | 0,9881 / 0,9937 | **0,9388 / 0,9428** | 0,0169 / 0,0276 |

- Un modelo que **solo ve las IP**, sin nada del tráfico, alcanza F1 ≈ 0,94: casi como el modelo completo (0,96).
- **Añadir la IP al modelo completo no mejora nada** en este modelo lineal.
- El atajo **sobrevive** al reparto por tiempo, porque es una propiedad de todo el banco de pruebas (4 IP de origen para todos los ataques), no del azar.
- Se retira la IP del modelo base porque **no es una señal generalizable**, no porque cambie la nota en este caso.

### 7.6 ¿Cambia el tráfico normal entre capturas?

Se entrena un clasificador que intenta distinguir la captura A de la B **usando solo tráfico normal** (muestra de hasta 300.000 filas, 70/30 estratificado):

| Resultado | Valor |
|---|---|
| **AUC (A frente a B, solo tráfico normal)** | **0,821** |
| Referencia | 0,5 = indistinguibles |

Variables con mayor coeficiente (positivo → empuja hacia B; negativo → hacia A):

| Variable | Coeficiente |
|---|---|
| `sbytes` | +7,08 |
| `proto_ospf` | −5,31 |
| `Dpkts` | −4,97 |
| `dbytes` | +4,79 |
| `Spkts` | −4,76 |
| `service_ftp` | −3,86 |
| `state_INT` | +3,37 |
| `smeansz` | −3,35 |
| `state_CON` | −3,19 |
| `ct_dst_src_ltm` | +2,81 |

> **Cautela.** Las variables de bytes y paquetes están correlacionadas entre sí, así que los coeficientes individuales (con signos opuestos entre `sbytes`/`dbytes` y `Spkts`/`Dpkts`) no deben leerse de forma aislada. La conclusión sólida es el **AUC**: lo «normal» de B se distingue del de A. Qué cambia exactamente no se ha determinado (el protocolo OSPF, con coeficiente fuerte hacia A, es una pista para estudiar).

---

## 8. Interpretación

1. **La hipótesis inicial no se sostiene con este modelo y este dataset.** Recall y precisión no caen más de 5 puntos al repartir por tiempo.
2. **La fuga de datos existe, pero este modelo no la aprovecha.** El 80,6 % de los ataques tiene gemela exacta en el reparto al azar, frente al 17,6 % por tiempo. Pero una **regresión logística no está pensada para memorizar ni aprender patrones complejos**: aprende un criterio lineal, no guarda ejemplos. Por eso la nota casi no se infla. Los modelos que sí memorizan (árboles, bosques, vecinos más cercanos) deberían beneficiarse mucho más de las gemelas. **Es una predicción a comprobar**, no un resultado.
3. **Lo que sí cambia son las falsas alarmas.** Por tiempo se multiplican por ≈ 3,4 en el caso duro. Para un SOC, ese es el coste operativo real, y recall y precisión por sí solos no lo mostraban.
4. **Posible mecanismo (hipótesis).** El tráfico normal de la captura B es distinguible del de la A (AUC 0,82). Un modelo entrenado con «lo normal de enero» encuentra en febrero conexiones que no reconoce y las marca como sospechosas. Esto encaja con los datos (la FPR por tiempo es casi nula dentro de A y se dispara en B), pero **no lo demuestra**.
5. **El detector no parece peor, sino desplazado.** El PR-AUC casi no cambia mientras suben recall y FPR. Con un umbral distinto podría recuperarse el equilibrio. Esa es la pregunta del siguiente bloque sobre el umbral de decisión.
6. **La proporción de ataques cambia mucho y distorsiona las comparaciones.** Por eso conviene medir la FPR, que es independiente de ella.

---

## 9. Errores cometidos y cómo se corrigieron

| Versión | Error | Cómo se detectó | Corrección |
|---|---|---|---|
| Ficha del capítulo | Confundir accuracy, recall y precisión | Revisión del ejemplo «lo marca todo» | Se reescribió con ese ejemplo |
| Ficha del capítulo | Hipótesis con recall y precisión en direcciones opuestas | Al ver que las gemelas inflan ambas | Se reformuló: ambas bajan por tiempo |
| Cuaderno v1 | Con un bloque sin ataques, el recall se forzaba a **0** en vez de «sin definir» y se promediaba con los demás pliegues | Revisando la salida: un pliegue con precisión, recall y F1 igual a 0 | v2: métricas calculadas a mano que devuelven `NaN`, y pliegues no válidos excluidos del veredicto |
| Cuaderno v1 | Los bloques se definían por número de filas y uno cruzaba el hueco de 26 días entre capturas | Al ver que un bloque abarcaba del 22 de enero al 18 de febrero | v2: bloques de igual duración dentro de cada captura |
| Cuaderno v2 | La etiqueta «inconsistente» del veredicto describe mal cambios pequeños pero consistentes | Al interpretar la salida | Se documenta en 6.1 |

---

## 10. Limitaciones y amenazas a la validez

- **Un solo dataset**, generado en laboratorio con ataques sintéticos. Los resultados pueden no trasladarse al tráfico real.
- **Un solo modelo**, lineal. Es una limitación central: es el que menos aprovecha la fuga de datos.
- **Solo 3 pliegues válidos**, y todos prueban en la captura B. Los dos pliegues con prueba en la captura A no tienen ataques.
- **Los pliegues no son independientes**: los conjuntos de entrenamiento se solapan y las pruebas son bloques contiguos. Con 3 pliegues, la regla «3 de 3» es exigente y no es una prueba estadística formal.
- **El reparto por tiempo es una sola ejecución**; no hay intervalo de variabilidad. Solo la parte aleatoria se repite.
- **Un solo umbral (0,5)**; el punto de operación puede estar condicionando el resultado de falsas alarmas.
- **Dos capturas únicas con una proporción de ataques muy distinta** (2 % frente a 20,6 %): reparto por tiempo y cambio de proporción están **confundidos**. El experimento no puede separar del todo el efecto del tiempo del efecto de la proporción de ataques.
- **Doble efecto del reparto por tiempo**: quita las gemelas (fuga de datos) y a la vez introduce cambios reales entre pasado y futuro. Los diagnósticos de la sección 7 intentan separarlos, pero no lo consiguen del todo.
- **La FPR como métrica se añadió tras ver la primera ejecución**: es exploratoria.
- **Cifras reconstruidas** (matriz de confusión y precisión a igual prevalencia) calculadas con métricas redondeadas.
- **Las variables `ct_*`** son contextuales y calculadas por los autores del dataset sobre ventanas de conexiones recientes.

---

## 11. Próximos pasos

1. **Repetir con un árbol de decisión** (y después con modelos de conjunto). Predicción a comprobar: un modelo que memoriza debería inflarse bastante más con el reparto al azar, y la diferencia con el reparto por tiempo debería ser mayor.
2. **Estudiar el umbral de decisión**: curvas precisión-recall y puntos de operación con FPR controlada.
3. **Investigar qué cambia en el tráfico normal** entre A y B (distribución de protocolos, estados y servicios).
4. **Entender el caso de Fuzzers** (recall 0,71 al azar y 0,99 por tiempo en el caso duro).
5. **Probar un segundo dataset** con ataques repartidos de forma más homogénea en el tiempo.
6. **Mejorar la etiqueta de veredicto** del cuaderno y contar los pliegues 0 y 1 para la FPR.

---

## 12. Reproducción

### Estructura del repositorio

```
.
├── README.md
├── laboratorio_azar_vs_tiempo_v2.ipynb
├── requirements.txt            # opcional
├── .gitignore                  # incluir UNSW-NB15/
└── UNSW-NB15/                  # NO se sube: se descarga aparte
    ├── UNSW-NB15_1.csv
    ├── UNSW-NB15_2.csv
    ├── UNSW-NB15_3.csv
    └── UNSW-NB15_4.csv
```

### Requisitos

- Python 3.9 o superior
- `pandas`, `numpy`, `scikit-learn >= 1.2`, `matplotlib`, `jupyter`

```bash
pip install pandas numpy "scikit-learn>=1.2" matplotlib jupyter
```

### Pasos

1. Descarga los cuatro CSV en bruto de UNSW-NB15 desde la página oficial de UNSW Canberra y colócalos en `UNSW-NB15/` junto al cuaderno. Los datos **no** se redistribuyen en este repositorio.
2. Abre el cuaderno desde esa misma carpeta (las rutas son relativas):
   ```bash
   jupyter lab
   ```
3. Revisa la configuración de la primera celda **antes de ejecutar**:

| Parámetro | Valor usado | Significado |
|---|---|---|
| `SAMPLE_FRAC` | `1.0` | Fracción de datos (usa `0.25` para una prueba rápida) |
| `BLOCKS_PER_CAPTURE` | `3` | Bloques por captura |
| `N_RANDOM_REPEATS` | `10` | Repeticiones del reparto al azar por pliegue |
| `N_RANDOM_REPEATS_HARD` | `5` | Repeticiones en el caso duro A → B |
| `THRESHOLD` | `0.5` | Umbral de decisión |
| `BIG_DIFF` | `0.05` | «Diferencia grande» (5 puntos) |
| `MIN_FRAC_AGREE` | `0.75` | Fracción de pliegues válidos requerida |
| `MIN_TEST_ATTACKS` | `100` | Ataques mínimos para que un pliegue sea válido |
| `FPR_RATIO_BIG` | `2.0` | Umbral exploratorio de la FPR |
| `SEED` | `42` | Semilla |

4. Ejecuta todas las celdas. Con `SAMPLE_FRAC = 1.0` el cuaderno tarda **varias decenas de minutos** (la ejecución documentada aquí usó todos los datos).

### Estructura del cuaderno

| Sección | Contenido |
|---|---|
| 1 / 1.1 | Carga, limpieza, detección de las dos capturas y bloques temporales |
| 2 | Modelo, métricas (con «sin definir») y repartos |
| 3 / 3.1 / 3.2 | Experimento principal en 5 pliegues, tablas por métrica, veredicto y trampa del accuracy |
| 4 / 4.1 / 4.2 | Caso duro A → B, tipos de ataque, tráfico normal entre capturas |
| 5.1–5.5 | Diagnósticos 70/30: gemelas, distancias, sin gemelas, tipos de ataque, IP |
| 6 | Conclusiones (guía para rellenar) |

---

## 13. Glosario

| Término | Significado |
|---|---|
| **IDS** | Sistema de detección de intrusiones |
| **SOC** | Centro de operaciones de seguridad: el equipo que revisa las alertas |
| **Fuga de datos** (*data leakage*) | Información de la prueba que «se cuela» en el entrenamiento y hace que la nota salga inflada |
| **Gemelas** | Filas idénticas o casi idénticas en entrenamiento y prueba |
| **Atajo** (*shortcut*) | Señal que predice bien en el banco de pruebas pero no en la realidad (p. ej. la IP) |
| **Deriva** (*drift*) | Cambio real en los datos entre pasado y futuro |
| **Prevalencia** | Proporción de ataques en un conjunto de datos |
| **Umbral de decisión** | Probabilidad a partir de la cual el modelo da la alerta (aquí 0,5) |
| **Pliegue de origen móvil** | Entrenar con todo el pasado y probar con el bloque siguiente, avanzando en el tiempo |
| **Punto de operación** | Combinación concreta de recall y FPR que resulta de un umbral |
| **AUC** | Área bajo la curva ROC: 0,5 = indistinguible, 1 = perfectamente separable |
| **Alert fatigue** | Saturación del analista por exceso de falsas alarmas |

---

## 14. Referencias

- Moustafa, N. y Slay, J. (2015). *UNSW-NB15: a comprehensive data set for network intrusion detection systems (UNSW-NB15 network data set)*. Military Communications and Information Systems Conference (MilCIS), IEEE.
- Documentación de scikit-learn: `LogisticRegression`, `Pipeline`, `ColumnTransformer`, `average_precision_score`.

---

*Serie «Del error al detector». Este laboratorio documenta una hipótesis refutada: forma parte del método, no de un fallo.*
