# Clasificador de tráfico de red

Clasificación de tráfico IT por **tipo de ataque** sobre el dataset UNSW-NB15.

El objetivo no es sacar buena nota global. Es enseñar por qué una buena nota
global puede esconder que el modelo no detecta ni un solo ataque de los tipos
que aparecen poco — que suelen ser justo los que más importan.

## Los datos

**UNSW-NB15**, del Australian Centre for Cyber Security. En Kaggle está subido
varias veces; busca `UNSW-NB15` y descarga los archivos ya divididos:

- `UNSW_NB15_training-set.csv` — ~175 000 filas
- `UNSW_NB15_testing-set.csv` — ~82 000 filas

Déjalos en `datos/`. Esa carpeta está en `.gitignore`: **el dataset no se sube
al repositorio**, solo el código que lo usa.

### Por qué este y no otro

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

python src/explorar.py        # paso 1: qué hay en los datos
```

## Estructura

```
datos/          los CSV descargados (no se suben)
src/
  explorar.py   paso 1 — entender el desequilibrio
  preparar.py   paso 2 — limpieza y división estratificada
resultados/     gráficos y tablas que genera el código
```

## Estado

- [x] Esqueleto del proyecto
- [ ] Paso 1 — exploración
- [ ] Paso 2 — preparación
- [ ] Paso 3 — baseline
- [ ] Paso 4 — modelo
- [ ] Paso 5 — tratar el desequilibrio
- [ ] Paso 6 — comparativa final

## Resultados

Se rellena al terminar. La tabla que importa es la de recall por clase, no la
del accuracy global.
