# Garden
# 🌸 Jardim Pontilhista em Python: do zero

Neste tutorial você vai **construir, do zero, uma animação em Python** em que várias flores nascem, crescem, florescem, brilham e murcham, tudo desenhado com milhares de pontinhos, como numa pintura pontilhista.

Você vai usar apenas **Python, NumPy e Pillow** (e Tkinter para a janela animada). No final você terá um único arquivo, `jardim.py`, que:

- abre uma janela com a animação;
- exporta imagens PNG de cada etapa;
- exporta um GIF animado.

> **Como seguir:** o tutorial é uma sequência de passos. Em cada passo você **cola o bloco de código no final do arquivo `jardim.py`**, na ordem em que eles aparecem. Ao terminar, o arquivo estará completo. Se preferir, o arquivo pronto também está disponível junto deste README.

---

## Sumário

0. [Preparação](#parte-0--preparação)
1. [A ideia por trás do projeto](#parte-1--a-ideia-por-trás-do-projeto)
2. [Construindo o código, passo a passo](#parte-2--construindo-o-código-passo-a-passo)
3. [Rodando](#parte-3--rodando)
4. [Personalizando](#parte-4--personalizando)
5. [Problemas comuns](#parte-5--problemas-comuns)
6. [Ideias para continuar](#parte-6--ideias-para-continuar)

---

## Parte 0 · Preparação

**Requisitos**

- Python 3.9 ou mais novo
- As bibliotecas `numpy` e `pillow`
- Tkinter (só para a janela animada)

```bash
pip install numpy pillow
```

O Tkinter já vem com o Python no Windows e no macOS. No Linux (Ubuntu/Debian):

```bash
sudo apt install python3-tk
```

**Crie uma pasta** para o projeto (por exemplo `jardim/`) e, dentro dela, um arquivo vazio chamado **`jardim.py`**. É nele que vamos colar tudo.

> 💡 Depois de cada passo você pode conferir se não há erro de digitação com:
>
> ```bash
> python -c "import jardim"
> ```
>
> Se não aparecer nada, está tudo certo.

---

## Parte 1 · A ideia por trás do projeto

Antes de escrever código, vale entender cinco ideias. Todo o resto é consequência delas.

**1. Tudo é feito de pontos.** O solo, o caule, as folhas e as pétalas são nuvens de pontinhos coloridos. Cada ponto tem uma posição, uma cor e um raio (1 a 2 pixels).

**2. Os pontos são sorteados uma única vez.** Na criação da flor, sorteamos os pontos de cada parte (com `numpy.random`). Esses sorteios são guardados e reutilizados em todos os quadros, então a flor não "treme" de um quadro para o outro.

**3. A posição é uma fórmula do tempo.** Para cada quadro, uma função `render(t)` calcula, com fórmulas, onde cada ponto fica naquele instante `t`. Assim a flor cresce, abre e murcha de forma contínua.

**4. Uma linha do tempo comanda tudo.** Uma tabela (`KEYS`) diz, em cada segundo, quanto o caule já cresceu, quanto a flor já abriu, quanto já murchou e assim por diante. Os valores entre as linhas são suavizados por interpolação.

**5. Cada flor é independente.** Uma `Plant` tem **posição, tamanho, semente e atraso próprios**. Um `Garden` (jardim) apenas desenha várias `Plant` no mesmo quadro, sobre o mesmo solo.

Resumindo o fluxo de cada quadro:

```text
tempo t  →  state(t)  →  fórmulas posicionam os pontos  →  splat desenha  →  imagem
```

---

## Parte 2 · Construindo o código, passo a passo

### Passo 1 · Imports e configurações

Começamos com as bibliotecas e alguns botões de ajuste. Abra o `jardim.py` e cole:

```python
import math
import sys
import time

import numpy as np
from PIL import Image
```

Agora as configurações gerais. Elas ficam no topo para você mexer fácil:

```python
SEED = 5
TIME_SCALE = 1.0
TICK_MS = 10
WIN_FRAC = 0.92
LOOP = False
```

- `SEED`: semente aleatória padrão (muda o formato das pétalas).
- `TIME_SCALE`: velocidade da animação (valores maiores deixam **mais lenta**).
- `TICK_MS`: intervalo entre quadros na janela (maior = menos pesado).
- `WIN_FRAC`: quanto da altura da tela a janela ocupa.
- `LOOP`: se `True`, a animação reinicia sozinha no final.

E a lista das flores do jardim:

```python
# (posição x [0 a 1], escala, atraso em segundos, seed)
# A PRIMEIRA é a flor principal: é ela que cria o solo.
FLOWERS = [
    (0.50, 1.00, 0.0, 5),
    (0.20, 0.70, 3.0, 11),
    (0.80, 0.75, 5.0, 23),
    (0.35, 0.55, 8.0, 31),
    (0.66, 0.50, 10.0, 47),
]
```

Cada linha é uma flor: `(posição x, escala, atraso, seed)`.

- **posição x:** de 0 (borda esquerda) a 1 (borda direita).
- **escala:** 1.0 é o tamanho padrão; 0.5 é metade.
- **atraso:** segundos até ela começar a nascer.
- **seed:** semente que muda pétalas e cores, para nenhuma ser igual à outra.

A **primeira** flor da lista é a flor principal: é ela que cria o solo que todas compartilham.

### Passo 2 · A linha do tempo (`KEYS`)

Esta tabela é o "roteiro" da vida da flor. Cada linha é `[tempo, caule, tamanho, abertura, murchar, folhas, brilho]`, e todos os valores vão de 0 a 1:

```python
KEYS = np.array([
    [0.0, 0.00, 0.00, 0.00, 0.0, 0.00, 0.00],
    [5.0, 0.42, 0.22, 0.00, 0.0, 0.25, 0.00],
    [10.0, 0.62, 0.42, 0.00, 0.0, 0.50, 0.35],
    [16.0, 0.80, 0.68, 0.50, 0.0, 0.72, 0.55],
    [24.0, 1.00, 1.00, 1.00, 0.0, 1.00, 1.00],
    [30.0, 1.00, 1.00, 1.00, 0.0, 1.00, 1.00],
    [39.0, 0.98, 0.85, 0.55, 1.0, 0.92, 0.00],
    [45.0, 0.98, 0.85, 0.55, 1.0, 0.90, 0.00],
])
T_END = float(KEYS[-1, 0])
```

| Coluna | Significado |
|--------|-------------|
| `tempo` | segundo da animação |
| `caule` | quanto o caule já cresceu |
| `tamanho` | tamanho da cabeça da flor |
| `abertura` | quanto as pétalas já se abriram |
| `murchar` | 0 = viva, 1 = murcha |
| `folhas` | quanto as folhas cresceram |
| `brilho` | intensidade do halo e das partículas brilhantes |

Lendo as linhas: de 0 a 24 s a flor brota e se abre; de 24 a 30 s ela fica no auge; de 30 a 39 s ela murcha; e `T_END` é o tempo da última linha (45 s).

### Passo 3 · Paletas de cores e formas

Cada parte da flor tem uma paleta: uma lista de cores `(R, G, B)`. As versões `*_WILT` são as cores da planta murcha (tons de marrom):

```python
SOIL_PAL = [(34, 25, 20), (50, 36, 28), (68, 50, 37), (88, 65, 47), (108, 82, 60), (64, 58, 52)]
STEM_PAL = [(62, 38, 120), (100, 58, 165), (140, 84, 200), (60, 160, 140)]
STEM_WILT = [(120, 88, 66), (96, 66, 52), (140, 108, 76)]
LEAF_PAL = [(60, 150, 140), (80, 120, 175), (130, 80, 195), (175, 105, 215)]
LEAF_WILT = [(130, 100, 70), (104, 76, 58), (146, 116, 84)]
RIB_PAL = [(210, 180, 255), (170, 230, 220)]
WILT_PAL = [(150, 86, 128), (116, 64, 108), (88, 52, 88), (126, 94, 104), (104, 72, 56)]
RIM_PAL = [(205, 165, 255), (255, 190, 250), (170, 140, 255)]
HI_PAL = [(235, 215, 255), (255, 225, 250)]
GOLD_PAL = [(255, 214, 110), (255, 238, 170), (255, 184, 90)]
STAMEN_PAL = [(255, 240, 255), (255, 215, 235), (255, 230, 150), (255, 190, 80)]
STAMEN_WILT = [(140, 112, 84), (120, 96, 72)]
```

Agora as **camadas de pétalas**. A flor tem 4 camadas (de fora para dentro). Em cada uma, `n` é o número de pétalas, `d` é o número de pontos por pétala, `L` é o comprimento relativo, `ac/ao` é o ângulo com a flor fechada/aberta, `kc/ko` é a curvatura fechada/aberta, `wc/wo` é a largura fechada/aberta e `off` é um deslocamento de ângulo (para as camadas não ficarem alinhadas):

```python
LAYERS = [
    dict(n=8, ac=0.60, ao=1.00, kc=-1.20, ko=0.45, L=1.00, wc=0.20, wo=0.34, d=1150, off=0.00,
         pal=[(210, 80, 225), (170, 58, 225), (128, 50, 218), (96, 44, 205), (78, 42, 185)]),
    dict(n=6, ac=0.48, ao=0.68, kc=-1.05, ko=0.15, L=0.88, wc=0.19, wo=0.31, d=950, off=0.40,
         pal=[(255, 150, 235), (245, 92, 218), (205, 62, 218), (150, 52, 222)]),
    dict(n=5, ac=0.36, ao=0.46, kc=-0.85, ko=-0.05, L=0.72, wc=0.17, wo=0.27, d=760, off=0.20,
         pal=[(255, 205, 245), (255, 135, 232), (238, 84, 216), (195, 62, 226)]),
    dict(n=4, ac=0.22, ao=0.32, kc=-0.50, ko=-0.10, L=0.46, wc=0.14, wo=0.22, d=480, off=0.55,
         pal=[(255, 238, 252), (255, 185, 242), (255, 125, 228)]),
]
```

E a definição das 4 folhas do caule. Cada tupla é `(posição no caule, lado, tamanho, ângulo, curvatura)`:

```python
LEAF_DEFS = [
    (0.10, +1, 1.00, 0.55, -0.80),
    (0.16, -1, 0.90, 0.50, -0.85),
    (0.40, +1, 0.70, 0.95, -0.70),
    (0.33, -1, 0.62, 0.90, -0.65),
]
```

Para mudar o visual da flor, é aqui que você mexe: troque as cores ou o número de pétalas e rode de novo.

### Passo 4 · As ferramentas do pontilhismo

Aqui está o coração técnico do projeto: como desenhar um ponto (um pequeno círculo) numa imagem usando NumPy, **sem laços lentos**.

Primeiro, uma tabela de "deslocamentos" que descreve um círculo de pixels, ordenados do centro para fora. Com ela, um ponto de raio 1 usa os primeiros 5 pixels, de raio 1,5 usa 9, e de raio 2 usa 13:

```python
_OFF = sorted(((dx, dy) for dy in range(-3, 4) for dx in range(-3, 4) if dx * dx + dy * dy <= 9),
              key=lambda p: (p[0] ** 2 + p[1] ** 2, p))
OFF = np.array(_OFF, np.int32)
D2 = (OFF ** 2).sum(1)
```

Depois, funções pequenas e úteis que vamos usar em tudo:

```python
def smoothstep(x, a, b):
    t = np.clip((x - a) / (b - a), 0, 1)
    return t * t * (3 - 2 * t)


def sinc(x):
    return np.sinc(x / np.pi)


def profile(u):
    return np.sin(np.pi * np.clip(u, 0, 1) ** 0.75) ** 0.85


def palette_pick(rng, pal, g, spread=0.55, jitter=0.07):
    pal = np.asarray(pal, np.float32)
    g = np.asarray(g, np.float32)
    x = np.clip(g, 0, 1) * (len(pal) - 1) + rng.normal(0, spread, g.shape)
    idx = np.clip(np.rint(x), 0, len(pal) - 1).astype(int)
    return (pal[idx] * (1 + rng.normal(0, jitter, (len(idx), 1)))).astype(np.float32)


def area_u(rng, n):
    out = np.empty(0)
    while out.size < n:
        u = rng.random(n * 3)
        out = np.concatenate([out, u[rng.random(n * 3) < profile(u)]])
    return out[:n]


def sample_v(rng, n, rim=0.14):
    v = rng.uniform(-1, 1, n)
    r = rng.random(n) < rim
    v[r] = np.sign(v[r]) * rng.uniform(0.88, 1.0, int(r.sum()))
    return v


def permute(d, rng):
    n = len(next(iter(d.values())))
    p = rng.permutation(n)
    return {k: v[p] for k, v in d.items()}


def cat(parts):
    return {k: np.concatenate([p[k] for p in parts]) for k in parts[0]}
```

O que cada uma faz:

- **`smoothstep`:** transição suave de 0 a 1 entre dois valores.
- **`sinc`:** auxiliar matemática para desenhar arcos (usada nas pétalas curvas).
- **`profile(u)`:** perfil de largura de uma pétala ou folha. Vale 0 nas duas pontas e é larga no meio, dando o formato de "gota".
- **`palette_pick`:** escolhe cores de uma paleta, seguindo um gradiente `g` com ruído, e varia um pouco o brilho de cada ponto.
- **`area_u`:** sorteia posições ao longo da pétala de forma que a **densidade de pontos acompanhe a largura** (mais pontos onde é mais larga).
- **`sample_v`:** sorteia a posição lateral, com um reforço nas bordas para desenhar o contorno.
- **`permute`:** embaralha todos os pontos de uma parte. Isso é essencial: mais adiante, para a flor *crescer*, mostramos só os primeiros `nv` pontos. Como estão embaralhados, esse pedaço é uma amostra uniforme da parte inteira.
- **`cat`:** junta vários dicionários de pontos em um só.

Agora a função que de fato desenha os pontos:

```python
def splat(canvas, x, y, col, rad, nmax=13):
    H, W = canvas.shape[:2]
    xi = np.rint(x).astype(np.int32)
    yi = np.rint(y).astype(np.int32)
    off = OFF[:nmax]
    xs = xi[:, None] + off[None, :, 0]
    ys = yi[:, None] + off[None, :, 1]
    m = (D2[None, :nmax] <= (rad * rad)[:, None] + 0.01) & (xs >= 0) & (xs < W) & (ys >= 0) & (ys < H)
    rows = np.nonzero(m)[0]
    canvas[ys[m], xs[m]] = col[rows]
```

Para cada ponto, ela calcula os pixels do círculo ao redor, descarta os que saem da imagem e pinta tudo de uma vez com indexação do NumPy.

Por fim, o `Batch`. Ele junta os pontos de várias partes da flor e os **ordena por profundidade (`z`)** antes de desenhar. Pontos com `z` maior são pintados depois, ficando na frente:

```python
class Batch:
    def __init__(self):
        self.p = []

    def add(self, x, y, col, rad, z):
        if len(x):
            self.p.append((x, y, col, rad, z))

    def draw(self, canvas):
        if not self.p:
            return
        x, y, col, rad, z = [np.concatenate([q[i] for q in self.p]) for i in range(5)]
        o = np.argsort(z, kind="stable")
        c = np.clip(col[o], 0, 255).astype(np.uint8)
        splat(canvas, x[o], y[o], c, rad[o])
```


> ✅ **Checkpoint:** rode `python -c "import jardim"`. Se não aparecer erro, está tudo certo até aqui.



### Passo 5 · Lendo a linha do tempo (`state`)

A função `state(t)` recebe um instante `t` e devolve os 6 valores (caule, tamanho, abertura, murchar, folhas, brilho) daquele momento. Ela usa uma interpolação *Catmull-Rom*, que passa exatamente pelos pontos de `KEYS` e é **suave**, sem "quinas" entre as fases:

```python
def state(t):
    t = float(min(max(t, 0.0), T_END))
    ts = KEYS[:, 0]
    i = min(max(int(np.searchsorted(ts, t, side="right")) - 1, 0), len(KEYS) - 2)
    u = (t - ts[i]) / (ts[i + 1] - ts[i])
    p0 = KEYS[max(i - 1, 0), 1:]
    p1 = KEYS[i, 1:]
    p2 = KEYS[i + 1, 1:]
    p3 = KEYS[min(i + 2, len(KEYS) - 1), 1:]
    v = 0.5 * (2 * p1 + (p2 - p0) * u + (2 * p0 - 5 * p1 + 4 * p2 - p3) * u * u
               + (3 * p1 - p0 - 3 * p2 + p3) * u ** 3)
    return np.clip(v, 0, 1)
```

Note que `t` é limitado entre 0 e `T_END`. Isso é o que faz uma flor com atraso ficar invisível antes da hora: com `t` negativo, `state` devolve tudo zero.

### Passo 6 · O caule: curva de Bézier

O caule não é uma linha reta, é uma curva suave. Usamos uma **curva de Bézier cúbica**, definida por 4 pontos de controle. A função `bez` devolve, para cada posição `s` ao longo da curva (de 0 a 1), as coordenadas `(x, y)` e também a **direção** da curva `(dx, dy)`. A direção serve para empurrar pontos para os lados e dar espessura ao caule.

A função `mixcol` mistura a cor "viva" com a cor "murcha" conforme o peso `w`:

```python
def bez(P, s):
    P0, P1, P2, P3 = P
    q = 1 - s
    x = q ** 3 * P0[0] + 3 * q * q * s * P1[0] + 3 * q * s * s * P2[0] + s ** 3 * P3[0]
    y = q ** 3 * P0[1] + 3 * q * q * s * P1[1] + 3 * q * s * s * P2[1] + s ** 3 * P3[1]
    dx = 3 * q * q * (P1[0] - P0[0]) + 6 * q * s * (P2[0] - P1[0]) + 3 * s * s * (P3[0] - P2[0])
    dy = 3 * q * q * (P1[1] - P0[1]) + 6 * q * s * (P2[1] - P1[1]) + 3 * s * s * (P3[1] - P2[1])
    return x, y, dx, dy


def mixcol(live, wilt, w):
    return live + (wilt - live) * w[:, None]
```


> ✅ **Checkpoint:** rode `python -c "import jardim"`. Se não aparecer erro, está tudo certo até aqui.



#### Agora a classe `Plant`

Uma `Plant` é **uma flor inteira**. Vamos criá-la em partes: primeiro o construtor, depois um método para construir cada parte, e por fim o `render`.

> Todos os métodos abaixo ficam **dentro** da classe `Plant`, com a mesma indentação. Cole um após o outro.

### Passo 7 · O construtor

O construtor já nasce preparado para o jardim: recebe posição (`cx`), escala (`scale`), semente (`seed`), atraso (`delay`) e uma função de chão (`ground`):

```python
class Plant:
    def __init__(self, W, H, cx=None, scale=1.0, seed=SEED, delay=0.0, ground=None):
        self.W, self.H = W, H
        self.rng = np.random.default_rng(seed)
        self.cx = W / 2 if cx is None else cx
        self.scale, self.delay, self.ground = scale, delay, ground
        self.y0 = H * 0.80 if ground is None else float(ground(self.cx)) + 0.01 * H
        self.Ls = H * 0.38 * scale
        self.Lf = H * 0.235 * scale
        if ground is None:  # só a flor principal cria solo e rachaduras
            self.build_soil()
        else:
            self.base, self.cr = None, None
        self.build_stem()
        self.build_head()
        self.build_stamens()
        self.build_leaves()
        self.build_fx()
```

Pontos importantes:

- `self.rng` é um gerador de números aleatórios com **semente própria**. Sementes diferentes dão flores diferentes; a mesma semente dá sempre a mesma flor.
- `Ls` e `Lf` são o comprimento do caule e das pétalas. Ambos são multiplicados por `scale`, então uma flor com `scale=0.5` é metade do tamanho.
- `ground` é opcional. Se for `None`, esta é a **flor principal**: ela cria o solo (`build_soil`). Se for uma função, a flor é **secundária**: ela se planta no solo da principal, na altura que `ground` indicar naquela posição `x`.
- `self.y0` é a altura onde o caule começa (a linha do solo naquele ponto).

### Passo 8 · O solo

A função `surface` diz a altura do solo em cada posição `x`. Ela é uma mistura de ondas suaves com uma pequena elevação no centro. Se a flor for secundária, ela simplesmente usa a função de chão da principal:

```python
    def surface(self, x):
        if self.ground is not None:
            return self.ground(x)
        H = self.H
        return (self.y0 - 0.022 * H * np.exp(-((x - self.cx) / (0.2 * self.W)) ** 2)
                - 0.003 * H * np.sin(x * 0.045) - 0.002 * H * np.sin(x * 0.13 + 1.3))
```

`build_soil` desenha a terra, uma única vez, numa imagem de fundo (`self.base`). São 26 mil pontos marrons, mais escuros com a profundidade e nas bordas, mais algumas pedrinhas. Ela também sorteia as "rachaduras": pontos de terra que sobem e descem quando o broto aparece.

```python
    def build_soil(self):
        W, H, rng = self.W, self.H, self.rng
        n = 26000
        x = rng.uniform(0, W, n)
        d = (rng.random(n) ** 1.5) * 0.14 * H
        y = self.surface(x) + d
        depth = d / (0.14 * H)
        edge = np.clip(np.minimum(x, W - x) / (0.2 * W), 0, 1) ** 0.8
        clump = 0.5 + 0.5 * np.sin(x * 0.07 + y * 0.05) * np.sin(y * 0.11 - x * 0.03)
        bright = (1 - depth) ** 0.8 * edge * (0.55 + 0.35 * clump + 0.35 * rng.random(n))
        col = palette_pick(rng, SOIL_PAL, rng.random(n), spread=1.0) * bright[:, None]
        rad = rng.choice([1.0, 1.5, 2.0], n, p=[0.4, 0.4, 0.2])
        self.base = np.zeros((H, W, 3), np.uint8)
        splat(self.base, x, y, np.clip(col, 0, 255).astype(np.uint8), rad)
        k = 170
        px = rng.uniform(0.05 * W, 0.95 * W, k)
        py = self.surface(px) + rng.random(k) ** 1.3 * 0.1 * H
        pc = palette_pick(rng, [(58, 52, 48), (84, 74, 64), (104, 90, 76), (46, 38, 32)], rng.random(k), spread=0.7)
        pc = pc * (np.clip(np.minimum(px, W - px) / (0.2 * W), 0, 1) ** 0.8)[:, None]
        splat(self.base, px, py, np.clip(pc, 0, 255).astype(np.uint8), rng.choice([2.0, 3.0], k, p=[0.6, 0.4]), nmax=29)
        m = 520
        cx = self.cx + rng.normal(0, 0.03 * H, m)
        self.cr = dict(
            x=cx,
            y=self.surface(cx) - rng.random(m) * 0.006 * H,
            amp=H * 0.05 * rng.random(m) ** 2.2,
            delay=rng.uniform(0, 2.5, m),
            col=(palette_pick(rng, SOIL_PAL, rng.uniform(0.3, 1, m), spread=0.8) * 0.8).astype(np.float32),
            rad=rng.choice([1.0, 1.5, 2.0], m, p=[0.4, 0.4, 0.2]),
        )
```

Cada quadro começa copiando `self.base`; assim o solo não precisa ser recalculado.

### Passo 9 · O caule e a curva de controle

`build_stem` sorteia 3600 pontos ao longo do caule: `s` é a posição ao longo da curva (0 = raiz, 1 = topo) e `lat` é o deslocamento lateral, que dá espessura. Cada ponto guarda uma cor viva e uma cor murcha.

`ctrl` calcula os 4 pontos de controle da Bézier. Repare como o formato muda com `wilt` (murchar): o caule se curva e a cabeça tomba:

```python
    def build_stem(self):
        rng, n = self.rng, 3600
        s = rng.random(n)
        lat = rng.uniform(-1, 1, n)
        rim = rng.random(n) < 0.18
        lat[rim] = np.sign(lat[rim]) * rng.uniform(0.85, 1.0, int(rim.sum()))
        live = palette_pick(rng, STEM_PAL[:3], s * 0.8 + rng.random(n) * 0.2)
        gr = rng.random(n) < 0.12
        live[gr] = np.asarray(STEM_PAL[3], np.float32) * (1 + rng.normal(0, 0.08, (int(gr.sum()), 1)))
        self.stem = permute(dict(
            s=s, lat=lat, live=live,
            wilt=palette_pick(rng, STEM_WILT, rng.random(n), spread=0.8),
            rn=rng.random(n), rad=rng.choice([1.0, 1.5, 2.0], n, p=[0.35, 0.45, 0.2])), rng)
```

```python
    def ctrl(self, L, wilt, sway):
        cx, y0 = self.cx, self.y0
        P0 = (cx, y0 + 4.0)
        P1 = (cx - 0.06 * L * (1 - wilt) + 0.02 * L * wilt, y0 - 0.33 * L)
        P2 = (cx + 0.05 * L * (1 - wilt) + 0.08 * L * wilt + sway * 0.5, y0 - L * (0.74 + 0.30 * wilt))
        P3 = (cx + sway + 0.32 * L * wilt, y0 - L * (1 - 0.15 * wilt))
        return P0, P1, P2, P3
```

### Passo 10 · A cabeça da flor (pétalas)

Este é o método mais rico. Para cada camada, e para cada pétala da camada, sorteamos os pontos (`u` ao longo da pétala, `v` ao longo da largura) e as cores:

- bordas ganham tons mais claros (`RIM_PAL`);
- uns poucos pontos viram "brilhos" (`HI_PAL`) e "centelhas douradas" (`GOLD_PAL`);
- cada pétala tem pequenas variações próprias de ângulo (`aj`), comprimento (`lj`), curvatura ao murchar (`pj`) e murcha (`wj`), para que ela não pareça um carimbo repetido.

```python
    def build_head(self):
        rng = self.rng
        parts = []
        for Ly in LAYERS:
            d = Ly["d"]
            for i in range(Ly["n"]):
                th = 2 * math.pi * i / Ly["n"] + Ly["off"] + rng.normal(0, 0.10)
                aj, lj = rng.normal(0, 0.06), rng.uniform(0.92, 1.08)
                pj, wj = rng.random(), rng.uniform(-0.5, 0.5)
                u, v = area_u(rng, d), sample_v(rng, d)
                col = palette_pick(rng, Ly["pal"], u + rng.normal(0, 0.10, d))
                sel = (np.abs(v) > 0.86) & (rng.random(d) < 0.55)
                col[sel] = palette_pick(rng, RIM_PAL, rng.random(int(sel.sum())))
                hl = rng.random(d) < 0.04
                col[hl] = palette_pick(rng, HI_PAL, rng.random(int(hl.sum())))
                fl = (rng.random(d) < 0.02) & (u < 0.55)
                col[fl] = palette_pick(rng, GOLD_PAL, rng.random(int(fl.sum())))
                f = lambda val: np.full(d, val, np.float32)
                parts.append(dict(
                    th=f(th), aj=f(aj), pj=f(pj), wj=f(wj), u=u.astype(np.float32), v=v.astype(np.float32),
                    ac=f(Ly["ac"]), ao=f(Ly["ao"]), kc=f(Ly["kc"]), ko=f(Ly["ko"]),
                    L=f(Ly["L"] * lj), wc=f(Ly["wc"]), wo=f(Ly["wo"]),
                    live=col, wilt=palette_pick(rng, WILT_PAL, 0.4 * u + 0.6 * rng.random(d), spread=0.7),
                    rn=rng.random(d).astype(np.float32),
                    rad=rng.choice([1.0, 1.5, 2.0], d, p=[0.3, 0.45, 0.25])))
        h = cat(parts)
        h["sp"] = profile(h["u"]).astype(np.float32)
        self.head = permute(h, rng)
```

No final tudo é juntado com `cat` e embaralhado com `permute`.

### Passo 11 · Estames, folhas e efeitos

Os **estames** são os filamentos finos no centro da flor, cada um terminando num pontinho maior (`rad = 2`):

```python
    def build_stamens(self):
        rng, n, nf = self.rng, 700, 60
        fid = rng.integers(0, nf, n)
        t = 1 - rng.random(n) ** 1.6
        live = palette_pick(rng, STAMEN_PAL, t, spread=0.35)
        self.st = permute(dict(
            th=rng.uniform(0, 2 * math.pi, nf)[fid].astype(np.float32),
            beta=rng.uniform(0.2, 0.85, nf)[fid].astype(np.float32),
            ell=rng.uniform(0.30, 0.55, nf)[fid].astype(np.float32),
            t=t.astype(np.float32), live=live,
            wilt=palette_pick(rng, STAMEN_WILT, rng.random(n), spread=0.6),
            rad=np.where(t > 0.88, 2.0, rng.choice([1.0, 1.5], n)).astype(np.float32)), rng)
```

As **folhas**: 900 pontos para cada uma das 4 folhas, com uma nervura central mais clara (`RIB_PAL`):

```python
    def build_leaves(self):
        rng, parts = self.rng, []
        d = 900
        for j in range(len(LEAF_DEFS)):
            u, v = area_u(rng, d), sample_v(rng, d, 0.10)
            rib = rng.random(d) < 0.09
            v[rib] = rng.normal(0, 0.03, int(rib.sum()))
            live = palette_pick(rng, LEAF_PAL, u + rng.normal(0, 0.1, d))
            live[rib] = palette_pick(rng, RIB_PAL, rng.random(int(rib.sum())))
            parts.append(dict(
                lid=np.full(d, j), u=u.astype(np.float32), v=v.astype(np.float32), live=live,
                wilt=palette_pick(rng, LEAF_WILT, rng.random(d), spread=0.8),
                rn=rng.random(d).astype(np.float32),
                rad=rng.choice([1.0, 1.5, 2.0], d, p=[0.35, 0.45, 0.2])))
        self.leaf = permute(cat(parts), rng)
```

Os **efeitos**: um halo suave ao redor da flor, um desenho de **7 círculos** (um central e seis ao redor, como a "flor da vida") que gira devagar, faíscas que sobem, e as pétalas que caem quando a flor murcha:

```python
    def build_fx(self):
        rng = self.rng
        n = 1800
        r = np.minimum(np.abs(rng.normal(0, 0.55, n)), 1.6)
        self.halo = permute(dict(
            a=rng.uniform(0, 2 * math.pi, n), r=r,
            b=((1 - r / 1.6) ** 1.5 * 0.55).astype(np.float32),
            fl=rng.random(n).astype(np.float32)), rng)
        cf, cj, rho, a = [], [], [], []
        for j in range(7):
            m = 700 if j == 0 else 170
            cf += [0.0 if j == 0 else 0.5] * m
            cj += [max(j - 1, 0)] * m
            rho += [1.0 if j == 0 else 0.5] * m
            a += list(rng.uniform(0, 2 * math.pi, m))
        self.ring = permute(dict(cf=np.array(cf), cj=np.array(cj), rho=np.array(rho), a=np.array(a),
                                 fl=rng.random(len(a))), rng)
        m = 280
        self.spark = dict(
            a=rng.uniform(0, 2 * math.pi, m), rho=rng.random(m), ph=rng.random(m),
            sp=rng.uniform(0.6, 1.4, m), rad=rng.choice([1.0, 2.0], m, p=[0.7, 0.3]),
            hue=rng.random(m))
        m = 260
        xl = self.cx + rng.normal(0, 0.12 * self.H * self.scale, m)
        self.fall = dict(
            r0=rng.random(m), ox=rng.normal(0, 1, m), oy=rng.normal(0, 1, m),
            xl=xl, yl=self.surface(xl) - rng.random(m) * 0.012 * self.H,
            ph=rng.uniform(0, 6.28, m),
            col=palette_pick(rng, WILT_PAL, rng.random(m), spread=0.8),
            rad=rng.choice([1.5, 2.0], m))
```


> ✅ **Checkpoint:** rode `python -c "import jardim"`. Se não aparecer erro, está tudo certo até aqui.



### Passo 12 · Desenhando um quadro: `render`

Agora a parte final da `Plant`. O `render(t)` pega o estado do instante `t` e **posiciona todos os pontos**. Vamos escrevê-lo em blocos, todos dentro do mesmo método.

**12a. Início, atraso e caule.** Três ideias aqui:

1. `t = t - self.delay`: com atraso de 5 s, quando o relógio global marca 5 s a flor vê `t = 0` e começa a nascer.
2. `canvas=None`: se receber uma imagem, desenha nela, e assim várias flores dividem o mesmo quadro. Se não, cria uma cópia do solo.
3. `Hs = H * scale`: espessuras e detalhes usam a altura **da flor** e não da tela, para uma flor pequena não ter caule grosso demais.

Aqui também aparece o balanço do caule (`sway`) e o crescimento: só os primeiros `nv` pontos do caule são usados, e `nv` cresce com `stem`:

```python
    def render(self, t, canvas=None):
        H, W = self.H, self.W
        Hs = H * self.scale          # altura "da flor" (escala espessuras e detalhes)
        t = t - self.delay           # atraso individual
        stem, size, bloom, wilt, leaf, glow = state(t)
        if canvas is None:
            canvas = self.base.copy()
        B = Batch()
        nodes = Batch()
        Lf = self.Lf

        L = self.Ls * stem
        sway = 0.012 * Hs * math.sin(t * 0.9) * stem * (1 - 0.6 * wilt)
        P = self.ctrl(L, wilt, sway)
        hx, hy = P[3]
        if stem > 0.01:
            S = self.stem
            nv = int(len(S["s"]) * np.clip(stem, 0.08, 1))
            s = S["s"][:nv]
            x, y, dx, dy = bez(P, s)
            nn = np.hypot(dx, dy) + 1e-6
            wst = 0.0085 * Hs * (1 - 0.3 * s) * (1 - 0.25 * wilt) * np.clip(0.4 + stem * 1.2, 0.4, 1)
            lat = S["lat"][:nv] * wst
            wi = np.clip(wilt * (0.75 + 0.5 * s) * (0.7 + 0.5 * S["rn"][:nv]), 0, 1)
            B.add(x - dy / nn * lat, y + dx / nn * lat,
                  mixcol(S["live"][:nv], S["wilt"][:nv], wi), S["rad"][:nv], np.full(nv, -500.0))
```

**12b. Folhas.** Mesma lógica: os pontos são posicionados ao longo de um arco, ancorados em pontos do caule, e as folhas vão ficando maiores e caídas conforme `leaf` e `wilt`:

```python
        if leaf > 0.02 and stem > 0.05:
            S = self.leaf
            nv = int(len(S["u"]) * np.clip(leaf, 0.15, 1))
            sa = np.array([d[0] for d in LEAF_DEFS])
            ax, ay, _, _ = bez(P, sa)
            lid = S["lid"][:nv]
            m = np.array([d[1] for d in LEAF_DEFS], np.float32)[lid]
            lfac = np.array([d[2] for d in LEAF_DEFS], np.float32)[lid]
            a0 = np.array([d[3] for d in LEAF_DEFS], np.float32)[lid] + (1 - leaf) * 0.8 - 0.6 * wilt
            c = np.array([d[4] for d in LEAF_DEFS], np.float32)[lid] - 0.5 * wilt
            u, v = S["u"][:nv], S["v"][:nv]
            LL = 0.19 * Hs * lfac * leaf * (1 - 0.25 * wilt)
            am = a0 + c * u * 0.5
            al = a0 + c * u
            sc = sinc(c * u * 0.5)
            lat = v * 0.27 * LL * profile(u)
            X = ax[lid] + LL * u * m * np.cos(am) * sc - lat * np.sin(al)
            Yup = LL * u * np.sin(am) * sc + lat * m * np.cos(al)
            wi = np.clip(wilt * (0.9 + 0.5 * u + 0.3 * S["rn"][:nv]) - 0.1, 0, 1)
            B.add(X, ay[lid] - Yup, mixcol(S["live"][:nv], S["wilt"][:nv], wi),
                  S["rad"][:nv], np.full(nv, -450.0))
```

**12c. Pétalas (a cabeça).** Cada ponto da pétala é posicionado com um arco de curvatura constante (por isso o `sinc`). A função interna `project` faz uma projeção 3D simples: inclina a cabeça (`tau`) e a gira conforme murcha (`phi`), devolvendo `(x, y)` na tela e a profundidade `zp`. Essa profundidade serve para ordenar os pontos e para sombrear: pétalas de trás ficam mais escuras.

```python
        if stem > 0.01 and size > 0.05:
            phi = wilt * 2.75 + 0.035 * math.sin(t * 0.9 + 1.0) * (1 - wilt)
            tau = 0.20 + 0.36 * bloom - 0.40 * wilt
            cphi, sphi, ct, st_ = math.cos(phi), math.sin(phi), math.cos(tau), math.sin(tau)
            Ls_ = Lf * size

            def project(xl, yl, zl):
                yp = yl * ct - zl * st_
                zp = yl * st_ + zl * ct
                return hx + xl * cphi + yp * sphi, hy - (-xl * sphi + yp * cphi), zp, yp

            vis = float(np.clip(size ** 2 * 1.15, 0.2, 1.0))
            h = self.head
            nv = int(len(h["u"]) * vis)
            g = {k: val[:nv] for k, val in h.items()}
            u, v = g["u"], g["v"]
            A0 = g["ac"] + (g["ao"] - g["ac"]) * bloom + g["aj"]
            A0 = A0 * (1 - 0.55 * wilt) + wilt * g["wj"] * 0.3
            K = g["kc"] + (g["ko"] - g["kc"]) * bloom + wilt * (0.6 + 0.6 * g["pj"])
            Lp = Ls_ * g["L"] * (1 - 0.28 * wilt * g["pj"])
            am = A0 + K * u * 0.5
            sc = sinc(K * u * 0.5)
            rr = Lp * u * np.sin(am) * sc
            hh = Lp * u * np.cos(am) * sc
            lat = v * (g["wc"] + (g["wo"] - g["wc"]) * bloom) * Lp * g["sp"]
            rr = rr + 0.025 * Ls_ - 0.5 * lat * lat / (Lp + 1.0)
            cth, sth = np.cos(g["th"]), np.sin(g["th"])
            X, Y, zp, yp = project(cth * rr - sth * lat, hh, sth * rr + cth * lat)
            shade = np.clip(0.72 + 0.28 * np.clip(zp / Ls_ * 0.5 + 0.5, 0, 1) + 0.12 * (hh / Ls_), 0.5, 1.2)
            wi = np.clip(wilt * (1.0 + 0.7 * u + 0.4 * g["rn"]) - 0.2, 0, 1)
            col = mixcol(g["live"], g["wilt"], wi) * shade[:, None]
            B.add(X, Y, col, g["rad"], zp)
```

**12d. Estames.**

```python
            ss = float(smoothstep(bloom, 0.30, 0.85))
            if ss > 0.02:
                S = self.st
                nv2 = int(len(S["t"]) * np.clip(size ** 2, 0.25, 1))
                g = {k: val[:nv2] for k, val in S.items()}
                ell = Ls_ * ss * g["ell"]
                beta = g["beta"] * (0.6 + 0.4 * bloom) + wilt * 0.3
                r = ell * g["t"] * np.sin(beta) + 0.02 * Ls_
                yl = 0.05 * Ls_ + ell * g["t"] * np.cos(beta)
                X, Y, zp, _ = project(np.cos(g["th"]) * r, yl, np.sin(g["th"]) * r)
                w = np.full(nv2, wilt, np.float32)
                B.add(X, Y, mixcol(g["live"], g["wilt"], w), g["rad"], zp + 3.0)
```

**12e. Brilho (halo, círculos e faíscas).** Só aparece quando `glow` está ativo:

```python
            if glow > 0.02:
                axis = np.array([math.sin(phi), -math.cos(phi)])
                ccx, ccy = hx + axis[0] * 0.5 * Ls_, hy + axis[1] * 0.5 * Ls_
                Hh = self.halo
                nh = int(len(Hh["a"]) * glow)
                rh = Hh["r"][:nh] * Ls_ * 1.1
                br = Hh["b"][:nh] * glow * (0.85 + 0.3 * np.sin(t * 1.3 + Hh["fl"][:nh] * 6.28))
                B.add(ccx + np.cos(Hh["a"][:nh]) * rh, ccy + np.sin(Hh["a"][:nh]) * rh,
                      np.asarray([150, 70, 240], np.float32)[None, :] * br[:, None],
                      np.ones(nh), np.full(nh, -2000.0))

                R = self.ring
                nr = int(len(R["a"]) * glow)
                Rs = Ls_ * 1.25
                cang = 2 * math.pi * R["cj"][:nr] / 6 + t * 0.15
                a = R["a"][:nr] + t * 0.05
                rx = ccx + Rs * (R["cf"][:nr] * np.cos(cang) + R["rho"][:nr] * np.cos(a))
                ry = ccy + Rs * (R["cf"][:nr] * np.sin(cang) + R["rho"][:nr] * np.sin(a))
                bb = (0.40 + 0.60 * glow) * (0.65 + 0.35 * np.sin(t * 2 + R["fl"][:nr] * 20))
                B.add(rx, ry, np.asarray([210, 145, 255], np.float32)[None, :] * bb[:, None],
                      np.ones(nr), np.full(nr, -1500.0))
                ja = 2 * math.pi * np.arange(6) / 6 + t * 0.15
                nodes.add(ccx + Rs * np.cos(ja), ccy + Rs * np.sin(ja),
                          np.tile(np.asarray([255, 215, 255], np.float32) * glow, (6, 1)),
                          np.full(6, 3.0), np.zeros(6))

                Sp = self.spark
                life = (t * 0.10 * Sp["sp"] + Sp["ph"]) % 1.0
                rho = (0.25 + 1.15 * Sp["rho"]) * Ls_ * (0.6 + 0.6 * life)
                px = ccx + np.cos(Sp["a"]) * rho
                py = ccy + np.sin(Sp["a"]) * rho - life * 0.25 * Ls_
                bs = (np.sin(np.pi * life) ** 0.8 * glow)[:, None]
                cs = np.where(Sp["hue"][:, None] < 0.5, np.asarray([255, 215, 255], np.float32),
                              np.asarray([255, 150, 235], np.float32))
                B.add(px, py, cs * bs, Sp["rad"], np.full(len(px), 2000.0))
```

**12f. Pétalas caindo.** Quando a flor murcha, parte dos pontos sai da cabeça e cai até o chão em curva:

```python
            if wilt > 0.02:
                F = self.fall
                fp = np.clip((wilt * 1.05 - F["r0"] * 0.7) / 0.35, 0, 1)
                mk = fp > 0
                if mk.any():
                    f = fp[mk]
                    sx = hx + F["ox"][mk] * 0.18 * Ls_
                    sy = hy + F["oy"][mk] * 0.18 * Ls_ + 0.25 * Ls_
                    x = sx + (F["xl"][mk] - sx) * f + np.sin(f * 6.28 + F["ph"][mk]) * 0.03 * Hs * (1 - f)
                    y = sy + (F["yl"][mk] - sy) * f * f
                    B.add(x, y, F["col"][mk], F["rad"][mk], np.full(int(mk.sum()), 500.0))
```

**12g. Finalização.** Desenhamos tudo ordenado por profundidade com `B.draw`, adicionamos os nós brilhantes do hexágono e, **só na flor principal**, as rachaduras do solo. Por fim devolvemos a imagem:

```python
        B.draw(canvas)
        if nodes.p:
            x, y, col, rad, _ = [np.concatenate([q[i] for q in nodes.p]) for i in range(5)]
            splat(canvas, x, y, np.clip(col, 0, 255).astype(np.uint8), rad, nmax=29)

        if self.cr is not None:
            C = self.cr
            e = np.clip((t - C["delay"]) / 3.0, 0, 1)
            sy = C["y"] - C["amp"] * np.sin(np.pi * e)
            splat(canvas, C["x"], sy, np.clip(C["col"], 0, 255).astype(np.uint8), C["rad"])
        return canvas
```

> ✅ **Teste a flor sozinha!** Crie um arquivo `teste.py` ao lado do `jardim.py`:
>
> ```py
> from jardim import Plant
> from PIL import Image
>
> p = Plant(800, 800)
> Image.fromarray(p.render(26.0)).save("teste.png")
> ```
>
> Rode `python teste.py` e abra `teste.png`: você deve ver **uma flor no auge**, plantada no solo. Troque o `26.0` por `5.0`, `16.0` ou `41.0` para ver outras fases.

### Passo 13 · O jardim: `Garden`

Com a `Plant` pronta, o jardim é simples. Ele cria a flor principal, depois cria as demais passando `ground=main.surface` (para se plantarem no mesmo solo), ordena por tamanho para que as menores fiquem atrás, e desenha todas no mesmo canvas:

```python
class Garden:
    """Várias flores no mesmo canvas, compartilhando o solo da flor principal."""

    def __init__(self, W, H):
        fx, sc, dl, sd = FLOWERS[0]
        main = Plant(W, H, cx=fx * W, scale=sc, seed=sd, delay=dl)
        plants = [main] + [
            Plant(W, H, cx=fx * W, scale=sc, seed=sd, delay=dl, ground=main.surface)
            for fx, sc, dl, sd in FLOWERS[1:]
        ]
        self.main = main
        self.plants = sorted(plants, key=lambda p: p.scale)  # menores atrás, maiores na frente
        self.end = T_END + max(p.delay for p in plants)

    def render(self, t):
        canvas = self.main.base.copy()
        for p in self.plants:
            p.render(t, canvas)
        return canvas
```

- `self.end` é a duração total: o tempo de uma flor **mais** o maior atraso. Sem isso, o GIF e o loop cortariam a última flor antes de murchar.
- `render(t)` começa com a cópia do solo da flor principal e pede a cada `Plant` que desenhe nele.

### Passo 14 · Exportando imagens e GIF

Duas funções para gravar o resultado em arquivo. `save_png` salva 6 quadros em momentos-chave e monta uma faixa com todos (`flower_stages.png`). `save_gif` renderiza a animação inteira a 12 quadros por segundo:

```python
def save_png():
    H = 1000
    plant = Garden(H, H)
    times = [3.0, 9.0, 16.0, 26.0, 41.0, 52.0]
    frames = []
    for i, tt in enumerate(times):
        a = plant.render(tt)
        Image.fromarray(a).save(f"flower_{i + 1}.png")
        frames.append(Image.fromarray(a).resize((500, 500), Image.LANCZOS))
    strip = Image.new("RGB", (500 * len(frames), 500))
    for i, f in enumerate(frames):
        strip.paste(f, (500 * i, 0))
    strip.save("flower_stages.png")
    print("Saved: flower_stages.png and stage frames")
```

```python
def save_gif(fps=12, size=640):
    plant = Garden(size, size)
    frames = [Image.fromarray(plant.render(i / fps)).convert("P", palette=Image.ADAPTIVE)
              for i in range(int(plant.end * fps) + 1)]
    frames[0].save("flower.gif", save_all=True, append_images=frames[1:],
                   duration=int(1000 / fps), loop=0, optimize=False)
    print("Saved: flower.gif")
```

### Passo 15 · A janela animada

A classe `App` abre uma janela Tkinter, mantém um relógio e, a cada `TICK_MS` milissegundos, renderiza o quadro do instante atual e o mostra na tela. Atalhos: **espaço** pausa, **R** reinicia, **S** salva o quadro atual, **Esc** fecha.

```python
class App:
    def __init__(self):
        import tkinter as tk
        from PIL import ImageTk

        self.root = tk.Tk()
        self.root.title("Pointillism Garden")
        self.root.configure(bg="black")
        sw, sh = self.root.winfo_screenwidth(), self.root.winfo_screenheight()
        H = int(min(sh * WIN_FRAC, sw * 0.96))
        W = H
        self.root.geometry(f"{W}x{H}+{(sw - W) // 2}+{max(0, (sh - H) // 2 - 20)}")
        self.cv = tk.Canvas(self.root, width=W, height=H, bg="black", highlightthickness=0)
        self.cv.pack(fill="both", expand=True)
        self.cv.create_text(W // 2, H // 2, text="preparing...", fill="#777777")
        self.root.update()

        self.plant = Garden(W, H)
        self.photo = ImageTk.PhotoImage(Image.fromarray(self.plant.render(0)))
        self.cv.delete("all")
        self.cv.create_image(0, 0, anchor="nw", image=self.photo)
        self.root.bind("<Escape>", lambda e: self.root.destroy())
        self.root.bind("<space>", lambda e: self.toggle())
        self.root.bind("<r>", lambda e: self.restart())
        self.root.bind("<s>", lambda e: Image.fromarray(self.plant.render(self.clock / TIME_SCALE)).save("flower_frame.png"))
        self.clock, self.last, self.paused = 0.0, time.perf_counter(), False
        self.tick()
```

```python
    def toggle(self):
        self.paused = not self.paused

    def restart(self):
        self.clock = 0.0

    def tick(self):
        now = time.perf_counter()
        if not self.paused:
            self.clock += now - self.last
        self.last = now
        t = self.clock / TIME_SCALE
        if LOOP and t > self.plant.end + 4:
            self.clock, t = 0.0, 0.0
        self.photo.paste(Image.fromarray(self.plant.render(t)))
        self.root.after(TICK_MS, self.tick)

    def run(self):
        self.root.mainloop()
```

### Passo 16 · O ponto de entrada

Por último, decidimos o que fazer conforme os argumentos da linha de comando:

```python
if __name__ == "__main__":
    if "--png" in sys.argv:
        save_png()
    elif "--gif" in sys.argv:
        save_gif()
    else:
        App().run()
```

> ✅ **Checkpoint:** rode `python -c "import jardim"`. Se não aparecer erro, está tudo certo até aqui.

Pronto! O código está completo.

---

## Parte 3 · Rodando

```bash
python jardim.py          # abre a janela animada
python jardim.py --png    # gera flower_1.png ... flower_6.png e flower_stages.png
python jardim.py --gif    # gera flower.gif (demora um pouco)
```

| Tecla | O que faz |
|-------|-----------|
| `Espaço` | pausa / continua |
| `R` | reinicia a animação |
| `S` | salva o quadro atual em `flower_frame.png` |
| `Esc` | fecha a janela |

> **Dica:** na primeira vez, rode `--png`. É a forma mais rápida de ver se tudo funcionou e não depende do Tkinter.

---

## Parte 4 · Personalizando

### Mudar o arranjo do jardim

Tudo está na lista `FLOWERS`:

- **Todas nascem juntas:** coloque `0.0` em todos os atrasos.
- **Fileira regular:**

  ```py
  FLOWERS = [(0.15 + 0.175 * i, 0.7, 2.0 * i, 10 + i) for i in range(5)]
  ```

- **Posições aleatórias:**

  ```py
  import random
  FLOWERS = [(random.uniform(0.15, 0.85), random.uniform(0.5, 1.0),
              random.uniform(0, 10), i) for i in range(6)]
  ```

  Lembre-se: o **primeiro** item vira a flor principal e cria o solo.

### Mudar o visual

- **Cores:** edite as paletas (`LAYERS`, `LEAF_PAL`, `STEM_PAL`, `RIM_PAL`…).
- **Número de pétalas:** em `LAYERS`, o campo `n`. O campo `d` controla a densidade de pontos.
- **Duração das fases:** a primeira coluna de `KEYS`.
- **Tamanho do brilho:** em `render`, mude `Rs = Ls_ * 1.25`. Valores menores deixam os círculos mais compactos, o que ajuda quando as flores se sobrepõem.
- **Flor que nunca murcha:** troque o `murchar` das últimas linhas de `KEYS` para `0.0`.

---

## Parte 5 · Problemas comuns

**`ModuleNotFoundError: No module named 'tkinter'`**
Instale o Tkinter (veja a Parte 0). Enquanto isso, `--png` e `--gif` funcionam sem ele.

**A animação está lenta ou travando**
Cada flor extra é desenhada a cada quadro. Use menos flores na lista, aumente `TICK_MS` ou use `--png`.

**As flores ficam muito sobrepostas**
Afaste as posições x em `FLOWERS` ou reduza as escalas.

**Uma flor secundária aparece flutuando ou enterrada**
Ela precisa ser criada com `ground=main.surface`. O `Garden` já faz isso, então só aparece se você criar uma `Plant` secundária manualmente sem passar o chão.

**O GIF demora demais**
Reduza `size` ou `fps` em `save_gif(fps=12, size=640)`.

**Dá erro de indentação ao colar**
Os métodos da `Plant` precisam estar com a mesma indentação dentro da classe. Cole cada bloco exatamente como aparece, sem mexer nos espaços.

---

## Parte 6 · Ideias para continuar

- Adicionar vaga-lumes ou pólen flutuando entre as flores.
- Fazer o vento balançar todas as flores de forma sincronizada.
- Dar paletas diferentes a cada `Plant` para ter flores de várias cores.
- Exportar em vídeo com `imageio` ou `ffmpeg`.

Divirta-se! 
