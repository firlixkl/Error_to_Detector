# Capítulo 0 · Antes de empezar: el planteamiento

> **Serie «Del error al detector»**
> No he ni empezado y ya me he equivocado: confundí la precisión con el recall. Buen sitio para arrancar.

Este capítulo no tiene experimentos ni modelos. Es el **contrato de la serie**: qué voy a hacer, desde dónde parto, cómo lo voy a hacer y qué puedes esperar (y qué no) de este repositorio.

Volver a la [portada del repositorio](../../README.md) · Ir al primer experimento: [Azar vs. tiempo](../../Subcapitulos/Azar-Tiempo/)

---

## Índice

1. [El post de arranque](#1-el-post-de-arranque)
2. [Mi primer error (antes de empezar)](#2-mi-primer-error-antes-de-empezar)
3. [Punto de partida](#3-punto-de-partida)
4. [La pregunta de fondo](#4-la-pregunta-de-fondo)
5. [Cómo funciona la serie](#5-cómo-funciona-la-serie)
6. [Mapa de capítulos](#6-mapa-de-capítulos)
7. [Los 10 modelos candidatos](#7-los-10-modelos-candidatos)
8. [Qué es esto y qué no es](#8-qué-es-esto-y-qué-no-es)
9. [Herramientas y datos](#9-herramientas-y-datos)
10. [Mini glosario](#10-mini-glosario)
11. [Cómo seguir la serie y participar](#11-cómo-seguir-la-serie-y-participar)

---

## 1. Mi primer error (antes de empezar)

El título de la serie no es un adorno. El primer tropiezo llegó al preparar el propio proyecto: **confundí la precisión con el recall**. Son dos preguntas distintas que suenan parecidas:

| Métrica | Pregunta que responde | Ejemplo en un detector de intrusiones |
|---|---|---|
| **Recall** (sensibilidad) | De los ataques reales, ¿cuántos capturé? | Si hubo 100 ataques y detecté 90, el recall es 0,90 |
| **Precisión** | De mis alertas, ¿cuántas eran ataques de verdad? | Si lancé 200 alertas y 90 eran ataques, la precisión es 0,45 |

La forma más rápida de ver que no son lo mismo: un detector que **marca todo como ataque** tiene un recall del 100 % (no se le escapa nada) y una precisión desastrosa (casi todo son falsas alarmas). Un detector que **marca todo como normal** tiene accuracy altísimo en datos desbalanceados y un recall del 0 %.

Este error quedó registrado en el historial de errores del primer experimento ([ver tabla de errores](../../Subcapitulos/Azar-Tiempo/README.md#9-errores-cometidos-y-cómo-se-corrigieron)). Esa es la idea de la serie: **anotar los errores, no esconderlos**.

---

## 2. Punto de partida

Para que quien lea sepa desde dónde se escribe todo esto:

- **Machine learning y deep learning:** base muy baja. Estoy cursando un máster de IA y aprendiendo los modelos supervisados clásicos (regresión lineal y logística, Naive Bayes, KNN, SVM) a la vez que construyo esta serie.
- **Ciberseguridad:** nivel inicial. Tengo base de redes y mis primeros pasos en seguridad ofensiva (los *Starting Point* de Hack The Box). Estoy aprendiendo el lado defensivo mientras avanzo.
- **Herramientas:** Python, Jupyter y GitHub ya listos.
- **Tiempo disponible:** menos de 5 horas semanales, así que cada capítulo está pensado para abarcarse en unas 8-10 horas de trabajo real.

Nada de esto es una disculpa. Es la razón de ser de la serie: dejar un mapa de cómo se aprende esto desde cero, con sus tropiezos.

---

## 3. La pregunta de fondo

> **¿Qué modelos de machine learning detectan mejor las intrusiones en red cuando se evalúan con criterios realistas, y qué ocurre cuando el atacante deja de ser predecible?**

"Mejor" no existe en abstracto. Existe el mejor modelo *para una tarea, con unos datos y unas métricas concretas*. Por eso la comparativa de modelos se evalúa con:

- **Desbalanceo:** casi todo el tráfico es benigno.
- **Coste asimétrico:** un ataque no detectado y una falsa alarma no cuestan lo mismo.
- **Reparto temporal:** entrenar con el pasado, probar con el futuro.
- **Generalización:** qué pasa al probar en un dataset distinto.
- **Robustez:** qué pasa si el atacante modifica ligeramente su comportamiento.

El último punto es el puente hacia el capítulo final, el de los atacantes agénticos.

---

## 4. Cómo funciona la serie

### Ritmo

Una publicación **bisemanal** en LinkedIn con el resumen del avance, y el desarrollo completo en este repositorio, con sus cuadernos de Jupyter. De vez en cuando, posts más breves y teóricos para enlazar unas publicaciones con otras (cuando el tiempo lo permita).

### Capítulos y subcapítulos

La serie se divide en **capítulos**, cada uno con un enfoque propio para armar el puzle del machine learning. Dentro de cada capítulo hay **subcapítulos**: experimentos cortos con una pregunta concreta, un cuaderno reproducible y dos publicaciones.

```
Serie
└── Capítulo (un gran bloque temático)
    └── Subcapítulo (un experimento con una pregunta)
        ├── Post 1: la intuición
        ├── Cuaderno + informe en este repositorio
        └── Post 2: la evidencia y el veredicto
```

### El ciclo de cada subcapítulo

1. **Hipótesis por escrito antes de medir**, con las reglas para decidir si se confirma o se refuta.
2. **Post 1:** más ligero. Qué creo que va a pasar y por qué importa.
3. **Cuaderno reproducible:** código, parámetros a la vista, semillas fijas.
4. **Post 2:** más trabajado. Datos, diagnósticos y veredicto, incluida la respuesta a "¿tenía razón?".
5. **Informe completo** en la carpeta del subcapítulo: método, resultados, errores y limitaciones.

### Principios

- **Una hipótesis refutada también es un resultado.**
- **Ninguna métrica va sola:** accuracy, recall, precisión, falsos positivos y PR-AUC cuentan historias distintas.
- **Lo planeado se separa de lo descubierto:** lo que se añade tras ver los datos se marca como exploratorio.
- **Las limitaciones están a la vista**, no escondidas en un apéndice.
- **Los errores se documentan:** qué se rompió, cómo se detectó y cómo se corrigió.

---

## 5. Mapa de capítulos

Son **8 capítulos principales (en principio)** más este capítulo 0. El plan puede cambiar si algo se complica; si un capítulo crece demasiado, se divide.

| Cap. | Título | Pregunta que guía el capítulo | Qué se aprende |
|---|---|---|---|
| **0** | Antes de empezar | ¿Qué voy a hacer y cómo? | El planteamiento, el método y las reglas del juego |
| **1** | Fundamentos: evaluar sin engañarse | ¿Cómo sé si un modelo funciona de verdad? | Qué es aprender de datos, entrenamiento y prueba, sobreajuste, fuga de datos, accuracy, precisión, recall y F1 |
| **2** | Anatomía de un ataque | ¿Qué ve realmente la red cuando alguien ataca? | Qué es un IDS y un flujo de red, tipos de ataque, desbalanceo y límites de los datasets |
| **3** | Los tres sospechosos | ¿Hasta dónde llegan los modelos más simples? | Regresión logística, Naive Bayes y KNN; un *pipeline* reproducible y sin fuga de datos |
| **4** | Árboles, bosques y fuerza bruta | ¿Cuánto mejora la complejidad y a qué precio? | SVM, árbol de decisión, Random Forest y Gradient Boosting; tiempo de entrenamiento y de predicción |
| **5** | Redes neuronales y anomalías | ¿Es mejor detectar lo conocido o lo raro? | MLP, Isolation Forest y autoencoder; aprendizaje supervisado frente a detección de anomalías |
| **6** | ¿Cuánto cuesta equivocarse? | ¿Cambia el ranking de modelos si importa el coste de los errores? | Función de coste, umbral de decisión, PR-AUC, recall por tipo de ataque y reparto temporal |
| **7** | Cuando los datos cambian | ¿Qué pasa cuando entrenas en un sitio y pruebas en otro? | Generalización entre datasets, cambio de distribución y evasión básica |
| **8** | ¿Y los agentes? | ¿Qué no sabe ver un detector clásico cuando el atacante es agéntico? | Qué cambia con ataques que se adaptan y encadenan acciones, por qué los datasets clásicos no lo capturan y qué haría falta para medirlo |

### Estado actual

| Cap. | Estado |
|---|---|
| 0 | Este documento |
| 1 | **En curso:** [Azar vs. tiempo](../../Subcapitulos/Azar-Tiempo/) completo; "El espejismo del 99 %" en preparación |
| 2-8 | Planificados |

---

## 6. Los 10 modelos candidatos

La comparativa final cubre estos modelos, repartidos entre los capítulos 3, 4 y 5:

| # | Modelo | Familia | Capítulo |
|---|---|---|---|
| 1 | Regresión logística | Lineal | 3 |
| 2 | Naive Bayes | Probabilístico | 3 |
| 3 | KNN | Basado en vecinos | 3 |
| 4 | SVM | Márgenes | 4 |
| 5 | Árbol de decisión | Árboles | 4 |
| 6 | Random Forest | Ensamblado | 4 |
| 7 | Gradient Boosting (XGBoost o LightGBM) | Ensamblado | 4 |
| 8 | MLP | Red neuronal | 5 |
| 9 | Isolation Forest | Anomalías | 5 |
| 10 | Autoencoder | Anomalías con deep learning | 5 |

Para cada uno se documenta qué asume, cuándo falla y cuánto tarda en entrenar y en predecir, porque en seguridad la latencia importa.

---

## 7. Qué es esto y qué no es

**Es:**
- Un diario de aprendizaje público, con código reproducible.
- Una comparativa de modelos con criterios de evaluación realistas.
- Un lugar donde los fallos y las hipótesis refutadas tienen sitio.

**No es:**
- Un tutorial escrito por un experto: está escrito por alguien que aprende mientras lo hace.
- Un ranking definitivo de "los mejores modelos de IA para ciberseguridad".
- Una prueba de que algo funcionaría en producción. Los datasets públicos son de laboratorio y tienen limitaciones que se señalan en cada informe.
- Un proyecto de modelos de lenguaje o agentes: el ángulo agéntico se trata en el último capítulo como **análisis de limitaciones**, no como un detector de agentes.

---

## 8. Herramientas y datos

- **Lenguaje y entorno:** Python 3.9 o superior, Jupyter.
- **Librerías habituales:** `pandas`, `numpy`, `scikit-learn`, `matplotlib`.
- **Dataset principal por ahora:** UNSW-NB15, elegido por sus marcas de tiempo fiables (imprescindibles para repartir por tiempo) y porque incluye IP de origen y destino, lo que permite comprobar si un modelo hace trampa con ellas. El detalle completo está en el [informe de Azar vs. tiempo](../../Subcapitulos/Azar-Tiempo/README.md#3-el-dataset-unsw-nb15).
- **Los datos no se redistribuyen** en este repositorio: cada informe indica dónde descargarlos.

---

## 9. Mini glosario

| Término | Significado |
|---|---|
| **Machine learning (ML)** | Que un programa aprenda patrones a partir de ejemplos en lugar de seguir reglas escritas a mano |
| **IDS** | Sistema de detección de intrusiones: clasifica cada conexión de red como normal o ataque |
| **Recall** | De los ataques reales, qué fracción se detecta |
| **Precisión** | De las alertas lanzadas, qué fracción eran ataques |
| **Falsa alarma (falso positivo)** | Una conexión normal que dispara una alerta |
| **Accuracy** | Fracción total de aciertos; engaña cuando casi todo es de una sola clase |
| **Fuga de datos** (*data leakage*) | Información de la prueba que se cuela en el entrenamiento e infla la nota |
| **Ataque agéntico** | Ataque dirigido por un agente de IA que planifica, ejecuta acciones por su cuenta y adapta su comportamiento según lo que encuentra |

El glosario completo de cada experimento está en su propio informe.

---

## 10. Cómo seguir la serie y participar

- **LinkedIn:** busca el hashtag **#Del_Error_Al_Detector** o sigue mi perfil ([añade aquí tu enlace](https://www.linkedin.com/)).
- **GitHub:** pulsa *Watch* o *Star* en el repositorio para enterarte de los nuevos subcapítulos.
- **Consejos y nuevos enfoques:** son bienvenidos. Si ves un fallo en un experimento, una métrica mal interpretada o una forma más sencilla de hacerlo, abre un *issue* o escríbeme. Revisar el trabajo también forma parte del método.

---

**Siguiente:** [Capítulo 1 · Fundamentos: evaluar sin engañarse](../../Subcapitulos/Azar-Tiempo/)

*Serie «Del error al detector».*
