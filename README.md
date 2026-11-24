# Del error al detector

> Aprendiendo machine learning en público, equivocándome con método, para construir un detector de intrusiones en red en el que se pueda confiar.

Este repositorio es el cuaderno de laboratorio de una serie de LinkedIn en español. Cada experimento parte de algo que *suena* razonable ("un 99 % de accuracy es un modelo excelente", "repartir los datos al azar es suficiente"), lo pone a prueba con código reproducible y cuenta honestamente qué salió: incluso cuando la hipótesis resulta ser falsa.

**El objetivo de fondo:** comparar modelos de ML para detección de intrusiones con criterios realistas (desbalanceo, coste de los errores, reparto temporal, generalización y robustez), y terminar preguntándonos qué cambia cuando el atacante es un agente de IA.

---

## Cómo funciona cada subcapítulo

Cada pregunta sigue el mismo ciclo, para que el proceso se pueda auditar:

1. **Hipótesis por escrito, antes de medir.** Y las reglas para decidir si se confirma o se refuta, también fijadas de antemano.
2. **Post 1, la intuición.** Más ligero: qué creo que va a pasar y por qué importa.
3. **Cuaderno de Jupyter reproducible.** Todo el código, parámetros a la vista y semillas fijas.
4. **Post 2, la evidencia.** Más trabajado: datos, diagnósticos y veredicto, incluida la respuesta a "¿tenía razón?".
5. **Informe completo** en el `README.md` de la carpeta del subcapítulo: método, resultados, errores cometidos y limitaciones.

> Lo que se decide *después* de ver los datos se marca como **exploratorio**, no como confirmatorio.

---

## Último resultado

**¿Reparto al azar o por tiempo?** (`Subcapitulos/Azar-Tiempo`)

La hipótesis era que recall y precisión de una regresión logística caerían al repartir los datos por tiempo en lugar de al azar. **No se confirmó**: en los pliegues válidos el recall sube de media 1,5 puntos y la precisión baja 1,1, ninguno por encima del umbral de 5 puntos fijado de antemano.

Lo que sí cambió, y no era la métrica que se estaba mirando, fueron las **falsas alarmas**: en el caso más duro (entrenar con la captura A y probar con la B), la tasa de falsos positivos pasó de 0,67 % a 2,25 % (≈ ×3,4). Con un modelo lineal, el efecto de la fuga de datos casi no se nota; el siguiente paso es repetirlo con modelos que sí memorizan.

---

## Mapa de la serie

| Cap. | Tema | Estado |
|---|---|---|
| 0 | Planteamiento y arranque de la serie | En marcha |
| 1 | Fundamentos de ML: evaluar sin engañarse | En curso |
| 2 | Anatomía de un ataque: datos de red e IDS | Planificado |
| 3 | Los tres sospechosos: regresión logística, Naive Bayes, KNN | Planificado |
| 4 | Árboles, bosques y fuerza bruta: SVM, árbol, Random Forest, Gradient Boosting | Planificado |
| 5 | Redes neuronales y anomalías: MLP, Isolation Forest, autoencoder | Planificado |
| 6 | ¿Cuánto cuesta equivocarse? Umbral basado en coste y PR-AUC | Planificado |
| 7 | Cuando los datos cambian: generalización entre datasets y evasión | Planificado |
| 8 | ¿Y los agentes? Qué no sabe ver un detector clásico | Planificado |

Cada capítulo agrupa varios **subcapítulos**: experimentos cortos con su propia pregunta, su cuaderno y sus dos posts.

---

## Subcapítulos

| Cap. | Subcapítulo | Pregunta | Estado | Post 1 | Post 2 |
|---|---|---|---|---|---|
| 1 | [Azar vs. tiempo](Subcapitulos/Azar-Tiempo/) | ¿Cuánto cambian recall y precisión al repartir por tiempo en lugar de al azar? | Completo | Pendiente de enlazar | Pendiente de enlazar |
| 1 | El espejismo del 99 % | ¿Por qué un 99 % de accuracy no implica que el modelo funcione? | En preparación | – | – |

---

## Estructura del repositorio

```
.
├── README.md                    # esta página
└── Subcapitulos/
    └── Azar-Tiempo/
        ├── README.md            # informe completo del experimento
        └── laboratorio_azar_vs_tiempo_v2.ipynb
```

Cada subcapítulo nuevo añade su propia carpeta con la misma forma: un informe y un cuaderno.

---

## Cómo reproducir un experimento

1. Entra en la carpeta del subcapítulo y lee su `README.md`: indica qué datos necesita y dónde descargarlos.
2. Instala las dependencias (Python 3.9 o superior):
   ```bash
   pip install pandas numpy "scikit-learn>=1.2" matplotlib jupyter
   ```
3. Descarga los datos aparte y colócalos junto al cuaderno. **Los datasets no se redistribuyen en este repositorio.**
4. Abre el cuaderno con `jupyter lab` y revisa la celda de configuración antes de ejecutar. Algunos cuadernos tardan decenas de minutos con todos los datos; suelen tener un parámetro (`SAMPLE_FRAC`) para hacer una prueba rápida.

---

## Principios

- **Una hipótesis refutada también es un resultado.** Forma parte del método, no es un fallo.
- **Ninguna métrica va sola.** Accuracy, recall, precisión, tasa de falsos positivos y PR-AUC cuentan historias distintas.
- **Se separa lo planeado de lo descubierto.** Lo que se añade tras ver los datos se declara como exploratorio.
- **Las limitaciones van a la vista**, no al final de un apéndice.
- **Se documentan los errores**, con qué se rompió, cómo se detectó y cómo se corrigió.

---

## Sobre el proyecto

Proyecto personal de aprendizaje de un estudiante de máster en IA con interés en ciberseguridad. Está pensado para aprender construyendo y para que quien lo lea pueda aprender también, tanto si es la primera vez que oye "data leakage" como si lleva años en el sector.

- LinkedIn: [añade aquí tu perfil](https://www.linkedin.com/)
- ¿Has encontrado un fallo en un experimento? Abre un *issue*: revisar el trabajo también forma parte del método.

---

*Serie «Del error al detector».*
