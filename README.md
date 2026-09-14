# Clasificador de tráfico de red

Clasificación de tráfico IT por **tipo de ataque** sobre el dataset UNSW-NB15.

El objetivo no es sacar buena nota global. Es mostrar por qué una buena nota
global puede esconder que el modelo no detecta ni un solo ataque de los tipos que
aparecen poco — que suelen ser justo los que más importan.

## Los datos

**UNSW-NB15**, del Australian Centre for Cyber Security (UNSW Canberra).

**Descarga usada en este proyecto:**
<https://www.kaggle.com/datasets/dhoogla/unswnb15>

Es la redistribución en Parquet de la división oficial del dataset. Se descargan
los dos archivos ya divididos:

| Archivo | Filas |
|---|---|
| `UNSW_NB15_training-set.parquet` | 175 341 |
| `UNSW_NB15_testing-set.parquet` | 82 332 |

El cuaderno acepta tanto `.parquet` como `.csv`, por si se usa otra de las copias
que hay en Kaggle. Parquet tiene preferencia: ocupa la mitad y conserva el tipo
de cada columna.

Los archivos se dejan **tal cual, sin renombrar**, en `datos/`. Esa carpeta está
en `.gitignore`: **el dataset no se sube al repositorio**, solo el código que lo
usa.

> **Nota de licencia.** UNSW-NB15 es de la UNSW y su uso es académico, con
> citación de los artículos de Moustafa & Slay. La copia de Kaggle es una
> redistribución: revisar las condiciones de ambas antes de publicar nada
> derivado.

### Por qué este dataset y no otro

| Dataset | Por qué no |
|---|---|
| NSL-KDD | Características de 1999. Sirve para aprender, pero está desfasado |
| CICIDS2017 | Realista, pero son varios GB en 8 CSV y viene bastante sucio |
| **UNSW-NB15** | Moderno, tamaño manejable, ya viene dividido y documentado |

## Puesta en marcha

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
pip install -r requirements.txt

jupyter lab notebooks/        # o abrir el cuaderno desde VS Code
```

## Estructura

```
datos/
  procesado/    conjuntos ya preparados que genera el cuaderno
notebooks/
  01-exploracion-y-preparacion.ipynb    exploración y preparación
  02-codificacion-y-baseline.ipynb      codificación, métricas y primer modelo
  03-desequilibrio.ipynb                técnicas contra el desequilibrio
  04-techo-irreducible.ipynb            qué error es culpa de los datos
  05-boosting-y-uno-contra-resto.ipynb  intentos de aprovechar el margen
resultados/     gráficos que generan los cuadernos
```

El trabajo se lleva en cuadernos y no en scripts porque la finalidad de este
proyecto es **explicar el razonamiento**, no solo producir un modelo. Cada
decisión queda escrita junto al código que la aplica.

### Rutas

**No hay ninguna ruta absoluta en el código.** Ni `C:\Users\...`, ni nombres de
usuario, ni la posición de la carpeta en el disco. Los dos cuadernos empiezan
calculando la raíz del proyecto a partir de dónde se estén ejecutando:

```python
from pathlib import Path

# Funciona tanto si el cuaderno se lanza desde notebooks/ como desde la raíz.
RAIZ = Path.cwd().parent if Path.cwd().name == "notebooks" else Path.cwd()
DATOS = RAIZ / "datos"
RESULTADOS = RAIZ / "resultados"
```

A partir de ahí todo se construye con el operador `/` de `pathlib`, que **elige
solo el separador correcto** (`\` en Windows, `/` en Linux y macOS). Por eso el
proyecto se puede clonar en cualquier carpeta, de cualquier equipo, y funciona
sin tocar una línea.

El motivo del `if`: Jupyter Lab arranca con el directorio de trabajo en
`notebooks/`, mientras que VS Code suele usar la raíz del proyecto. La condición
cubre los dos casos.

---

## Lo que ha aparecido por el camino

Esta sección recoge las complicaciones reales del dataset. Son el contenido del
proyecto, no obstáculos que haya que esconder.

### 1. El desequilibrio no está donde parecía

La intuición inicial era que los ataques serían una minoría frente al tráfico
normal. Es falso: **los ataques son el 68 %** del conjunto de entrenamiento.

El desequilibrio real está **entre los tipos de ataque**:

| Clase | Filas | % |
|---|---|---|
| Normal | 56 000 | 31,9 |
| Generic | 40 000 | 22,8 |
| Exploits | 33 393 | 19,0 |
| … | | |
| Shellcode | 1 133 | 0,65 |
| **Worms** | **130** | **0,07** |

**431 a 1** entre la clase más común y la más rara. Con 130 ejemplos de `Worms`,
un modelo puede ignorarlos por completo sin que el accuracy se resienta.

### 2. Casi la mitad del entrenamiento son filas repetidas

**78 519 duplicados de 175 341 filas (44,8 %).** Y no están repartidos:

| Clase | % duplicadas | Filas que quedan |
|---|---|---|
| **Generic** | **98,0** | 40 000 → **1 800** |
| DoS | 79,9 | 12 264 → 3 369 |
| Analysis | 54,8 | 2 000 → 1 119 |
| Worms | 10,8 | 130 → 123 |

`Generic` pasa de ser la segunda clase más grande a la sexta. **Eliminar
duplicados no es limpieza, cambia el problema.**

Tiene explicación: UNSW-NB15 se generó con un simulador de tráfico, y los ataques
`Generic` son repeticiones casi idénticas del mismo patrón.

### 3. La división oficial ya viene con fuga de datos

El hallazgo más serio. **20 851 filas del conjunto de prueba (25,3 %) aparecen
idénticas en el de entrenamiento.**

| Clase | % del test que ya estaba en train |
|---|---|
| DoS | 53,8 |
| Generic | 48,7 |
| Reconnaissance | 44,0 |

Un modelo evaluado sobre esa división **aprueba el examen porque ya vio las
respuestas**. Cualquier métrica publicada sobre el test oficial de UNSW-NB15
está inflada, y hay bastantes cuadernos en Kaggle que lo hacen.

### Cómo se ha resuelto

- Se entrena con el conjunto **sin duplicados**: 96 822 filas.
- Se evalúa **dos veces**:
  - `test_oficial` (82 332 filas) — comparable con lo que publica todo el mundo.
  - `test_limpio` (47 716 filas) — sin fuga. **Es el número honesto.**

**La diferencia entre ambos resultados es uno de los resultados del proyecto.**

### 4. Dos categorías de `state` solo existen en el conjunto de prueba

`ACC` y `CLO` aparecen en prueba y nunca en entrenamiento. Afectan a **5 filas**,
así que no mueven ninguna métrica — y por eso son peligrosas. Un codificador que
no contemple valores desconocidos no da un número malo: **lanza una excepción**.
Con 5 filas entre 47 716 el fallo pasa desapercibido en desarrollo y aparece el
día que el modelo vea tráfico real.

Resuelto con `OrdinalEncoder(handle_unknown="use_encoded_value", unknown_value=-1)`.

### 5. Las etiquetas del dataset se contradicen entre sí

El hallazgo que no estaba previsto, y salió de una anomalía en los resultados.

Al medir el coste de la fuga, todas las métricas salieron **mejores** en el
conjunto limpio (accuracy 0,753 → 0,774; f1_macro 0,457 → 0,477). Eso es lo
contrario de lo esperado: si el conjunto oficial contiene filas ya vistas en
entrenamiento, quitarlas debería *bajar* la puntuación.

Investigando por qué apareció esto:

**1 397 vectores de características de entrenamiento llevan más de una etiqueta**
— valores idénticos en las 34 columnas, clasificados como ataques distintos. Son
6 121 filas. De las 20 851 filas filtradas, **2 255 (10,8 %) tienen una etiqueta
que contradice todo lo que el modelo pudo aprender para ese vector**:

| Clase | % de sus filas filtradas contradictorias | Recall al limpiar |
|---|---|---|
| DoS | **38,6 %** | +0,080 |
| Exploits | **37,7 %** | +0,199 |
| Analysis | 30,2 % | −0,000 |
| **Generic** | **0,3 %** | **−0,385** |

La fuga hace **dos cosas opuestas a la vez**: regala aciertos donde los
duplicados son consistentes (`Generic`) y regala fallos donde se contradicen
(`DoS`, `Exploits`). Sobre la división oficial ambos efectos se cancelan en parte
y el resultado agregado parece razonable.

**Una métrica publicada sobre la división oficial de UNSW-NB15 no es solo
optimista: es incomparable**, porque mezcla puntuación regalada con puntuación
imposible. Y mientras existan vectores idénticos con etiquetas distintas hay un
techo de acierto que ningún modelo puede superar.

### Lo que queda por resolver

- En la prueba limpia, `Worms` se queda con **41 ejemplos**. Cada acierto o fallo
  mueve su recall 2,4 puntos. Hay que advertirlo al reportar.
- Falta **cuantificar el techo** que imponen las etiquetas contradictorias, para
  saber qué parte del error es irreducible.

## Estado

- [x] Esqueleto del proyecto
- [x] Paso 1 — exploración
- [x] Paso 2 — limpieza, duplicados y división honesta
- [x] Paso 3 — codificación, métricas y baseline
- [x] Paso 4 — tratar el desequilibrio (`class_weight`, SMOTE, submuestreo)
- [x] Paso 5 — cuantificar el techo de las etiquetas contradictorias
- [x] Paso 6 — *boosting* y uno-contra-resto
- [ ] Paso 7 — comparativa final y cierre del README

## Resultados hasta ahora

Todo medido sobre `test_limpio` (47 716 filas, sin fuga).

| Modelo | accuracy | f1_macro |
|---|---|---|
| Dummy — responde siempre «Normal» | 0,648 | 0,079 |
| Regresión logística | 0,602 | 0,310 |
| **Random Forest (200 árboles)** | **0,774** | **0,477** |

**Un modelo que no mira ni una sola característica saca 0,648 de accuracy.** Esa
es la razón por la que la métrica principal de este proyecto es el f1_macro, no
el accuracy.

`Exploits` es el sumidero del baseline: cinco de las diez clases lo tienen como
error principal. Cuando el bosque duda, responde la clase de ataque con más
ejemplos. **`Worms` acierta 3 de 41** conviviendo con un accuracy global de 0,774
— que es exactamente lo que este proyecto quería demostrar.

Decisiones y gráficos en `notebooks/02-codificacion-y-baseline.ipynb`.

### Tras tratar el desequilibrio

Se probaron siete estrategias eligiendo sobre una **partición de validación**, no
sobre la prueba. Ganó el **submuestreo con tope de 5 000 filas por clase**.

| Métrica | RF base | Tope 5 000 |
|---|---|---|
| accuracy | 0,774 | 0,662 |
| **f1_macro** | 0,477 | **0,491** |
| precision_macro | 0,530 | 0,483 |
| recall_macro | 0,477 | **0,587** |

Y clase por clase, donde está la diferencia real:

| Clase | Ejemplos | f1 base | f1 con tope | |
|---|---|---|---|---|
| **Worms** | 41 | 0,118 | **0,404** | recall 0,073 → 0,439 (3 → 18 de 41) |
| **DoS** | 1 378 | 0,343 | **0,393** | recall 0,239 → 0,496 |
| Generic | 1 234 | 0,724 | 0,762 | |
| Reconnaissance | 1 912 | 0,780 | 0,792 | |
| Backdoor | 267 | 0,281 | 0,288 | |
| Analysis | 250 | 0,051 | 0,042 | **no mejora con nada** |
| Fuzzers | 4 139 | 0,443 | 0,415 | |
| Exploits | 7 280 | 0,799 | 0,758 | |
| Shellcode | 291 | 0,364 | 0,293 | |
| Normal | 30 924 | 0,871 | 0,763 | recall 0,808 → 0,622 |

### Lo que cuesta, dicho sin adornos

El f1_macro solo sube 1,4 puntos, pero eso esconde dos cambios grandes que se
cancelan en el promedio. Contado en conexiones:

| | RF base | Tope 5 000 |
|---|---|---|
| Alarmas falsas | 5 944 (19,2 % del tráfico normal) | **11 688 (37,8 %)** |
| Ataques clasificados como normales | 1 468 (8,7 %) | **233 (1,4 %)** |

**Las alarmas falsas se duplican; los ataques que pasan desapercibidos se
reducen a una sexta parte.** No es una mejora gratuita: es un intercambio, y cuál
conviene depende de si cada alerta la revisa una persona o alimenta a otro
sistema. Este proyecto elige el tope 5 000 porque su objetivo declarado es
detectar los ataques raros — pero la contrapartida queda escrita, no escondida.

### Tres cosas que no funcionaron

- **`class_weight="balanced"`**, la recomendación más repetida, mueve el f1_macro
  de 0,4889 a 0,5009. Un punto. Penalizar más los errores no da información nueva
  cuando el problema es que no hay señal suficiente.
- **SMOTE completo** pasa de 77 457 a 391 150 filas y tarda nueve veces más que el
  baseline, para quedar por debajo del submuestreo con tope. La versión moderada
  logra lo mismo en 9,5 s: si se usa SMOTE, no hace falta igualar del todo.
- **El submuestreo completo** (980 filas, el 1,3 % de los datos) es el peor en
  f1_macro… y el mejor en recall de clases raras. Esa contradicción resultó ser
  el hallazgo del cuaderno.

Detalle completo y barrido del tope en `notebooks/03-desequilibrio.ipynb`.

### ¿0,491 es bueno? El techo del dataset

Un f1_macro de 0,491 no significa nada sin saber cuál es el máximo posible. Y ese
máximo existe y se puede calcular: **donde dos filas tienen valores idénticos en
las 34 columnas pero etiquetas distintas, ningún modelo puede acertar las dos.**

El *oráculo* —un clasificador que responde la etiqueta mayoritaria de cada grupo
de filas idénticas— marca ese techo:

| | Modelo | Oráculo (techo) | Distancia |
|---|---|---|---|
| accuracy | 0,662 | 0,980 | 0,318 |
| **f1_macro** | **0,491** | **0,845** | **0,354** |

**Las etiquetas contradictorias explican como mucho 15 puntos de f1_macro. Los
otros 35 son margen de mejora real.** El modelo está al **58,1 % de su techo**.

### Qué es culpa de los datos y qué del modelo

| Clase | f1 máximo | f1 modelo | % del techo | |
|---|---|---|---|---|
| Shellcode | 0,991 | 0,293 | 29,6 % | **el mayor margen** |
| Worms | 0,976 | 0,404 | 41,4 % | |
| DoS | 0,885 | 0,393 | 44,4 % | |
| Fuzzers | 0,926 | 0,415 | 44,8 % | |
| **Analysis** | **0,379** | 0,042 | 11,1 % | **techo bajo** |
| **Backdoor** | **0,361** | 0,288 | **79,8 %** | **techo bajo, casi alcanzado** |
| Normal | 1,000 | 0,763 | 76,3 % | |
| Generic | 0,987 | 0,762 | 77,2 % | |
| Exploits | 0,980 | 0,758 | 77,3 % | |
| Reconnaissance | 0,962 | 0,792 | 82,3 % | |

Solo `Analysis` y `Backdoor` tienen techo bajo, y el motivo está medido: **el
76,8 % y el 78,7 % de sus filas** están sobre vectores que también llevan otra
etiqueta. Su recall máximo alcanzable es 0,284 y 0,277 — ni el modelo perfecto
detectaría un tercio de esos ataques.

Y `Backdoor` ya está al 79,8 % de su techo: **no es una clase que el modelo lleve
mal, es una que el dataset no permite resolver.**

### La información sí está en las características

Si el modelo manda el 78 % de los `Worms` a `Exploits`, ¿es porque las
características no distinguen un gusano de un exploit, o porque el desequilibrio
aplasta a la clase pequeña? Se comprueba entrenando clasificadores **binarios y
equilibrados**, dos clases cada vez:

| Par | ROC AUC | |
|---|---|---|
| Normal vs Exploits | 0,993 | control |
| **Worms vs Exploits** | **0,983** | con solo 123 gusanos |
| Backdoor vs Exploits | 0,977 | |
| Shellcode vs Fuzzers | 0,977 | |
| Analysis vs Exploits | 0,956 | |
| DoS vs Exploits | 0,851 | |
| **Analysis vs Backdoor** | **0,566** | **indistinguibles** |

**Un binario separa `Worms` de `Exploits` con AUC 0,983 usando 123 ejemplos.** La
señal existe; el modelo multiclase no la aprovecha porque `Exploits` tiene 157
veces más ejemplos y se lleva todas las dudas.

La única excepción real es **`Analysis vs Backdoor`: AUC 0,566, casi una moneda
al aire** — y las dos se separan bien de `Exploits` por separado. No son clases
difíciles: **son la misma región del espacio de características con dos nombres.**

Fusionarlas sube el f1_macro a 0,528, pero el techo sube más (0,933): el
porcentaje del techo alcanzado **baja** al 56,5 %. Es una corrección honesta del
etiquetado, no una mejora del modelo, y se reporta como tal.

Detalle en `notebooks/04-techo-irreducible.ipynb`.

### Intentar cobrar ese margen — y no conseguirlo

Con 35 puntos de margen identificados y la señal demostrada, el paso 6 probó las
dos vías obvias: **boosting** y **uno contra resto**.

**Boosting solo no arregla nada** (f1_macro 0,4899 en validación, peor que el
Random Forest del paso 4). **Boosting *más* reequilibrado sí**: 0,5636, cuatro
puntos por encima. No son alternativas, se suman. Subir la capacidad del modelo
—500 iteraciones, paso más corto, árboles mayores— no aporta nada.

**Uno contra resto no funciona**, y eso corrige lo que yo había propuesto al
cerrar el paso 5. Los binarios son buenos por separado —AUC 0,98, bien medido—
pero **sus probabilidades no son comparables entre sí**: cada uno se entrena con
su propio equilibrio, así que el binario de `Worms` (entrenado con un 50 % de
gusanos) reparte confianza con mucha más alegría que el de `Normal`. Tomar el
máximo entre ellas compara escalas distintas. El mejor OvR se queda en 0,5543.

### Y una sorpresa: validación prometía cuatro veces más de lo que hay

| | Validación | `test_limpio` |
|---|---|---|
| Ventaja de HGB sobre el paso 4 | **+4,1 puntos** | **+0,2 puntos** |

`X_val` es un trozo aleatorio de `train`, así que se parece mucho a lo ya visto.
`test_limpio` se construyó en el paso 2 **quitando justamente las filas presentes
en entrenamiento**: contiene las conexiones *menos* parecidas.

**Validación sirve para ordenar candidatos, no para predecir la cifra final.** El
orden sí se mantuvo —los dos HGB quedaron arriba en ambas— pero la magnitud no.

### Modelo final

`HistGradientBoostingClassifier` con `sample_weight="balanced"`. Entrena en unos
4 segundos.

| | RF tope 5 000 | **HGB + peso** |
|---|---|---|
| accuracy | 0,662 | 0,644 |
| f1_macro | 0,491 | **0,493** |
| **recall_macro** | 0,587 | **0,678** |
| % del techo | 58,1 % | **58,4 %** |
| f1 de `Worms` | 0,404 | **0,517** |
| recall de `Shellcode` | 0,663 | **0,966** |
| Alarmas falsas | 11 688 | 12 100 |
| **Ataques no detectados** | 233 | **132** |

**Por 412 alarmas falsas más (+3,5 %), el sistema deja pasar 101 ataques menos
(−43 %).** Es el mejor intercambio de todo el proyecto.

El f1_macro apenas se mueve porque, otra vez, la precisión paga la factura:
`Shellcode` detecta el 96,6 % de los casos pero con precisión 0,113, así que su
f1 *baja* (0,293 → 0,203) pese a triplicar el recall.

### El resultado que cierra el proyecto

| Paso | f1_macro | % del techo |
|---|---|---|
| 2 · Random Forest base | 0,477 | 56,5 % |
| 4 · Tras tratar el desequilibrio | 0,491 | 58,1 % |
| 6 · Tras boosting y OvR | 0,493 | **58,4 %** |

**Dos cuadernos de trabajo y el porcentaje del techo ha subido 2 puntos.** Dos
familias de modelos y ocho configuraciones llegan al mismo sitio.

Eso no es un fracaso, es información — y confirma la advertencia que el cuaderno
4 había dejado escrita: **el techo de 0,845 solo contaba duplicados exactos, y
por eso era optimista.** Las clases no se solapan solo en filas idénticas, se
solapan en regiones enteras: `Analysis` y `Backdoor` son indistinguibles entre
sí, y `DoS` vs `Exploits` se separan con AUC 0,851 frente al 0,98 de otros pares.

**El margen real es bastante menor de 35 puntos.** Cuánto exactamente no se sabe,
y medirlo requeriría estimar el solapamiento entre clases — un problema más
difícil que el original. Lo que sí se puede afirmar es que **no se debe a que el
modelo sea flojo.**

Detalle en `notebooks/05-boosting-y-uno-contra-resto.ipynb`.
