# Tokens de UI e a arquitetura de camadas HTML sobre o canvas

Este documento descreve, com os valores reais, como o projeto de referência (Hub Sim Racing) monta uma interface
"wireframe técnico" em HTML/CSS puro por cima de canvases WebGL. Nada aqui é genérico: cada valor foi lido no código.

> **Convenção das referências.** `arquivo:linhas` aponta para o projeto de referência no commit `ff34fab` (versão 0.2.1),
> com o caminho a partir da raiz do projeto (`hub/web/...`). Se você copiar um trecho, copie também o comentário: os
> comentários do projeto explicam o porquê.

## 1. O que existe e o que não existe neste projeto

| Tópico | Situação |
|---|---|
| Paleta em variáveis CSS (`:root`) | Existe. Um único bloco, em `hub/web/css/tokens.css:4-30`. |
| Tema claro / escuro | **Não existe de verdade.** O hub é sempre escuro. O atributo `data-theme` ainda é gravado no `<html>` (`app.js:37`) e o seletor de tema existe nos Ajustes, mas nenhuma regra CSS varia com ele; o comentário do `tokens.css:1-3` diz isso. O `CLAUDE.md` do projeto ainda descreve três temas: está desatualizado. |
| Marcas em L nos cantos (mira de HUD) | Existe, em duas técnicas (seção 4). |
| Desfoque de fundo (`backdrop-filter`) | Existe: `--blur: blur(6px)`. |
| Glassmorphism "de verdade" (vidro fosco com raio, brilho de borda, sombra suave) | **Não existe neste projeto.** O que existe é fundo translúcido + `blur(6px)` + borda de 1 px, com `--radius: 0` e `--shadow: none`. O efeito de desfoque só é perceptível na barra do topo e nos diálogos; os cards ficam sobre seções de fundo opaco. |
| Sombras projetadas (CSS) | Quase não existem: `--shadow: none` nos cards. Só duas sombras reais: `.toast` (`0 6px 24px rgba(0,0,0,.18)`, `app.css:168`) e `dialog.modal` (`0 20px 60px rgba(0,0,0,.25)`, `app.css:171`). |
| Fontes externas | Não. `--font` e `--mono` são fontes do sistema. Um teste (`tests/test_web_static.py::test_no_external_resources`) proíbe recursos externos. |
| Framework CSS/JS | Não. CSS escrito à mão, JS de módulos ES. |

## 2. Paleta exata (todos os tokens)

Fonte: `hub/web/css/tokens.css:4-30`. Um único `:root`, `color-scheme: dark`.

### 2.1 Superfícies

| Token | Valor | Para quê |
|---|---|---|
| `--bg-base` | `#070707` | Sólido, no `html` (`app.css:3`): a camada mais ao fundo. |
| `--bg` | `rgba(10, 10, 10, .70)` | Translúcido, no `body` (`app.css:12`). |
| `--panel` | `rgba(255, 255, 255, .025)` | Cards e quadros: quase transparentes. |
| `--panel-solid` | `rgba(12, 12, 12, .94)` | Diálogos, menu, ferramentas do hero: legíveis por cima do 3D. |
| `--side` | `rgba(10, 10, 10, .55)` | Barra de título própria (só na janela do pywebview). |
| `--sec-edge` | `#000` | Fundo da primeira e da última seção. |
| `--sec-mid` | `#1c1c1c` | O cinza da seção do meio (bem perto do preto). |
| `--sec-grad` | `linear-gradient(180deg, #000 0%, #3a3a3a 50%, #000 100%)` | Opção "degradê dentro de cada seção" (`html.bg-cada`). |

### 2.2 Texto, linhas e acento

| Token | Valor |
|---|---|
| `--ink` | `#E8E8E5` |
| `--ink-2` | `#A9A9A5` |
| `--ink-3` | `#6F6F6B` |
| `--line` | `rgba(255, 255, 255, .10)` |
| `--line-2` | `rgba(255, 255, 255, .05)` |
| `--corner` | `rgba(255, 255, 255, .55)` (marcas em L) |
| `--accent` | `#EDEDEA` |
| `--accent-ink` | `#0A0A0A` (texto sobre o acento) |

### 2.3 Estados

| Token | Valor | Fundo associado |
|---|---|---|
| `--ok` | `#7FB487` | `--ok-bg: rgba(127, 180, 135, .12)` |
| `--warn` | `#E0B45E` | `--warn-bg: rgba(224, 180, 94, .12)` |
| `--err` | `#F08A80` | `--err-bg: rgba(240, 138, 128, .12)` |

Regra de design do projeto: o verde só significa "conectado" (ou "peça acesa" no 3D, onde `GLOW = 0x7fb487` é o mesmo
verde). O resto é monocromático.

### 2.4 Tokens de componentes específicos

| Token | Valor | Usado em |
|---|---|---|
| `--bt-hit-bg` | `rgba(237, 237, 234, .14)` | Linha da tabela do botão apertado agora. |
| `--rpm-off` | `rgba(255, 255, 255, .07)` | Segmento apagado da barra de RPM. |
| `--rpm-on` | `#E8E8E5` | Segmento aceso. |
| `--cal-grid` | `rgba(255, 255, 255, .08)` | Grade do gráfico de calibração. |
| `--cal-band` | `rgba(237, 237, 234, .10)` | Faixa gravada. |
| `--cal-handle` | `#EDEDEA` | Alças e linhas. |
| `--pedal-gas` | `#7FB487` | Barra do acelerador. |
| `--pedal-brake` | `#F08A80` | Barra do freio. |
| `--pedal-clutch` | `#7FA8D0` | Barra da embreagem. |

### 2.5 Forma, desfoque, medidas e movimento

| Token | Valor |
|---|---|
| `--radius` | `0` (quinas retas) |
| `--radius-sm` | `2px` (botões, campos) |
| `--shadow` | `none` |
| `--blur` | `blur(6px)` |
| `--topbar-h` | `44px` |
| `--ease` | `cubic-bezier(.2, .7, .2, 1)` |

### 2.6 Os tokens do painel de telemetria (segundo sistema, mesmo vocabulário)

O painel de celular/tablet (`hub/web/painel/painel-tokens.css`) tem seus próprios tokens `--p-*`, repetindo os valores do
hub no estilo "wireframe" e acrescentando mais três estilos (LCD âmbar, Neon, Analógico). Pontos que valem copiar:

- O bloco `:root, [data-estilo="wireframe"]` é o padrão; `[data-estilo="lcd"]`, `"neon"` e `"analogico"` só redefinem
  variáveis. O `<html>` leva `data-estilo="..."` e todo o CSS usa `var(--p-...)` (`painel-tokens.css:1-3`).
- Os widgets de canvas pedem cor por **nome de tom** (`data-tom="ok|warn|err|info|neutral"`), e o ajudante
  `widgets/canvas.js` lê o valor com `getComputedStyle` e guarda por estilo (`canvas.js:20-32`). Nenhuma cor literal no JS.
- Valores do estilo base (todos): `--p-bg #070707`, `--p-panel rgba(255,255,255,.025)`, `--p-ink #E8E8E5`,
  `--p-ink-2 #A9A9A5`, `--p-ink-3 #6F6F6B`, `--p-line rgba(255,255,255,.10)`, `--p-corner rgba(255,255,255,.55)`,
  `--p-ok #7FB487`, `--p-warn #E0B45E`, `--p-err #F08A80`, `--p-info #7FA8D0`, `--p-led-off rgba(255,255,255,.07)`,
  `--p-overlay rgba(12,12,12,.94)`, `--p-corner-size 8px`, `--p-radius 0`, `--p-gap 6px`.

## 3. Tipografia

Fontes (sistema, sem download): `tokens.css:26-27`

```css
--font: "Segoe UI Variable Text", "Segoe UI", system-ui, sans-serif;
--mono: "Cascadia Mono", Consolas, ui-monospace, monospace;
```

Escala realmente usada (`hub/web/css/app.css`):

| Elemento | Declaração | Linha |
|---|---|---|
| `body` | `14px/1.45 var(--font)`, `-webkit-font-smoothing: antialiased` | 12 |
| `.brand` | `600 13px`, `letter-spacing: .02em` | 35 |
| `.hero h1` | `300 clamp(40px, 6vw, 76px)/1`, `letter-spacing: -.03em` | 59 |
| `.hero-kicker` | `11px mono`, `letter-spacing: .1em`, maiúsculas, `--ink-3` | 58 |
| `.hero-sub` | `12px mono`, `--ink-2` | 60 |
| `.hero-field small` | `10px mono`, `.1em`, maiúsculas, `--ink-3` | 61 |
| `.hero-field b` (leitura grande) | `400 26px/1.1 mono`, `font-variant-numeric: tabular-nums` | 62 |
| `.sec .page-head h1` | `300 clamp(30px, 4vw, 46px)/1.05`, `letter-spacing: -.02em` | 76 |
| `.card h2` | `500 11px/1.4 mono`, `.08em`, maiúsculas, `--ink-3` | 122 |
| `.tile .k` | `11px mono`, `.08em`, maiúsculas | 125 |
| `.tile .v` | `500 26px/1.3 mono`, `tabular-nums` | 126 |

Padrão visual: **títulos finos e grandes (peso 300) em sans, rótulos pequenos em mono maiúsculo com espaçamento, números
sempre em mono com `tabular-nums`** (evita o texto "tremer" quando o valor muda a 30 Hz).

## 4. Bordas, marcas em L, sombras e desfoque

### 4.1 Marcas em L (mira de HUD) nos cantos do hub

`hub/web/css/app.css:108-120`: oito traços de 10 px desenhados como **fundo do próprio elemento**, sem pseudo-elementos:

```css
.card, .tile, dialog.modal {
  background-image:
    linear-gradient(var(--corner), var(--corner)), /* ...oito vezes, um por traço... */
  background-size: 10px 1px, 1px 10px, 10px 1px, 1px 10px, 10px 1px, 1px 10px, 10px 1px, 1px 10px;
  background-position: left top, left top, right top, right top, left bottom, left bottom, right bottom, right bottom;
  background-repeat: no-repeat;
  background-origin: border-box;
}
```

Detalhe importante: `background-origin: border-box` faz os traços encostarem na **borda** (não no padding). E por usar
`background-image`, a cor de fundo do componente vai em `background-color` (`.card { background-color: var(--panel) }`,
`app.css:121`), não no atalho `background`, senão o atalho apagaria as marcas.

### 4.2 Marcas em L no painel (outra técnica)

`hub/web/painel/painel.css:22-26`: só dois cantos (superior esquerdo e inferior direito), com `::before`/`::after` de
`var(--p-corner-size)` (8 px; 0 nos estilos Neon e Analógico, que não têm marcas):

```css
.w::before, .w::after { content: ""; position: absolute; width: var(--p-corner-size); height: var(--p-corner-size); pointer-events: none; }
.w::before { top: -1px; left: -1px; border-top: 1px solid var(--p-corner); border-left: 1px solid var(--p-corner); }
.w::after { bottom: -1px; right: -1px; border-bottom: 1px solid var(--p-corner); border-right: 1px solid var(--p-corner); }
```

### 4.3 Bordas

- Padrão: `1px solid var(--line)` (`.card`, `.tile`, `.btn`, `.chip`, `.seg`).
- Divisões internas: `1px solid var(--line-2)` (mais fraca).
- Pílulas: `border-radius: 999px` em `.status a`, `.chip`, `.update-notice`; círculo (`50%`) em `.dot` e no cursor.
- Todo o resto usa `--radius: 0` ou `--radius-sm: 2px`.
- Foco de teclado: `:focus-visible { outline: 2px solid var(--ink-2); outline-offset: 2px; }` (`app.css:190`).

### 4.4 Desfoque (`backdrop-filter`) e transparência

- `.topbar`: `background: rgba(7, 7, 7, .78); backdrop-filter: var(--blur);` (`app.css:32-33`). É onde o desfoque aparece
  de fato, porque o conteúdo rola por baixo.
- `.card` e `.tile`: `background-color: var(--panel); backdrop-filter: var(--blur);` (`app.css:121,124`).
- `dialog.modal`: `background-color: var(--panel-solid); backdrop-filter: var(--blur);` e
  `dialog.modal::backdrop { background: rgba(0, 0, 0, .55) }` (`app.css:171-172`).

### 4.5 Movimento CSS (só o essencial)

- `@keyframes fade` (`from { opacity: 0; transform: translateY(4px) } to { opacity: 1; transform: none }`) com `.18s var(--ease)`
  em `.page-inner` e `.toast` (`app.css:101-102`).
- Transições de `.15s var(--ease)` em item de menu, botões e chip.
- `@media (prefers-reduced-motion: reduce)` zera animações e transições (`app.css:204-206`).

## 5. Arquitetura de camadas: HTML sobre o canvas

### 5.1 A casca (`hub/web/index.html:12-39`)

```html
<body class="readonly">
<header class="titlebar pywebview-drag-region" id="titlebar" hidden>...</header>   <!-- só no pywebview sem borda -->
<header class="topbar" id="topbar">...</header>                                     <!-- barra fina fixa -->
<nav class="menu" id="menu" aria-label="Seções" data-lenis-prevent hidden></nav>
<section class="hero" id="hero" aria-label="Volante">
  <div class="hero-stage"></div>   <!-- aqui entra o <canvas> do Three.js -->
  <div class="hero-left"></div>
  <div class="hero-right"></div>
  <div class="hero-tools"></div>
</section>
<main class="sections" id="page" tabindex="-1"></main>                              <!-- seções empilhadas -->
<div class="cursor" id="cursor" aria-hidden="true"></div>
<div class="toasts" id="toasts" role="status" aria-live="polite"></div>
<script type="importmap">{"imports": {"three": "/vendor/three/three.module.js", "three/addons/": "/vendor/three/addons/", "lenis": "/vendor/lenis/lenis.mjs"}}</script>
<script type="module" src="/js/app.js"></script>
```

Observe: `importmap` aponta só para `/vendor/...` local. É isso que permite `import * as THREE from "three"` sem bundler.
`body.readonly` começa ligado e o `app.js` o desliga quando o servidor diz que o cliente é admin (`app.js:49`).

### 5.2 Pilha de camadas (z-index, de baixo para cima)

| Camada | Posição | `z-index` | Onde |
|---|---|---|---|
| `html` (fundo `--bg-base`) e `body` (`--bg`, translúcido) | normal | n/a | `app.css:3,12` |
| `.hero` (volante 3D) | `sticky; top: var(--topbar-h)` | **0** | `app.css:50` |
| `.hero-left`, `.hero-right` (leituras) | `relative` dentro do hero | 1 | `app.css:55` |
| `.hero-tools` (barra do editor) | `absolute` | 2 | `app.css:63` |
| `.sections` / `.sec` (painéis) | `relative` / `sticky` no desktop | **1** | `app.css:67-72` |
| `.menu` (hambúrguer) | `fixed` | 29 | `app.css:45` |
| `.topbar` | `fixed` | 30 | `app.css:32` |
| `.titlebar` (pywebview) | `fixed` | 31 | `app.css:29` |
| `.sheet` (folha em tela cheia) | `fixed; inset: 0` | 40 | `app.css:333` |
| `.toasts` | `fixed` | 50 | `app.css:167` |
| `.cursor` (ponto) | `fixed` | 100 | `app.css:81` |
| `dialog.modal` | camada superior do navegador (`showModal()`) | n/a | `ui.js` |

### 5.3 `pointer-events` e transparência

- O canvas do hero cobre a área inteira: `.hero-stage { position: absolute; inset: 0 }` e
  `.hero-stage canvas { display: block; width: 100%; height: 100% }` (`app.css:53-54`).
- As leituras que ficam por cima do canvas (`.hero-left`, `.hero-right`) têm `pointer-events: none` (`app.css:55`). Sem isso
  elas engoliriam o arrasto do OrbitControls. Elas ficam **na frente** do canvas só visualmente.
- A camada de rótulos presos ao modelo (`.viewer-labels`) é `position: absolute; inset: 0; pointer-events: none`, e só os
  chips ganham `pointer-events: auto` no modo de edição (`app.css:7-11`). Rótulos atrás de outra peça ficam com
  `opacity: .3` (`.chip.behind`).
- A sobreposição de escurecimento `.sections.stacked > .sec::after` é `pointer-events: none` (`app.css:73-74`).
- O `<canvas>` do Three.js é transparente: `new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true })`,
  `setClearColor(0x000000, 0)` e `scene.background = null` (`hub/web/js/webgl.js:112,117,120`). O fundo da página aparece
  através do canvas; a "iluminação" do conjunto vem da cena 3D, não de um fundo desenhado.
- Cursor próprio: com `html.has-cursor`, o cursor nativo some (`html.has-cursor, html.has-cursor * { cursor: none }`) e
  até no canvas (`html.has-cursor canvas { cursor: none !important }`, porque o OrbitControls escreve `cursor: auto`
  direto no canvas). Sobre controles o cursor nativo volta (`app.css:80-90`).

### 5.4 O hero fixo

`app.css:50-52`:

```css
.hero { position: sticky; top: var(--topbar-h); height: calc(100vh - var(--topbar-h)); z-index: 0; overflow: hidden;
  display: grid; grid-template-columns: minmax(170px, 1fr) minmax(0, 2.6fr) minmax(170px, 1fr); align-items: end; padding: 0 28px 34px; }
body.has-titlebar .hero { top: calc(var(--topbar-h) + 36px); height: calc(100vh - var(--topbar-h) - 36px); }
```

O hero é um `section` de altura de tela, **grudado** abaixo da barra. Como as seções têm `z-index: 1`, elas "sobem por
cima" dele ao rolar, e o hero parece ficar parado ao fundo. A grade de três colunas (`1fr 2.6fr 1fr`) serve para o texto
da esquerda, o canvas (que é `absolute` e ignora a grade) e as leituras da direita. O texto fica embaixo
(`align-items: end`).

### 5.5 Seções empilhadas (sticky) com `--cover`

Definição CSS (`app.css:67-74`):

```css
.sections { position: relative; z-index: 1; }
.sec { background: color-mix(in srgb, var(--sec-mid) calc(var(--t, 0) * 100%), var(--sec-edge));
  padding: 64px 28px 80px; min-height: 70vh; border-top: 1px solid var(--line); scroll-margin-top: var(--topbar-h); }
.sections.stacked > .sec { position: sticky; }
.sections.stacked > .sec::after { content: ""; position: absolute; inset: 0; background: #000; pointer-events: none;
  opacity: calc(var(--cover, 0) * .6); }
```

Como funciona (`hub/web/js/motion.js:19-37` e `109-123`):

1. `stack(sections)` liga a classe `stacked` só quando `matchMedia("(min-width: 761px)")` é verdadeiro.
2. Para cada `.sec`, o `top` do `sticky` vem do JS: `stickTop(altura, janela, barra)` = `Math.min(barra, janela - altura)`
   (`poses.js:109-111`). Seção curta gruda logo abaixo da barra; seção **mais alta que a tela** só gruda quando o
   rodapé dela chega ao fim da tela (top negativo), para nada ficar escondido pela seção seguinte.
3. Um `ResizeObserver` observa cada seção e recalcula o `top` quando a altura muda (`motion.js:25-30`); o `resize` da
   janela também.
4. `--cover` (0 a 1) é animado pelo GSAP **ligado à rolagem**: enquanto a seção seguinte sobe por cima, a anterior
   escurece até 60% de preto (`opacity: calc(var(--cover) * .6)`):

```js
gsap.fromTo(s, { "--cover": 0 }, { "--cover": 1, ease: "none",
  scrollTrigger: { trigger: next, start: "top bottom", end: () => `top ${topbar()}px`, scrub: true } });
```

Sem as bibliotecas (modo nativo), `--cover` nunca é definido, então vale o `0` do `var(--cover, 0)`: nada escurece. A
seção só ganha um fade de opacidade quando entra na tela (`html.motion-native`, `app.css:77-78`).

### 5.6 O fundo entre os painéis (`sectionTone` e `color-mix`)

Cada seção recebe `--t` (0 a 1) em `app.js:305`: `el.style.setProperty("--t", String(sectionTone(i, plan.length)))`.

`sectionTone` (`hub/web/js/poses.js:193-199`):

```js
export function sectionTone(i, n) {
  if (!Number.isInteger(i) || !Number.isInteger(n) || n <= 1 || i < 0 || i >= n) return 0;
  const t = Math.round(clamp(Math.sin(Math.PI * i / (n - 1)), 0, 1) * 1e4) / 1e4;
  return t === 0 ? 0 : t;                      // Math.round(-0) seria -0
}
```

Fórmula: `t = sin(π · i / (n − 1))`, arredondado a 4 casas. O CSS então mistura `--sec-mid` (`#1c1c1c`) com `--sec-edge`
(`#000`) na proporção `t`. Exemplo com 5 seções: `t = 0; 0,7071; 1; 0,7071; 0` -> `#000000`, `#141414` (28 x 0,7071 = 19,8),
`#1c1c1c`, `#141414`, `#000000`. A primeira e a última ficam pretas, a do meio é a mais clara. Tudo isso só funciona com
`color-mix` (navegadores recentes; o WebView2 do Windows atende).

Opção "degradê dentro de cada painel": `html.bg-cada .sec { background: var(--sec-grad) }` (`app.css:69`), ligada por
`applySectionBg` (`app.js:39`) a partir da preferência `section_bg` (`"entre"` padrão, `"cada"`).

### 5.7 A folha (sheet)

`hub/web/js/sheet.js` + `app.css:331-345`. Uma página aberta **por cima de tudo**: `position: fixed; inset: 0; z-index: 40`,
fundo opaco `var(--bg-base)` (para o hub de baixo não aparecer atrás de gráficos), rolagem própria
(`.sheet-body { overflow-y: auto; overscroll-behavior: contain }`). Regras de projeto:

- Ao abrir: `lock(true)` e `inert` em `hero`, `page`, `topbar`, `menu` (sem foco por teclado nem leitor de tela no fundo).
- A rolagem da página de baixo para: `html.sheet-open, html.sheet-open body { overflow: hidden }` (nativa) e
  `lenis.stop()` (suave) em `motion.js:130`.
- A folha leva `data-lenis-prevent` para o Lenis não capturar a roda do mouse dentro dela.
- Esc fecha, a menos que haja um `dialog[open]` por cima (o diálogo fecha primeiro).
- Quem decide abrir/fechar é o roteador (hash `#/buttons`): a folha só **pede** para fechar (`onRequestClose`).
- Com a barra de título do pywebview, `body.has-titlebar .sheet { top: 36px }` deixa os botões da janela à mostra.

### 5.8 Responsividade (≤ 760 px)

`app.css:192-203`:

```css
@media (max-width: 760px) {
  .topbar { padding: 0 6px 0 14px; gap: 10px; }
  .brand { display: none; }
  .hero, body.has-titlebar .hero { position: relative; top: 0; height: auto; display: block; padding: 0 16px 20px; }
  .hero-stage { position: relative; height: 56vh; }
  .hero-tools { left: 16px; right: 16px; top: 8px; max-width: none; }
  .hero-left { margin-top: 10px; }
  .hero-right { justify-items: start; text-align: left; grid-template-columns: repeat(3, minmax(0, 1fr)); margin-top: 14px; }
  .sec { padding: 36px 16px 52px; min-height: 0; }
  .slider-row { grid-template-columns: 1fr 64px; }
  .slider-row input { grid-column: 1 / -1; grid-row: 2; }
}
```

E, no JS, a mesma fronteira de **761 px** em quatro lugares: `stack()` (empilhar), `gsap.matchMedia("(min-width: 761px)")`
(parallax do hero), `pointer: fine` (cursor em ponto e OrbitControls) e `narrow = w <= 760` no layout de fundo do
`webgl.js:171`. No celular: hero não fixo (`position: relative`), sem parallax, sem seções empilhadas, sem
OrbitControls (para a rolagem por toque funcionar) e o canvas ocupa `56vh`.

## 6. Receita para reaproveitar

1. Copie o bloco `:root` de tokens e troque só os valores; nunca escreva cor literal fora dele (inclusive em JS e canvas 2D).
2. Mantenha `html` sólido, `body` translúcido, canvas transparente (`alpha: true`, `setClearColor(0x000000, 0)`).
3. Hero `sticky` com `z-index: 0` e seções `z-index: 1`. Leituras sobre o canvas com `pointer-events: none`.
4. Seções `sticky` só acima do breakpoint, `top` calculado por `stickTop` e recalculado por `ResizeObserver`.
5. Toda animação ligada à rolagem usa `scrub: true` e `ease: "none"`; nada anima por tempo (ver
   `interactive-physics-parallax.md`).
6. Texto dinâmico sempre como texto (`textContent`/`createTextNode`); o projeto tem um teste que proíbe `innerHTML` nos
   módulos de UI e 3D (`tests/test_web_static.py:63-67,81-86`). A única exceção está em `pages/rim.js:207`, um SVG fixo
   do próprio código.
