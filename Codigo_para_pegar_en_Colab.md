# Código para copiar y pegar en Colab
En Colab: **+ Código** para las celdas de CÓDIGO y **+ Texto** para las de TEXTO. Pegá el contenido de cada bloque, en orden (1 a 42). Las de código se pegan tal cual; las de texto se pegan como Markdown. **Antes de ejecutar, subí la carpeta `sprites/` (los 5 PNG) al panel de archivos de Colab.**

---
## CELDA 1 – TEXTO (pegar en una celda de texto)

````markdown
# Dinosaurio con aprendizaje por refuerzo
**Aprendizaje Automático – UTN FRC – Lic. en Inteligencia Artificial – Trabajo Práctico Alternativo (Opción B)**

Este notebook tiene dos partes que se corren en orden, de arriba hacia abajo:

| Parte | Entrega | Qué hace |
|---|---|---|
| **Parte 1** | Entrega 2 – Construcción del entorno | Programa el juego del dinosaurio y prueba que sus reglas funcionan |
| **Parte 2** | Entrega 3 – Modelado y análisis | Entrena un agente Q-Learning, analiza cómo aprende y qué pasa al cambiar la recompensa |

**Cómo usarlo:** *Entorno de ejecución → Ejecutar todas*. No hace falta instalar nada (solo usa `random`, `numpy` y `matplotlib`, que ya vienen en Colab). Para ver las imágenes del dino hace falta la carpeta `sprites/` junto al notebook (ver sección 3); sin ella se dibujan rectángulos de colores. Con semillas fijas, todos los resultados se repiten igual en cada corrida. Tarda unos 2 minutos en total, casi todo en la sección 11 (los 4 experimentos de recompensa) y la 12.

**Convención de código:** las constantes van en MAYÚSCULAS, las funciones y variables en minúsculas, y cada bloque tiene comentarios que explican *qué* hace y *por qué*.

**Resumen del problema (Entrega 1).** Un dinosaurio corre en una grilla de 5 columnas × 3 alturas y se le acercan obstáculos: cactus (en el suelo) y pájaros (en el aire). En cada paso decide si salta o no. Si choca, termina la partida. El agente tiene que aprender solo, por prueba y error, cuándo conviene saltar.
````

---
## CELDA 2 – TEXTO (pegar en una celda de texto)

````markdown
---
# PARTE 1 – ENTREGA 2: Construcción del entorno

**Consigna de la Entrega 2 (Opción B)** y dónde se cumple cada punto:

| Punto de la consigna | Sección |
|---|---|
| Implementar el entorno | 2 |
| Construir la grilla y sus elementos | 2 y 3 (dibujo de la grilla) |
| Definir los estados posibles | 2 y 4 |
| Definir las acciones | 1 y 2 |
| Implementar la función de recompensas | 1 y 2 |
| Establecer las condiciones de inicio y finalización de un episodio | 2 |
| Realizar pruebas iniciales del entorno | 5, 6 y 7 |
````

---
## CELDA 3 – TEXTO (pegar en una celda de texto)

````markdown
## 1. Parámetros y constantes

Todo lo que podamos querer cambiar después (sobre todo las recompensas) está acá arriba, en un solo lugar.
Así el resto del código no tiene números "mágicos" y la Entrega 3 puede modificar la recompensa sin tocar el entorno.
````

---
## CELDA 4 – CÓDIGO

```python
import random                      # generador de números al azar (obstáculos y agente al azar)
import numpy as np                 # arreglos numéricos (tabla Q, promedios)
import matplotlib.pyplot as plt    # gráficos
from matplotlib.patches import Rectangle  # para dibujar la grilla

# ---------- Recompensas (decisión de diseño de la Entrega 1) ----------
# paso:     se gana en cada paso que el dino sobrevive           -> premia seguir con vida
# esquivar: se gana además cuando un obstáculo pasa sin chocar   -> premia el logro principal
# choque:   se recibe al chocar y termina el episodio            -> castigo fuerte
RECOMPENSAS = {"paso": 1, "esquivar": 10, "choque": -100}

# ---------- Fin de episodio por éxito ----------
TOPE_OBSTACULOS = 30   # si esquiva 30 obstáculos cortamos el episodio (equivale a 120 pasos)

# ---------- Acciones posibles ----------
NADA, SALTAR = 0, 1
NOMBRE_ACCION = {NADA: "nada", SALTAR: "saltar"}

# ---------- Tipos de obstáculo ----------
CACTUS, PAJARO = 0, 1
NOMBRE_TIPO = {CACTUS: "cactus", PAJARO: "pájaro"}

# ---------- Geometría del dino ----------
# La "fase" resume en qué momento del salto está el dino. Cada fase tiene una altura:
#   fase 0 = en el suelo (altura 0)      fase 1 = subiendo (altura 1)
#   fase 2 = punto más alto (altura 2)   fase 3 = bajando (altura 1)
ALTURA_SEGUN_FASE = [0, 1, 2, 1]

# Alturas de la grilla que tapa cada obstáculo: el cactus solo el suelo, el pájaro las dos de arriba.
# Hay choque cuando el dino está en una de esas alturas justo cuando el obstáculo llega a su columna.
OCUPA = {CACTUS: {0}, PAJARO: {1, 2}}
```

---
## CELDA 5 – TEXTO (pegar en una celda de texto)

````markdown
## 2. El entorno

**Grilla.** 5 columnas × 3 alturas. El dino siempre está en la columna 0 y no se mueve en horizontal. Los obstáculos aparecen en la columna 4 y avanzan una columna por paso hacia él. Es el "entorno equivalente" a la grilla de 5×5 de la consigna, con dos ventajas: la altura es relevante para el juego y el espacio de estados sigue siendo chico.

**Estado.** Una terna `(fase, distancia, tipo)`:
- `fase` (0 a 3): momento del salto. Usamos la fase y no solo la altura porque con altura 1 el dino puede estar subiendo o bajando, y lo que pasa después es distinto.
- `distancia` (1 a 4): cuántas columnas le faltan al obstáculo para llegar al dino.
- `tipo` (0 = cactus, 1 = pájaro).

Esto da 4 × 4 × 2 = **32 estados**.

**Acciones.** `NADA` (0) y `SALTAR` (1). Si el dino ya está en el aire, ninguna acción tiene efecto (igual que en el juego original).

**Qué pasa en cada paso, en este orden:**
1. el dino se mueve (empieza un salto, sigue en el aire o se queda en el suelo),
2. el obstáculo avanza una columna,
3. si el obstáculo llegó a la columna 0, se fija si choca (misma altura) o si fue esquivado.

**Inicio de episodio:** dino en el suelo y un obstáculo de tipo al azar en la columna 4.
**Fin de episodio:** el dino choca (*terminado*) o esquiva 30 obstáculos (*truncado*, no es una muerte; en la Entrega 3 esta diferencia importa).

`step()` devuelve `estado, recompensa, terminado, truncado, info`, igual que Gymnasium. `reset()` devuelve solo el estado, para simplificar.
````

---
## CELDA 6 – CÓDIGO

```python
class DinoEnv:
    # Entorno del dinosaurio. Sigue la interfaz de Gymnasium: reset() y step(accion).

    def __init__(self, recompensas=RECOMPENSAS, tope=TOPE_OBSTACULOS, semilla=None):
        self.r = recompensas                  # diccionario con las recompensas (ver sección 1)
        self.tope = tope                      # obstáculos a esquivar para terminar con éxito
        self.azar = random.Random(semilla)    # generador propio: con semilla fija, los resultados se repiten

    def reset(self):
        # Empieza un episodio nuevo: dino en el suelo y un obstáculo nuevo. Devuelve el estado inicial.
        self.fase = 0
        self.esquivados = 0
        self.nuevo_obstaculo()
        return self.estado()

    def nuevo_obstaculo(self):
        # Crea un obstáculo en la columna 4 (distancia 4) con tipo al azar, 50 % cactus y 50 % pájaro.
        self.distancia = 4
        self.tipo = self.azar.choice([CACTUS, PAJARO])

    def estado(self):
        # Lo que "ve" el agente antes de decidir.
        return (self.fase, self.distancia, self.tipo)

    def step(self, accion):
        # Ejecuta un paso del juego y devuelve (estado, recompensa, terminado, truncado, info).

        # 1) Movimiento del dino
        if self.fase == 0:
            if accion == SALTAR:
                self.fase = 1                 # estaba en el suelo y salta: empieza la subida
        else:
            self.fase = (self.fase + 1) % 4   # en el aire la acción no cambia nada; tras la fase 3 vuelve a 0 (suelo)

        # 2) El obstáculo se acerca una columna
        self.distancia -= 1
        if self.distancia > 0:
            # Todavía no llegó al dino: el paso transcurre sin eventos
            return self.estado(), self.r["paso"], False, False, {"evento": "nada"}

        # 3) El obstáculo llegó a la columna 0: chocan si están a la misma altura
        if ALTURA_SEGUN_FASE[self.fase] in OCUPA[self.tipo]:
            return self.estado(), self.r["choque"], True, False, {"evento": "choque"}

        # Lo esquivó: suma el logro, aparece el siguiente obstáculo y se revisa el tope
        self.esquivados += 1
        self.nuevo_obstaculo()
        truncado = self.esquivados >= self.tope
        recompensa = self.r["paso"] + self.r["esquivar"]
        return self.estado(), recompensa, False, truncado, {"evento": "esquivó"}

    def render(self):
        # Dibuja la grilla en texto. D = dino, C = cactus, P = pájaro, X = choque, . = vacío.
        letra = "C" if self.tipo == CACTUS else "P"
        for altura in (2, 1, 0):              # de arriba hacia abajo
            fila = ""
            for col in range(5):
                dino = col == 0 and altura == ALTURA_SEGUN_FASE[self.fase]
                obstaculo = col == self.distancia and altura in OCUPA[self.tipo]
                fila += "X " if dino and obstaculo else "D " if dino else letra + " " if obstaculo else ". "
            print(fila)
        print()
```

---
## CELDA 7 – TEXTO (pegar en una celda de texto)

````markdown
## 3. La grilla dibujada (con imágenes)

Una función que dibuja un fotograma del entorno con `matplotlib` (la usamos acá y en la Parte 2 para mostrar los movimientos del agente).

**Imágenes (sprites).** El dibujo usa cinco PNG con fondo transparente que tienen que estar en una carpeta `sprites/` **al lado de este notebook**:

| Archivo | Qué se dibuja |
|---|---|
| `dino_corriendo.png` | dino en el suelo (fase 0) |
| `dino_salto.png` | dino en el aire (fases 1, 2 y 3) |
| `dino_choque.png` | dino en el momento del choque |
| `cactus.png` | obstáculo cactus |
| `pajaro.png` | obstáculo pájaro |

- **En VS Code / Jupyter local:** poné la carpeta `sprites/` junto al `.ipynb`.
- **En Colab:** abrí el ícono de carpeta de la izquierda y subí la carpeta `sprites/` (arrastrándola) antes de ejecutar.
- Si falta algún archivo, el notebook **no se rompe**: dibuja rectángulos de colores en su lugar (azul = dino, verde = cactus, naranja = pájaro, rojo = choque).
- El cuadro semitransparente detrás de cada obstáculo marca las celdas que ocupa: es la zona donde hay choque. El pájaro ocupa dos celdas (alturas 1 y 2) y el cactus una (altura 0).
````

---
## CELDA 8 – CÓDIGO

```python
import os   # para armar las rutas de los archivos de imagen

CARPETA_SPRITES = "sprites"   # carpeta con los PNG, junto al notebook
NOMBRES_SPRITES = ["dino_corriendo", "dino_salto", "dino_choque", "cactus", "pajaro"]


def cargar_sprites(carpeta=CARPETA_SPRITES):
    # Lee los 5 PNG. Devuelve un diccionario {nombre: imagen}; si un archivo falta, su valor es None
    # y dibujar_fotograma() usa rectángulos de colores en su lugar.
    sprites = {}
    for nombre in NOMBRES_SPRITES:
        try:
            sprites[nombre] = plt.imread(os.path.join(carpeta, nombre + ".png"))   # matriz RGBA con valores 0 a 1
        except FileNotFoundError:
            sprites[nombre] = None
    faltan = [n for n, img in sprites.items() if img is None]
    if faltan:
        print(f"Aviso: no se encontraron {faltan} en la carpeta '{carpeta}/'. Se dibujan rectángulos de colores.")
    else:
        print("Sprites cargados:", ", ".join(sprites))
    return sprites


SPRITES = cargar_sprites()


def poner_sprite(ax, img, cx, cy, ancho_max, alto_max, zorder=3):
    # Dibuja la imagen centrada en (cx, cy), escalada para entrar en ancho_max x alto_max sin deformarse.
    alto_px, ancho_px = img.shape[:2]
    escala = min(ancho_max / ancho_px, alto_max / alto_px)
    w, h = ancho_px * escala, alto_px * escala
    ax.imshow(img, extent=(cx - w / 2, cx + w / 2, cy - h / 2, cy + h / 2), zorder=zorder)


def dibujar_fotograma(ax, estado, choque=False, titulo=""):
    # Dibuja un estado (fase, distancia, tipo) como grilla de 5 columnas x 3 alturas en el eje ax.
    fase, distancia, tipo = estado

    # Las 15 celdas de la grilla
    for col in range(5):
        for altura in range(3):
            ax.add_patch(Rectangle((col - 0.5, altura - 0.5), 1, 1, fill=False, edgecolor="lightgray", zorder=1))

    # Obstáculo: cuadro semitransparente en las celdas que ocupa (zona de choque) + su imagen
    color_obs = "tab:green" if tipo == CACTUS else "tab:orange"
    for altura in OCUPA[tipo]:
        ax.add_patch(Rectangle((distancia - 0.5, altura - 0.5), 1, 1, color=color_obs, alpha=0.18, zorder=2))
    img_obs = SPRITES["cactus" if tipo == CACTUS else "pajaro"]
    if img_obs is not None:
        if tipo == CACTUS:
            poner_sprite(ax, img_obs, distancia, 0, 0.9, 0.95)          # una celda: el suelo
        else:
            poner_sprite(ax, img_obs, distancia, 1.5, 0.95, 1.6)        # centrado entre las alturas 1 y 2
    else:                                                               # respaldo sin imagen
        for altura in OCUPA[tipo]:
            ax.add_patch(Rectangle((distancia - 0.4, altura - 0.4), 0.8, 0.8, color=color_obs, zorder=3))

    # Dino: siempre en la columna 0, a la altura que marca su fase.
    # Imagen según la situación: choque, en el suelo (corriendo) o en el aire (salto).
    altura_dino = ALTURA_SEGUN_FASE[fase]
    nombre_dino = "dino_choque" if choque else ("dino_corriendo" if fase == 0 else "dino_salto")
    img_dino = SPRITES[nombre_dino]
    if img_dino is not None:
        poner_sprite(ax, img_dino, 0, altura_dino, 0.95, 0.95, zorder=4)
    else:                                                               # respaldo sin imagen
        ax.add_patch(Rectangle((-0.4, altura_dino - 0.4), 0.8, 0.8,
                               color="tab:red" if choque else "tab:blue", zorder=4))

    # Ejes (se fijan al final porque imshow cambia los límites)
    ax.set_xlim(-0.5, 4.5); ax.set_ylim(-0.5, 2.5); ax.set_aspect("equal")
    ax.set_xticks(range(5)); ax.set_yticks(range(3))
    ax.set_xlabel("columna"); ax.set_ylabel("altura")
    ax.set_title(titulo, fontsize=9)


# Los dos tipos de obstáculo recién aparecidos, con el dino en el suelo
fig, ejes = plt.subplots(1, 2, figsize=(10, 3.6))
dibujar_fotograma(ejes[0], (0, 4, CACTUS), titulo="Inicio con cactus (estado (0, 4, 0))")
dibujar_fotograma(ejes[1], (0, 4, PAJARO), titulo="Inicio con pájaro (estado (0, 4, 1))")
plt.tight_layout(); plt.show()
```

---
## CELDA 9 – TEXTO (pegar en una celda de texto)

````markdown
## 4. Espacio de estados y de acciones

Listamos explícitamente los 32 estados y comprobamos, jugando al azar, que **todos se pueden alcanzar**. Si algún estado fuera inalcanzable, la tabla Q de la Parte 2 tendría filas que nunca se aprenden.
También armamos la conversión estado ↔ número (0 a 31), que usará la tabla Q como índice de fila.
````

---
## CELDA 10 – CÓDIGO

```python
N_ESTADOS = 4 * 4 * 2   # fase (4) x distancia (4) x tipo (2)

def estado_a_indice(estado):
    # Convierte (fase, distancia, tipo) en un número de 0 a 31 (fila de la tabla Q).
    fase, distancia, tipo = estado
    return fase * 8 + (distancia - 1) * 2 + tipo

def indice_a_estado(i):
    # Operación inversa de estado_a_indice.
    return i // 8, (i % 8) // 2 + 1, i % 2

# Los 32 estados posibles
todos = [(f, d, t) for f in range(4) for d in range(1, 5) for t in (CACTUS, PAJARO)]
assert len(todos) == N_ESTADOS
# La conversión es biyectiva: 32 estados <-> 32 índices distintos, y es reversible
assert sorted(estado_a_indice(e) for e in todos) == list(range(N_ESTADOS))
assert all(indice_a_estado(estado_a_indice(e)) == e for e in todos)

# Jugamos 200 partidas con acciones al azar y anotamos todos los estados que aparecen
env = DinoEnv(semilla=0)
azar_exploracion = random.Random(0)
vistos = set()
for _ in range(200):
    estado, fin = env.reset(), False
    while not fin:
        vistos.add(estado)
        estado, r, terminado, truncado, info = env.step(azar_exploracion.choice([NADA, SALTAR]))
        fin = terminado or truncado

print(f"Estados posibles: {N_ESTADOS} | alcanzados jugando al azar: {len(vistos)}")
assert vistos == set(todos), "hay estados inalcanzables"

# Cuántos de esos estados permiten una decisión real: solo los que tienen al dino en el suelo (fase 0)
decisiones = [e for e in todos if e[0] == 0]
print(f"Estados donde la acción importa (dino en el suelo): {len(decisiones)} de {N_ESTADOS}")
```

---
## CELDA 11 – TEXTO (pegar en una celda de texto)

````markdown
## 5. Pruebas del entorno

Probamos cada regla por separado. Si alguna falla, `assert` corta y muestra cuál. Cada prueba verifica una regla de la Entrega 1:

- arranque en el suelo y con el obstáculo en la columna 4,
- el cactus se esquiva en el aire (saltar con el cactus a distancia 1, 2 o 3) y se choca en el suelo; saltar a distancia 4 es demasiado temprano,
- el pájaro se esquiva en el suelo y se choca si se salta a distancia 1, 2 o 3,
- en el aire la acción no cambia nada,
- los valores de recompensa coinciden con la tabla de la Entrega 1,
- la recompensa total máxima de un episodio es 420 y el tope termina con *truncado* (no con *terminado*).

`jugar()` arma una partida con el obstáculo que queramos y le pasa una lista de acciones.
````

---
## CELDA 12 – CÓDIGO

```python
def jugar(tipo, acciones, recompensas=RECOMPENSAS):
    # Arma una partida con un obstáculo de tipo dado y ejecuta la lista de acciones.
    # Devuelve el primer evento que no sea "nada" (choque o esquivó) con su recompensa.
    env = DinoEnv(recompensas, semilla=0)
    env.reset()
    env.tipo = tipo                       # forzamos el tipo para probar cada caso
    for a in acciones:
        estado, r, terminado, truncado, info = env.step(a)
        if info["evento"] != "nada":
            return info["evento"], r
    return "nada", r


# --- Arranque: en el suelo, obstáculo en la columna 4, sin esquivados
env = DinoEnv(semilla=0)
estado_inicial = env.reset()
assert estado_inicial[:2] == (0, 4) and env.esquivados == 0

# --- Cactus: si no salta, choca. Si salta con el cactus a 1, 2 o 3 columnas, lo esquiva
assert jugar(CACTUS, [NADA] * 4) == ("choque", -100)
for d in (1, 2, 3):
    acciones = [NADA] * (4 - d) + [SALTAR] + [NADA] * 4    # espera hasta distancia d y salta
    assert jugar(CACTUS, acciones) == ("esquivó", 11), d
# A distancia 4 salta demasiado temprano: cae al suelo justo cuando llega el cactus
assert jugar(CACTUS, [SALTAR] + [NADA] * 3) == ("choque", -100)

# --- Pájaro: al revés. Quedarse en el suelo lo esquiva, saltar a 1, 2 o 3 columnas lo hace chocar
assert jugar(PAJARO, [NADA] * 4) == ("esquivó", 11)
for d in (1, 2, 3):
    assert jugar(PAJARO, [NADA] * (4 - d) + [SALTAR] + [NADA] * 4) == ("choque", -100), d
# Saltar a distancia 4 también esquiva al pájaro: cuando llega, el dino ya aterrizó
assert jugar(PAJARO, [SALTAR] + [NADA] * 3) == ("esquivó", 11)

# --- En el aire la acción no cambia nada: saltar o no hacer nada da las mismas fases
fases = []
for a in (NADA, SALTAR):
    env = DinoEnv(semilla=0)
    env.reset()
    env.step(SALTAR)                      # empieza un salto
    env.step(a)                           # estando en el aire, a no debería importar
    fases.append(env.fase)
assert fases[0] == fases[1] == 2

# --- Recompensas parametrizables: el entorno usa los valores que se le pasan
assert jugar(CACTUS, [NADA] * 4, {"paso": 0, "esquivar": 0, "choque": -7}) == ("choque", -7)

# --- Tope: con una regla escrita a mano ("saltar si hay un cactus a 1 columna") esquiva todo.
#     Esto prueba que ningún obstáculo es imposible: si el agente choca, es culpa de su decisión, no del entorno.
env = DinoEnv(semilla=1)
estado, total, pasos, fin = env.reset(), 0, 0, False
while not fin:
    fase, distancia, tipo = estado
    accion = SALTAR if (fase == 0 and tipo == CACTUS and distancia == 1) else NADA
    estado, r, terminado, truncado, info = env.step(accion)
    total, pasos, fin = total + r, pasos + 1, terminado or truncado
# 30 obstáculos, 120 pasos, 120*1 + 30*10 = 420, termina por tope (truncado) y no por choque (terminado)
assert (env.esquivados, pasos, total, terminado, truncado) == (30, 120, 420, False, True)

print("Todas las pruebas pasaron")
```

---
## CELDA 13 – TEXTO (pegar en una celda de texto)

````markdown
## 6. Una partida dibujada

La regla a mano ("saltar solo si hay un cactus a 1 columna"), paso a paso, en texto y en gráficos. `D` es el dino, `C` el cactus, `P` el pájaro.
````

---
## CELDA 14 – CÓDIGO

```python
# Regla a mano: función estado -> acción. Sirve para probar el entorno y después como referencia (línea base).
def regla_a_mano(estado):
    fase, distancia, tipo = estado
    return SALTAR if (fase == 0 and tipo == CACTUS and distancia == 1) else NADA


env = DinoEnv(semilla=3)
estado = env.reset()
env.render()                                         # estado inicial en texto
estados_partida = [(estado, False, "inicio")]        # guardamos cada fotograma para dibujarlo abajo
for paso in range(8):
    accion = regla_a_mano(estado)
    estado, r, terminado, truncado, info = env.step(accion)
    print(f"Paso {paso + 1}: {NOMBRE_ACCION[accion]} -> recompensa {r} ({info['evento']})")
    env.render()
    estados_partida.append((estado, info["evento"] == "choque",
                            f"paso {paso + 1}: {NOMBRE_ACCION[accion]}"))

# Los mismos 9 fotogramas, dibujados
fig, ejes = plt.subplots(3, 3, figsize=(12, 8.5))
for ax, (est, choque, titulo) in zip(ejes.flat, estados_partida):
    dibujar_fotograma(ax, est, choque, titulo)
plt.tight_layout(); plt.show()
```

---
## CELDA 15 – TEXTO (pegar en una celda de texto)

````markdown
## 7. Líneas base: agente al azar y regla a mano

Antes de entrenar, medimos dos agentes de referencia con 1000 partidas cada uno. Son lo que va a usar la Parte 2 para comparar:
- **Al azar** (el piso): elige saltar o no con probabilidad 50 %. Cualquier agente que aprenda tiene que ganarle por mucho.
- **Regla a mano** (el techo): esquiva siempre los 30 obstáculos. Muestra que el entorno se puede resolver por completo.

Métricas: obstáculos esquivados por episodio, recompensa total por episodio y proporción de episodios que llegan al tope de 30.
`jugar_episodio()` y `evaluar()` son funciones generales: reciben una *política* (una función `estado -> acción`) y la prueban en el entorno.
````

---
## CELDA 16 – CÓDIGO

```python
def jugar_episodio(env, politica):
    # Juega un episodio completo con una política (función estado -> acción).
    # Devuelve: recompensa total, obstáculos esquivados, pasos jugados.
    estado, total, pasos, fin = env.reset(), 0, 0, False
    while not fin:
        estado, r, terminado, truncado, info = env.step(politica(estado))
        total, pasos, fin = total + r, pasos + 1, terminado or truncado
    return total, env.esquivados, pasos


def evaluar(politica, episodios=1000, semilla=42, recompensas=RECOMPENSAS):
    # Juega varios episodios y devuelve los promedios. La semilla fija permite comparar políticas
    # en las mismas condiciones y repetir los resultados.
    env = DinoEnv(recompensas, semilla=semilla)
    res = np.array([jugar_episodio(env, politica) for _ in range(episodios)])
    recompensa, esquivados, pasos = res[:, 0], res[:, 1], res[:, 2]
    return {"recompensa": recompensa.mean(),
            "esquivados": esquivados.mean(),
            "pasos": pasos.mean(),
            "tasa_tope": (esquivados == TOPE_OBSTACULOS).mean(),   # fracción de partidas perfectas
            "mejor": int(esquivados.max())}


# El agente al azar usa su propio generador, separado del del entorno
azar = random.Random(7)
politica_azar = lambda estado: azar.choice([NADA, SALTAR])

linea_base = {"azar": evaluar(politica_azar), "regla a mano": evaluar(regla_a_mano)}
for nombre, m in linea_base.items():
    print(f"{nombre:13s} | esquiva {m['esquivados']:5.2f} de 30 | recompensa {m['recompensa']:6.1f} | "
          f"llega al tope en {m['tasa_tope']:.0%} | mejor episodio: {m['mejor']} obstáculos")
```

---
## CELDA 17 – TEXTO (pegar en una celda de texto)

````markdown
### Cierre de la Parte 1
El entorno cumple todo lo de la Entrega 2: grilla, 32 estados (todos alcanzables), 2 acciones, función de recompensas, inicio y fin de episodio, y pruebas que pasan. Las líneas base muestran que el problema tiene sentido: al azar casi no esquiva nada (≈ 1 de 30) y una regla simple lo resuelve por completo (30 de 30, recompensa 420).
````

---
## CELDA 18 – TEXTO (pegar en una celda de texto)

````markdown
---
# PARTE 2 – ENTREGA 3: Modelado y análisis

Usa el entorno `DinoEnv` y las funciones de la Parte 1 (hay que haber corrido esa parte antes).

**Consigna de la Entrega 3 (Opción B)** y dónde se cumple cada punto:

| Punto de la consigna | Sección |
|---|---|
| Implementar y entrenar el agente | 8 y 9 |
| Ejecutar múltiples episodios de aprendizaje | 9 (3000 episodios × 10 semillas) |
| Analizar cómo evoluciona el comportamiento del agente | 10 |
| Representar gráficamente sus movimientos | 12 (gráficos) y 12.1 (animaciones) |
| Analizar la evolución de la recompensa durante el entrenamiento | 9 |
| Visualizar la estrategia o política aprendida | 11 |
| Modificar la función de recompensa y analizar cómo afecta al agente | 13 |
| (Extra) Sensibilidad a los hiperparámetros | 14 |
| Fundamentar las decisiones y relacionar resultados con el problema | texto de cada sección y 15 |
````

---
## CELDA 19 – TEXTO (pegar en una celda de texto)

````markdown
## 8. El agente Q-Learning

**Idea.** El agente tiene una tabla `Q` de 32 filas (estados) × 2 columnas (acciones). `Q[s, a]` estima cuánta recompensa total va a juntar si en el estado `s` hace la acción `a` y después juega lo mejor posible. Arranca en cero. Después de cada paso corrige solo la celda que usó:

`Q(s,a) ← Q(s,a) + α · [ r + γ · max Q(s',·) − Q(s,a) ]`

- `α` (tasa de aprendizaje): cuánto se corrige el valor viejo en cada paso.
- `γ` (descuento): cuánto valen las recompensas futuras frente a las inmediatas.
- `max Q(s',·)`: el valor de lo mejor que se puede hacer desde el estado siguiente.

**Decisiones de diseño y su justificación:**
- **Terminado vs. truncado.** Si el dino choca, el objetivo es solo `r`: después del choque no hay futuro. Si la partida se corta por el tope de 30 obstáculos, el juego *podría* seguir, así que igual sumamos `γ · max Q(s',·)`. Tratar el tope como un final enseñaría, falsamente, que ese estado es "malo".
- **Exploración ε-greedy con decaimiento por episodio.** Con probabilidad `ε` el agente elige una acción al azar (explora) y si no, la mejor según la tabla (explota). `ε` empieza en 1 (todo al azar) y baja ×0,995 por episodio hasta 0,01 (unos 920 episodios). Sin explorar, nunca descubriría que saltar a tiempo evita el choque.
- **Desempate al azar** cuando dos acciones valen igual (por ejemplo, al principio, con la tabla en cero). Sin esto, `argmax` elegiría siempre "nada" y el agente tardaría en probar saltar.
- **Se cuentan visitas** por celda para poder revisar después si la tabla se aprendió con suficientes datos.
````

---
## CELDA 20 – CÓDIGO

```python
# Valores iniciales de los hiperparámetros (los de la Entrega 1)
HIPERPARAMETROS = {
    "alfa": 0.1,          # tasa de aprendizaje
    "gamma": 0.99,        # descuento: alto, porque el castigo de un mal salto llega unos pasos después
    "eps_ini": 1.0,       # al principio explora siempre
    "eps_min": 0.01,      # piso de exploración
    "decaimiento": 0.995, # epsilon se multiplica por esto al final de cada episodio
}
EPISODIOS = 3000          # episodios de entrenamiento


class AgenteQ:
    # Agente Q-Learning tabular.

    def __init__(self, alfa, gamma, eps_ini, eps_min, decaimiento, semilla=None):
        self.alfa, self.gamma = alfa, gamma
        self.eps, self.eps_min, self.decaimiento = eps_ini, eps_min, decaimiento
        self.Q = np.zeros((N_ESTADOS, 2))                  # tabla Q: 32 estados x 2 acciones, todo en cero
        self.visitas = np.zeros((N_ESTADOS, 2), dtype=int) # cuántas veces se actualizó cada celda
        self.azar = random.Random(semilla)                 # generador propio del agente

    def mejor_accion(self, estado):
        # Acción con mayor Q en este estado. Si empatan, elige una al azar.
        q = self.Q[estado_a_indice(estado)]
        if q[0] == q[1]:
            return self.azar.choice([NADA, SALTAR])
        return int(np.argmax(q))

    def elegir(self, estado):
        # Política ε-greedy usada durante el entrenamiento.
        if self.azar.random() < self.eps:
            return self.azar.choice([NADA, SALTAR])        # explora
        return self.mejor_accion(estado)                   # explota

    def aprender(self, estado, accion, r, estado_sig, terminado):
        # Regla de actualización de Q-Learning para una transición (s, a, r, s').
        s, s2 = estado_a_indice(estado), estado_a_indice(estado_sig)
        # Si chocó (terminado) no hay futuro: el objetivo es solo r. El truncado SÍ tiene futuro.
        objetivo = r if terminado else r + self.gamma * self.Q[s2].max()
        self.Q[s, accion] += self.alfa * (objetivo - self.Q[s, accion])
        self.visitas[s, accion] += 1

    def fin_episodio(self):
        # Baja epsilon un poco al terminar cada episodio, sin pasar del piso.
        self.eps = max(self.eps_min, self.eps * self.decaimiento)

    def politica(self, estado):
        # Política final: sin exploración, siempre la mejor acción según la tabla.
        return self.mejor_accion(estado)


def entrenar(recompensas=RECOMPENSAS, episodios=EPISODIOS, semilla=0, guardar_en=(), **hiper):
    # Entrena un agente nuevo. Devuelve (agente, historial).
    #   recompensas: función de recompensa del entorno (se cambia en la sección 13)
    #   guardar_en:  episodios en los que se guarda una copia de la tabla Q (para ver la evolución)
    #   **hiper:     hiperparámetros que se quieren cambiar respecto de HIPERPARAMETROS
    params = {**HIPERPARAMETROS, **hiper}
    env = DinoEnv(recompensas, semilla=semilla)
    agente = AgenteQ(semilla=semilla + 1000, **params)
    hist = {"recompensa": [], "esquivados": [], "pasos": [], "epsilon": []}
    agente.copias = {}                                     # {episodio: copia de Q en ese momento}

    for ep in range(episodios + 1):
        if ep in guardar_en:
            agente.copias[ep] = agente.Q.copy()
        if ep == episodios:
            break
        estado, total, pasos, fin = env.reset(), 0, 0, False
        while not fin:
            accion = agente.elegir(estado)                                  # 1) decide
            estado_sig, r, terminado, truncado, info = env.step(accion)     # 2) el entorno responde
            agente.aprender(estado, accion, r, estado_sig, terminado)       # 3) corrige su tabla
            estado, total, pasos, fin = estado_sig, total + r, pasos + 1, terminado or truncado
        hist["recompensa"].append(total)                                    # lo anotado en este episodio
        hist["esquivados"].append(env.esquivados)
        hist["pasos"].append(pasos)
        hist["epsilon"].append(agente.eps)
        agente.fin_episodio()
    return agente, {k: np.array(v) for k, v in hist.items()}
```

---
## CELDA 21 – TEXTO (pegar en una celda de texto)

````markdown
## 9. Entrenamiento y evolución de la recompensa

Entrenamos 3000 episodios con los valores iniciales (α = 0,1, γ = 0,99, ε: 1 → 0,01).
Como un solo entrenamiento depende del azar, lo repetimos con **10 semillas** distintas y mostramos el promedio con una banda de ±1 desvío estándar. Si la banda es angosta, el resultado no depende de la suerte.

Las curvas son de la *política de entrenamiento* (con exploración). Por eso suben despacio y se quedan algo por debajo de 30: mientras ε sea grande el agente elige al azar, y aun con ε = 0,01 una acción al azar de vez en cuando alcanza para chocar. **Cuánto sabe realmente** la tabla se mide aparte, sin exploración, en la sección 10.
````

---
## CELDA 22 – CÓDIGO

```python
SEMILLAS = range(10)                                            # 10 entrenamientos independientes
EPISODIOS_CLAVE = (0, 10, 20, 30, 40, 50, 75, 100, 200, 400, 800, EPISODIOS)   # cuándo guardar copias de Q

corridas = [entrenar(semilla=s, guardar_en=EPISODIOS_CLAVE) for s in SEMILLAS]
agente, hist = corridas[0]            # la corrida 0 es la que usamos para mostrar tabla Q y partidas
historiales = [h for _, h in corridas]


def suavizar(x, ventana=50):
    # Media móvil: promedia cada episodio con los 49 anteriores, para ver la tendencia y no el ruido.
    return np.convolve(x, np.ones(ventana) / ventana, mode="valid")


def curva_con_banda(ax, clave, hists, titulo, ylabel, ventana=50):
    # Dibuja la media entre semillas de una métrica y una banda de ±1 desvío.
    datos = np.array([suavizar(h[clave], ventana) for h in hists])
    x = np.arange(datos.shape[1]) + ventana
    media, desvio = datos.mean(axis=0), datos.std(axis=0)
    ax.plot(x, media)
    ax.fill_between(x, media - desvio, media + desvio, alpha=0.25)
    ax.set_title(titulo); ax.set_xlabel("episodio"); ax.set_ylabel(ylabel); ax.grid(alpha=0.3)


fig, ejes = plt.subplots(1, 3, figsize=(16, 4))
curva_con_banda(ejes[0], "recompensa", historiales, "Recompensa por episodio (media móvil 50)", "recompensa")
ejes[0].axhline(linea_base["regla a mano"]["recompensa"], color="g", ls="--", label="regla a mano")
ejes[0].axhline(linea_base["azar"]["recompensa"], color="r", ls="--", label="al azar")
ejes[0].legend()
curva_con_banda(ejes[1], "esquivados", historiales, "Obstáculos esquivados (media móvil 50)", "obstáculos")
ejes[1].axhline(TOPE_OBSTACULOS, color="g", ls="--")
ejes[2].plot(hist["epsilon"])
ejes[2].set_title("Epsilon (exploración)"); ejes[2].set_xlabel("episodio"); ejes[2].grid(alpha=0.3)
plt.tight_layout(); plt.show()

# Episodio en que la media móvil de ENTRENAMIENTO llega al 95 % del tope.
# Ojo: esto mide cuándo bajó lo suficiente ε, no cuándo aprendió la tabla (eso se ve en la sección 10).
def episodio_de_convergencia(h, umbral=0.95 * TOPE_OBSTACULOS, ventana=50):
    idx = np.where(suavizar(h["esquivados"], ventana) >= umbral)[0]
    return int(idx[0]) + ventana if len(idx) else None

print("Episodio en que la curva de ENTRENAMIENTO llega al 95 % del tope, por semilla:",
      [episodio_de_convergencia(h) for h in historiales])
```

---
## CELDA 23 – TEXTO (pegar en una celda de texto)

````markdown
## 10. Cómo evoluciona el comportamiento del agente

Durante el entrenamiento guardamos copias de la tabla Q. Ahora evaluamos cada copia **sin exploración** (ε = 0) en 500 partidas nuevas. Así se ve cuánto sabe el agente en cada etapa, separado de cuánto explora.

Después mostramos **la misma partida** (misma semilla del entorno) jugada con la tabla en distintos momentos del entrenamiento, para ver el comportamiento cambiar con los ojos:
- con la tabla vacía el agente siempre elige "nada" y choca con el primer cactus,
- con poco entrenamiento acierta algunas decisiones y falla otras,
- con la tabla final salta solo cuando corresponde.
````

---
## CELDA 24 – CÓDIGO

```python
def politica_desde_Q(Q):
    # Política greedy (sin exploración) a partir de una tabla Q guardada.
    # Con empate elige NADA, para que la partida sea siempre la misma (y la tabla vacía = "nunca salta").
    return lambda estado: int(np.argmax(Q[estado_a_indice(estado)]))


# --- Gráfico 1: qué tan bueno es el agente según cuánto entrenó (promedio de las 10 semillas)
momentos = list(EPISODIOS_CLAVE)
evolucion = np.array([[evaluar(politica_desde_Q(ag.copias[ep]), 500, semilla=123)["esquivados"] for ep in momentos]
                      for ag, _ in corridas])
media, desvio = evolucion.mean(axis=0), evolucion.std(axis=0)

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(momentos, media, marker="o")
ax.fill_between(momentos, media - desvio, media + desvio, alpha=0.25)
ax.axhline(TOPE_OBSTACULOS, color="g", ls="--", label="tope (30)")
ax.axhline(linea_base["azar"]["esquivados"], color="r", ls="--", label="al azar")
ax.set_xscale("symlog", linthresh=10)            # escala log para ver bien los primeros episodios
ax.set_xlabel("episodios de entrenamiento"); ax.set_ylabel("obstáculos esquivados (sin exploración)")
ax.set_title("Qué sabe el agente según cuánto entrenó (10 semillas)"); ax.legend(); ax.grid(alpha=0.3)
plt.show()
for ep, m, d in zip(momentos, media, desvio):
    print(f"Tras {ep:5d} episodios: esquiva {m:5.2f} ± {d:4.2f} de 30")

# --- Resultado final contra las líneas base (1000 partidas nuevas por semilla, sin exploración)
finales = [evaluar(ag.politica, 1000, semilla=999) for ag, _ in corridas]
print(f"\nAgente entrenado ({EPISODIOS} episodios): esquiva {np.mean([f['esquivados'] for f in finales]):.2f} de 30, "
      f"recompensa {np.mean([f['recompensa'] for f in finales]):.1f}, "
      f"llega al tope en {np.mean([f['tasa_tope'] for f in finales]):.0%} de las partidas")
print(f"Al azar: {linea_base['azar']['esquivados']:.2f} de 30 | Regla a mano: {linea_base['regla a mano']['esquivados']:.0f} de 30")
```

---
## CELDA 25 – CÓDIGO

```python
def grabar_episodio(politica, semilla, max_pasos=120, recompensas=RECOMPENSAS):
    # Juega un episodio y guarda todo lo necesario para dibujarlo.
    # Devuelve: lista de alturas del dino, lista de (paso, tipo, evento) cuando un obstáculo llega,
    #           lista de acciones, y lista de estados (para los fotogramas).
    env = DinoEnv(recompensas, semilla=semilla)
    estado, fin, paso = env.reset(), False, 0
    alturas, llegadas, acciones, estados = [], [], [], [estado]
    while not fin and paso < max_pasos:
        a = politica(estado)
        tipo_obstaculo = estado[2]                            # el tipo del obstáculo que se está enfrentando
        estado, r, terminado, truncado, info = env.step(a)
        alturas.append(ALTURA_SEGUN_FASE[env.fase])           # altura del dino después de moverse
        acciones.append(a)
        estados.append(estado)
        if info["evento"] != "nada":                          # el obstáculo llegó: esquivó o chocó
            llegadas.append((paso, tipo_obstaculo, info["evento"]))
        paso += 1
        fin = terminado or truncado
    return alturas, llegadas, acciones, estados


def linea_de_tiempo(ax, politica, semilla, titulo, pasos_a_mostrar=40, recompensas=RECOMPENSAS):
    # Dibuja un episodio como línea de tiempo: la línea azul es la altura del dino y los bloques
    # son los obstáculos en el momento en que llegan a la columna 0 (naranja = esquivado, rojo = choque).
    alturas, llegadas, acciones, _ = grabar_episodio(politica, semilla, recompensas=recompensas)
    n = min(len(alturas), pasos_a_mostrar)
    ax.step(range(n), alturas[:n], where="mid", color="tab:blue", lw=2)
    for paso, tipo, evento in llegadas:
        if paso >= n:
            continue
        for h in OCUPA[tipo]:
            ax.add_patch(Rectangle((paso - 0.4, h - 0.4), 0.8, 0.8,
                                   color="tab:red" if evento == "choque" else "tab:orange", alpha=0.6))
        ax.text(paso, 2.6, "C" if tipo == CACTUS else "P", ha="center")   # tipo de obstáculo
    # Triángulo verde en cada salto real (acción SALTAR estando en el suelo)
    saltos = [i for i in range(n) if acciones[i] == SALTAR and (i == 0 or alturas[i - 1] == 0)]
    ax.scatter(saltos, [0] * len(saltos), marker="^", color="green", zorder=5, label="saltó")
    ax.set_yticks([0, 1, 2]); ax.set_ylim(-0.6, 3); ax.set_ylabel("altura")
    ax.set_title(titulo, fontsize=10); ax.grid(alpha=0.3); ax.legend(loc="upper right", fontsize=8)


# --- Gráfico 2: la misma partida con la tabla del episodio 0, 10, 30 y final
etapas = [0, 10, 30, EPISODIOS]
fig, ejes = plt.subplots(len(etapas), 1, figsize=(14, 2.6 * len(etapas)), sharex=True)
for ax, ep in zip(ejes, etapas):
    linea_de_tiempo(ax, politica_desde_Q(agente.copias[ep]), semilla=5,
                    titulo=f"Misma partida con la tabla Q del episodio {ep}")
ejes[-1].set_xlabel("paso")
plt.tight_layout(); plt.show()
```

---
## CELDA 26 – TEXTO (pegar en una celda de texto)

````markdown
## 11. Tabla Q y política aprendida

Las decisiones reales ocurren solo en los **8 estados con el dino en el suelo** (fase 0): 4 distancias × 2 tipos de obstáculo. En el aire ninguna acción cambia nada, así que sus filas de la tabla Q valen lo mismo para las dos acciones.

Abajo: la tabla Q de esos 8 estados con la política resultante, un mapa de calor con la ventaja de saltar `Q(saltar) − Q(nada)`, y el valor `max Q` de los 32 estados.

**Cómo leerlo.**
- Para el **cactus** hay tres distancias válidas para saltar (1, 2 y 3): cualquiera funciona, así que la política puede elegir saltar en cualquiera. Saltar a distancia 4 es un error porque el dino aterriza justo cuando llega el cactus.
- Para el **pájaro**, saltar a distancia 1, 2 o 3 es siempre un error. A distancia 4 da igual (el dino ya aterrizó cuando llega), por eso ahí la ventaja es ≈ 0.
- La política aprendida no tiene por qué ser igual a la regla a mano, pero sí debe rendir igual. Lo verificamos con la tasa de partidas que llegan al tope, no comparando tablas.
````

---
## CELDA 27 – CÓDIGO

```python
# --- Tabla de las 8 decisiones con el dino en el suelo
print(f"{'obstáculo':10s} {'dist':>4s} {'Q(nada)':>9s} {'Q(saltar)':>10s}  {'política':8s} {'visitas':>8s}")
for tipo in (CACTUS, PAJARO):
    for d in (4, 3, 2, 1):
        s = estado_a_indice((0, d, tipo))
        print(f"{NOMBRE_TIPO[tipo]:10s} {d:4d} {agente.Q[s, NADA]:9.1f} {agente.Q[s, SALTAR]:10.1f}  "
              f"{NOMBRE_ACCION[agente.mejor_accion((0, d, tipo))]:8s} {agente.visitas[s].sum():8d}")

# --- Mapa de calor: ventaja de saltar = Q(saltar) - Q(nada). Verde: conviene saltar. Rojo: conviene no saltar.
ventaja = np.zeros((2, 4))
for tipo in (CACTUS, PAJARO):
    for j, d in enumerate((4, 3, 2, 1)):
        s = estado_a_indice((0, d, tipo))
        ventaja[tipo, j] = agente.Q[s, SALTAR] - agente.Q[s, NADA]

fig, ejes = plt.subplots(1, 2, figsize=(14, 3.2))
lim = np.abs(ventaja).max()
im = ejes[0].imshow(ventaja, cmap="RdYlGn", vmin=-lim, vmax=lim)
ejes[0].set_xticks(range(4)); ejes[0].set_xticklabels(["a 4", "a 3", "a 2", "a 1"])
ejes[0].set_yticks([0, 1]); ejes[0].set_yticklabels(["cactus", "pájaro"])
for i in range(2):
    for j in range(4):
        etiqueta = "indif." if abs(ventaja[i, j]) < 1 else "SALTAR" if ventaja[i, j] > 0 else "nada"
        ejes[0].text(j, i, f"{ventaja[i, j]:.0f}\n{etiqueta}", ha="center", va="center", fontsize=9)
ejes[0].set_title("Ventaja de saltar con el dino en el suelo")
plt.colorbar(im, ax=ejes[0])

# --- Valor de cada uno de los 32 estados: max Q. Filas = fase, columnas = distancia. Una grilla por tipo.
V = np.zeros((2, 4, 4))
for i in range(N_ESTADOS):
    fase, d, tipo = indice_a_estado(i)
    V[tipo, fase, d - 1] = agente.Q[i].max()
ejes[1].imshow(np.hstack([V[CACTUS], np.full((4, 1), np.nan), V[PAJARO]]), cmap="viridis")
ejes[1].set_xticks([0, 1, 2, 3, 5, 6, 7, 8]); ejes[1].set_xticklabels(["1", "2", "3", "4"] * 2)
ejes[1].set_yticks(range(4)); ejes[1].set_yticklabels(["suelo", "subiendo", "arriba", "bajando"])
ejes[1].set_xlabel("distancia al obstáculo   (izquierda: cactus | derecha: pájaro)")
ejes[1].set_title("Valor max Q de los 32 estados")
plt.tight_layout(); plt.show()

# --- ¿La política aprendida nunca choca? Contamos las semillas con 100 % de partidas perfectas
perfectas = sum(f["tasa_tope"] == 1.0 for f in finales)
print(f"Semillas cuya política llega al tope en el 100 % de las partidas: {perfectas} de {len(finales)}")

# --- ¿Se visitaron todas las celdas de decisión? (si no, la tabla ahí no está respaldada por datos)
visitas_decision = [agente.visitas[estado_a_indice((0, d, t))].min() for t in (CACTUS, PAJARO) for d in (4, 3, 2, 1)]
print(f"Mínimo de visitas a una celda de decisión (acción menos probada): {min(visitas_decision)}")
```

---
## CELDA 28 – TEXTO (pegar en una celda de texto)

````markdown
## 12. Movimientos del agente

El agente entrenado jugando una partida, dibujado de dos formas:
1. **Fotogramas**: la grilla paso a paso, con el dino, el obstáculo y la acción elegida.
2. **Línea de tiempo** de un episodio largo, contra la de un agente al azar con la misma partida (la secuencia de obstáculos es la misma porque usan la misma semilla del entorno).
````

---
## CELDA 29 – CÓDIGO

```python
# --- Fotogramas: primeros pasos de una partida con la política aprendida
_, _, acciones, estados = grabar_episodio(agente.politica, semilla=5, max_pasos=12)
fig, ejes = plt.subplots(3, 4, figsize=(14, 8.5))
for k, ax in enumerate(ejes.flat):
    if k < len(estados):
        titulo = "estado inicial" if k == 0 else f"paso {k}: {NOMBRE_ACCION[acciones[k - 1]]}"
        dibujar_fotograma(ax, estados[k], titulo=titulo)
    else:
        ax.axis("off")
plt.tight_layout(); plt.show()

# --- Línea de tiempo: agente entrenado vs. al azar, misma partida
fig, ejes = plt.subplots(2, 1, figsize=(14, 5.5), sharex=True)
linea_de_tiempo(ejes[0], agente.politica, semilla=5, titulo="Agente entrenado (naranja = obstáculo esquivado; C/P = tipo)")
azar = random.Random(7)   # se reinicia el generador del agente al azar para que el resultado sea repetible
linea_de_tiempo(ejes[1], politica_azar, semilla=5, titulo="Agente al azar en la misma partida (rojo = choque)")
ejes[1].set_xlabel("paso")
plt.tight_layout(); plt.show()
```

---
## CELDA 30 – TEXTO (pegar en una celda de texto)

````markdown
### 12.1 Animaciones

Las mismas partidas, pero en movimiento. Cada animación tiene un reproductor (play, pausa y barra de tiempo) y se genera con `FuncAnimation` de `matplotlib`; cada cuadro es un fotograma de `dibujar_fotograma()`. Se dibujan con los sprites de la sección 3.

`animar_partidas()` recibe una lista de paneles `(título, política)` y los juega **todos con la misma semilla del entorno**, es decir, con la misma secuencia de obstáculos, uno al lado del otro. Cuando una partida termina por choque, su panel queda congelado en el último cuadro.

Se pueden guardar como GIF (por ejemplo, para la presentación de la Entrega 4) con `anim.save("nombre.gif", writer="pillow")`; para eso la función devuelve también el objeto `anim`.
````

---
## CELDA 31 – CÓDIGO

```python
from matplotlib.animation import FuncAnimation   # arma la animación a partir de una función que dibuja cada cuadro
from IPython.display import HTML, display        # muestra el reproductor dentro de Colab

plt.rcParams["animation.embed_limit"] = 60       # tamaño máximo (MB) del reproductor embebido; el valor por defecto (20) puede quedar chico


def animar_partidas(paneles, semilla, max_pasos=40, intervalo=500, columnas=2):
    # Anima varias partidas a la vez, todas con la misma semilla del entorno.
    #   paneles:   lista de (título, política); cada política es una función estado -> acción
    #   max_pasos: cantidad máxima de pasos a mostrar (limita el peso de la animación)
    #   intervalo: milisegundos entre cuadros
    # Devuelve (reproductor HTML, objeto anim). El objeto anim sirve para guardar un GIF.
    grabaciones = [grabar_episodio(pol, semilla, max_pasos) for _, pol in paneles]
    cuadros = max(len(g[3]) for g in grabaciones)               # el panel más largo marca la duración

    n = len(paneles)
    cols = min(columnas, n)
    filas = -(-n // cols)                                       # división hacia arriba
    fig, ejes = plt.subplots(filas, cols, figsize=(5.2 * cols, 3.4 * filas), squeeze=False)
    ejes = ejes.flatten()
    for ax in ejes[n:]:
        ax.axis("off")                                          # apaga los ejes que sobran

    def actualizar(k):
        # Dibuja el cuadro k de cada panel
        for ax, (titulo, _), (alturas, llegadas, acciones, estados) in zip(ejes, paneles, grabaciones):
            ax.clear()
            j = min(k, len(estados) - 1)                        # si la partida ya terminó, se queda en su último cuadro
            choco = j == len(estados) - 1 and len(llegadas) > 0 and llegadas[-1][2] == "choque"
            esquivados = sum(1 for paso, _, ev in llegadas if ev == "esquivó" and paso < j)
            accion = "inicio" if j == 0 else NOMBRE_ACCION[acciones[j - 1]]
            dibujar_fotograma(ax, estados[j], choque=choco,
                              titulo=f"{titulo}\npaso {j} ({accion}) | esquivados: {esquivados}" + ("  ¡CHOQUE!" if choco else ""))

    anim = FuncAnimation(fig, actualizar, frames=cuadros, interval=intervalo)
    plt.close(fig)                                              # evita que se muestre además un gráfico estático
    return HTML(anim.to_jshtml()), anim
```

---
## CELDA 32 – TEXTO (pegar en una celda de texto)

````markdown
**Animación 1 – El agente entrenado jugando.** Una partida completa con la política aprendida (los primeros 40 pasos).
````

---
## CELDA 33 – CÓDIGO

```python
reproductor, anim = animar_partidas([("Agente entrenado", agente.politica)], semilla=5, max_pasos=40)
display(reproductor)   # para guardar un GIF:  anim.save("agente.gif", writer="pillow")
```

---
## CELDA 34 – TEXTO (pegar en una celda de texto)

````markdown
**Animación 2 – Evolución del aprendizaje.** La misma partida jugada con la tabla Q del episodio 0, 10, 30 y final. Con la tabla vacía el dino nunca salta y choca contra el primer cactus; con la tabla final no choca.
````

---
## CELDA 35 – CÓDIGO

```python
etapas = [0, 10, 30, EPISODIOS]
paneles = [(f"Tabla Q del episodio {ep}", politica_desde_Q(agente.copias[ep])) for ep in etapas]
reproductor, anim = animar_partidas(paneles, semilla=5, max_pasos=40)
display(reproductor)
```

---
## CELDA 36 – TEXTO (pegar en una celda de texto)

````markdown
**Animación 3 – Agente al azar contra agente entrenado.** Misma secuencia de obstáculos para los dos. El de la izquierda salta o no a la suerte; el de la derecha decide con su tabla Q.
````

---
## CELDA 37 – CÓDIGO

```python
azar = random.Random(7)   # se reinicia el generador del agente al azar para que la animación sea repetible
reproductor, anim = animar_partidas([("Al azar", politica_azar), ("Agente entrenado", agente.politica)],
                                    semilla=5, max_pasos=40)
display(reproductor)
```

---
## CELDA 38 – TEXTO (pegar en una celda de texto)

````markdown
## 13. Cambiando la función de recompensa

La recompensa es lo único que le dice al agente qué queremos. Entrenamos el mismo agente (mismos α, γ, ε, episodios y semillas) con cuatro funciones distintas y medimos qué hace con cada una. Lo único que cambia entre experimentos es la recompensa.

| Configuración | paso | esquivar | choque | Qué queremos ver |
|---|---|---|---|---|
| Base (Entrega 1) | +1 | +10 | −100 | Referencia |
| Solo castigo | 0 | 0 | −100 | ¿Alcanza con castigar el choque? (recompensa dispersa) |
| Castigo débil | +1 | +10 | −5 | ¿Qué pasa si chocar casi no cuesta? |
| Costo de vivir | −1 | +2 | −5 | Cada paso cuesta más de lo que se gana: ¿conviene morir pronto? |

**Hipótesis antes de correrlo:**
- *Solo castigo*: debería aprender igual, porque el único evento que distingue buenas de malas decisiones es el choque.
- *Castigo débil*: también debería aprender, porque chocar corta todas las ganancias futuras de +1 y +10 por obstáculo, aunque el choque en sí cueste poco.
- *Costo de vivir*: con `paso = −1` y `esquivar = +2`, esquivar un obstáculo (4 pasos) deja **−2** neto, y chocar al principio cuesta como mucho −8. A este agente le conviene terminar rápido: es el clásico *reward hacking* (optimiza lo que pedimos, no lo que quisimos pedir).

Las métricas son siempre de la partida real (obstáculos esquivados y pasos vividos), no de la recompensa, porque esa cambia de escala entre configuraciones.
````

---
## CELDA 39 – CÓDIGO

```python
CONFIGS = {
    "Base":           {"paso": 1,  "esquivar": 10, "choque": -100},
    "Solo castigo":   {"paso": 0,  "esquivar": 0,  "choque": -100},
    "Castigo débil":  {"paso": 1,  "esquivar": 10, "choque": -5},
    "Costo de vivir": {"paso": -1, "esquivar": 2,  "choque": -5},
}

resultados_cfg, hist_cfg, agentes_cfg = {}, {}, {}
for nombre, rec in CONFIGS.items():
    corr = [entrenar(rec, semilla=s) for s in SEMILLAS]                    # 10 entrenamientos por configuración
    # Se evalúa con la MISMA función de recompensa con que se entrenó (sin exploración, 500 partidas)
    resultados_cfg[nombre] = [evaluar(ag.politica, 500, semilla=999, recompensas=rec) for ag, _ in corr]
    hist_cfg[nombre] = [h for _, h in corr]
    agentes_cfg[nombre] = corr[0][0]                                       # nos quedamos con el de la semilla 0

# --- Tabla de resultados
print(f"{'configuración':15s} {'esquiva (de 30)':>16s} {'tasa de tope':>13s} {'pasos vividos':>14s}")
for nombre, ev in resultados_cfg.items():
    print(f"{nombre:15s} {np.mean([e['esquivados'] for e in ev]):10.2f} ± {np.std([e['esquivados'] for e in ev]):4.1f} "
          f"{np.mean([e['tasa_tope'] for e in ev]):13.0%} {np.mean([e['pasos'] for e in ev]):14.1f}")

# --- Curvas de aprendizaje de las cuatro configuraciones
fig, ax = plt.subplots(figsize=(9, 4))
for nombre, hs in hist_cfg.items():
    datos = np.array([suavizar(h["esquivados"], 50) for h in hs])
    ax.plot(np.arange(datos.shape[1]) + 50, datos.mean(axis=0), label=nombre)
ax.axhline(TOPE_OBSTACULOS, color="gray", ls="--", lw=1)
ax.set_xlabel("episodio"); ax.set_ylabel("obstáculos esquivados (media móvil 50)")
ax.set_title("Mismo agente, distintas recompensas"); ax.legend(); ax.grid(alpha=0.3)
plt.show()

# --- Política resultante en las 8 decisiones, para cada configuración (C = cactus, P = pájaro, @n = distancia)
print()
print(f"{'configuración':15s} " + " ".join(f"{NOMBRE_TIPO[t][0].upper()}@{d}" for t in (CACTUS, PAJARO) for d in (4, 3, 2, 1)))
for nombre, ag in agentes_cfg.items():
    celdas = ["SALTA" if ag.mejor_accion((0, d, t)) == SALTAR else "nada" for t in (CACTUS, PAJARO) for d in (4, 3, 2, 1)]
    print(f"{nombre:15s} " + " ".join(f"{c:>5s}" for c in celdas))

# --- El comportamiento "suicida" a la vista: misma partida con la política de Base y de Costo de vivir
fig, ejes = plt.subplots(2, 1, figsize=(14, 5.5), sharex=True)
linea_de_tiempo(ejes[0], agentes_cfg["Base"].politica, semilla=5, titulo="Recompensa base")
linea_de_tiempo(ejes[1], agentes_cfg["Costo de vivir"].politica, semilla=5,
                titulo="Costo de vivir (aprende a chocar contra el primer obstáculo)", recompensas=CONFIGS["Costo de vivir"])
ejes[1].set_xlabel("paso")
plt.tight_layout(); plt.show()
```

---
## CELDA 40 – TEXTO (pegar en una celda de texto)

````markdown
## 14. Sensibilidad a α, γ y ε

Variamos un hiperparámetro por vez, dejando el resto en los valores iniciales, con la recompensa base y 10 semillas.
Como la tabla ya es perfecta después de pocas decenas de episodios (sección 10), comparamos con un presupuesto chico: **100 episodios de entrenamiento**. La métrica es cuánto esquiva el agente después, sin explorar, y en cuántas de las 10 semillas llega siempre al tope.
````

---
## CELDA 41 – CÓDIGO

```python
def barrido(nombre, valores, semillas=range(10), episodios=100):
    # Entrena 10 agentes por cada valor del hiperparámetro y mide cómo juegan sin explorar.
    filas = []
    for v in valores:
        ev = [evaluar(entrenar(semilla=s, episodios=episodios, **{nombre: v})[0].politica, 300, semilla=999)
              for s in semillas]
        filas.append((v, np.mean([e["esquivados"] for e in ev]), sum(e["tasa_tope"] == 1.0 for e in ev)))
    return filas


for nombre, valores in {"alfa": [0.01, 0.1, 0.5], "gamma": [0.0, 0.5, 0.99], "decaimiento": [0.95, 0.99, 0.995, 0.999]}.items():
    print(f"\n{nombre} (100 episodios de entrenamiento)")
    print(f"{'valor':>8s} {'esquiva (sin explorar)':>23s} {'semillas perfectas':>19s}")
    for v, esq, perfectas in barrido(nombre, valores):
        print(f"{v:8} {esq:23.2f} {perfectas:15d}/10")
```

---
## CELDA 42 – TEXTO (pegar en una celda de texto)

````markdown
## 15. Conclusiones

Los números salen de correr este notebook con las semillas fijas (0 a 9). Si se cambian las semillas o los parámetros, hay que revisarlos.

- **Aprendizaje.** El agente al azar esquiva ≈ 1 obstáculo de 30. Con Q-Learning, la política sin exploración llega a esquivar los 30 en el 100 % de las partidas, en las 10 semillas, a partir de ~40 episodios (a los 30 todavía esquiva ≈ 21,5 ± 13 de 30). La curva de entrenamiento sube mucho más lento (≈ 1000 episodios) y se queda cerca de 27 porque incluye la exploración: ε sigue alto cuando la tabla ya es perfecta.
- **Política.** Con cactus salta a distancia 1 (también sirven 2 y 3) y nunca a distancia 4; con pájaro no salta a distancia 1, 2 ni 3. A distancia 4 con pájaro da igual. La política aprendida puede diferir de la regla a mano y rendir igual.
- **Recompensa.** Con la base, con solo castigo por choque y con castigo débil (−5), el agente aprende a esquivar todo, porque seguir vivo ya vale más que cualquier choque; solo cambia *dónde* salta cuando es indiferente. Con costo de vivir (paso −1, esquivar +2, choque −5) aprende a chocar contra el primer obstáculo (4 pasos de vida): maximiza lo que le pedimos, no lo que queríamos. La función de recompensa define el comportamiento tanto como el algoritmo.
- **Hiperparámetros.** α = 0,01 es demasiado lento para 100 episodios (4 de 10 semillas perfectas); α = 0,1 y 0,5 alcanzan. γ = 0 **falla** (2 de 10 semillas): sin descuento el agente no ve que saltar a distancia 4 con cactus, o saltar con pájaro a distancia 2 o 3, termina en choque unos pasos después. Esto confirma por qué γ alto es necesario. El decaimiento de ε casi no importa a 100 episodios porque Q-Learning aprende la política greedy aun explorando (es *off-policy*).
- **Relación con el problema.** El dinosaurio es un problema donde nadie da respuestas etiquetadas, las decisiones son secuenciales y el castigo llega con demora: justo el tipo de problema para el que sirve el aprendizaje por refuerzo.
- **Limitaciones.** El entorno es chico y casi determinístico (solo es aleatorio el tipo del próximo obstáculo), por eso una tabla lo resuelve por completo y el aprendizaje termina rápido. Un paso siguiente sería agregar velocidad variable o separación aleatoria entre obstáculos para que haya decisiones menos triviales.
````
