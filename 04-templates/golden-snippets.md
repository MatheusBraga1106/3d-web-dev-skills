# Golden snippets: blocos reais do projeto, prontos para copiar

Todos os blocos abaixo foram **copiados do projeto de referência** (Hub Sim Racing, commit `ff34fab`, versão 0.2.1) por um
script que extrai as linhas indicadas, então o código é exatamente o que roda lá, com os comentários originais em português.
Cada bloco traz o caminho e as linhas de origem e uma lista "o que observar". Exceções, avisadas no próprio título: o
servidor FastAPI mínimo (seção 5) é **escrito neste repositório** no mesmo padrão do projeto, e foi testado; e o widget de canvas
(seção 6) tem o desenho do mostrador omitido, marcado com `// ...`.

Mapa: 1 cena e câmera, 2 laço e pose, 3 seções empilhadas e movimento, 4 HTML/CSS (tokens e overlay), 5 servidor FastAPI +
WebSocket, 6 widget de canvas com `devicePixelRatio`, 7 carregamento de modelo contra corrida, 8 carregar o 3D sob demanda.

Para a teoria de cada um, veja `02-design-and-3d-scenes/` e `03-utilities-and-pipeline/`.

---

## 1. Setup da cena em `createViewer`

### 1.1 Constantes (cena, luz, brilho)

Fonte: `hub/web/js/webgl.js:18-30`

```js
const MAX_FPS = 30, MAX_DPR = 1.5, IDLE_DEG_PER_S = 10;
const FACE_Y = 0.14;           // face dos modelos em código: onde fica um botão solto "no ar" entre as peças
const PICK_LIMIT = 10;         // o mesmo limite das posições 3D no backend (hub/plugins/rim/layout.py)
const LAYOUTS = {
  background: { tilt: 62, yaw: -24, fov: 32, dist: 10 },   // fundo: à direita e afastado
  center: { tilt: 70, yaw: 0, fov: 30, dist: 8 },          // card: centralizado
};
const SOLID = { color: 0x161616, metalness: 0.5, roughness: 0.45, transparent: true, opacity: 0.6 };   // = models3d.js
const EDGE = { color: 0xdcdcd8, transparent: true, opacity: 0.85 };
const GLOW = 0x7fb487;
const EDGE_LAYER = 1;          // arestas: a câmera desenha, os raios (rótulos, pick) não testam         // peça de um botão apertado (o verde de "conectado")
const deg = THREE.MathUtils.degToRad;
const reduceMotion = window.matchMedia("(prefers-reduced-motion: reduce)").matches;
```

O que observar:

- `LAYOUTS` guarda **câmera por intenção** (`center` / `background`): `tilt` inclina o conjunto, `yaw` o gira, `fov` e `dist` enquadram.
- `SOLID`/`EDGE`/`GLOW` são os únicos "materiais" do projeto; trocar a aparência é trocar estes três objetos.
- `EDGE_LAYER` existe para o raycast ignorar as arestas. O comentário da linha 28 está misturado com o de `GLOW` no original.

### 1.2 Renderer, cena, câmera, luzes e grafo

Fonte: `hub/web/js/webgl.js:105-137`

```js
export function createViewer(container, { interactive = false, layout = "background", orbit = interactive, zoom = true,
  style: initialStyle = "wireframe" } = {}) {
  const cfg = LAYOUTS[layout] || LAYOUTS.background;
  const canvas = document.createElement("canvas");
  canvas.setAttribute("aria-hidden", "true");
  let renderer;
  try {
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true });
  } catch (e) {
    console.warn("3D desligado: WebGL indisponível.", e);
    return null;
  }
  renderer.setClearColor(0x000000, 0);
  renderer.outputColorSpace = THREE.SRGBColorSpace;
  const scene = new THREE.Scene();
  scene.background = null;                       // transparente: a interface translúcida fica por cima
  const camera = new THREE.PerspectiveCamera(cfg.fov, 1, 0.1, 100);
  camera.position.set(0, 0, cfg.dist);
  camera.layers.enable(EDGE_LAYER);
  scene.add(new THREE.AmbientLight(0xffffff, 0.35));
  const key = new THREE.DirectionalLight(0xffffff, 1.9);
  key.position.set(3, 4, 6);
  scene.add(key);
  const rim = new THREE.DirectionalLight(0xc8d0d8, 0.7);
  rim.position.set(-5, 2, -4);
  scene.add(rim);
  const root = new THREE.Group(), stage = new THREE.Group();
  stage.rotation.x = deg(cfg.tilt);             // os modelos são montados "deitados": a inclinação os mostra de frente
  root.add(stage);
  scene.add(root);
  container.append(canvas);
  const labelLayer = interactive ? document.createElement("div") : null;
  if (labelLayer) { labelLayer.className = "viewer-labels"; container.append(labelLayer); }
```

O que observar:

- Retorna `null` sem WebGL: **quem chama decide o plano B** (só números, desenho 2D).
- Fundo transparente (`alpha: true`, `setClearColor(0x000000, 0)`, `scene.background = null`): a página aparece atrás do canvas.
- Três luzes: ambiente fraca (0.35), principal forte (1.9) e contraluz fria (0.7). Nenhuma projeta sombra.
- `root > stage > modelo`: `stage` leva a inclinação, `root` leva o yaw e a escala responsiva, o modelo leva o giro do volante.

### 1.3 OrbitControls sem inércia e com zoom opcional

Fonte: `hub/web/js/webgl.js:148-156`

```js
  const controls = interactive && orbit ? new OrbitControls(camera, canvas) : null;   // sem orbit (toque): a rolagem da página passa
  if (controls) {
    controls.enablePan = false;
    controls.enableZoom = zoom;            // sem zoom: a roda do mouse rola a página
    controls.enableDamping = false;              // sem inércia: nada fica desenhando sozinho depois de soltar
    controls.minDistance = cfg.dist * 0.5;
    controls.maxDistance = cfg.dist * 1.6;
    controls.addEventListener("change", request);
  }
```

O que observar: sem `orbit` (toque), nenhum controle é criado e a rolagem da página passa; `enableDamping = false` mantém o laço
ocioso ao soltar o mouse; o evento `change` é o único gatilho de redesenho do orbit.

### 1.4 Resize com `ResizeObserver` e teto de `devicePixelRatio`

Fonte: `hub/web/js/webgl.js:158-183`

```js
  function request() {
    if (!dead && !raf) raf = requestAnimationFrame(tick);
  }

  function resize() {
    if (dead) return;
    const w = Math.max(1, container.clientWidth), h = Math.max(1, container.clientHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, MAX_DPR));
    renderer.setSize(w, h, false);
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    if (layout === "background") {
      const viewH = 2 * Math.tan(deg(camera.fov / 2)) * camera.position.z;
      const narrow = w <= 760;   // celular: centralizado e menor, atrás dos cards translúcidos
      root.position.set(narrow ? 0 : viewH * camera.aspect * 0.24, narrow ? -viewH * 0.12 : 0, 0);
      root.scale.setScalar(narrow ? 0.7 : 1);
    } else {
      root.scale.setScalar(centerScale(camera.aspect));
    }
    request();
  }
  const ro = new ResizeObserver(resize);
  ro.observe(container);
  const onVisible = () => { if (!document.hidden) { lastFrame = 0; request(); } };
  document.addEventListener("visibilitychange", onVisible);
  resize();
```

O que observar: `setSize(w, h, false)` não mexe no CSS; `Math.min(devicePixelRatio, MAX_DPR)` é reaplicado a cada resize; a tela
estreita e o layout `center` escalam o conjunto para o aro nunca passar das bordas.

---

## 2. O laço (desenha só quando sujo) e a pose

### 2.1 `request` e `tick`

Fonte: `hub/web/js/webgl.js:260-281` (o `request()`, linhas 158-160, está no bloco 1.4)

```js
  function tick(t) {
    raf = 0;
    if (dead || document.hidden) return;         // escondida: nada é desenhado; volta no visibilitychange
    if (lastFrame && t - lastFrame < 1000 / MAX_FPS - 2) { raf = requestAnimationFrame(tick); return; }   // até 30/s
    const spinning = idle && current && !reduceMotion;
    if (spinning && lastFrame) pose.yaw += IDLE_DEG_PER_S * Math.min(0.1, (t - lastFrame) / 1000);
    lastFrame = t;
    root.rotation.y = deg(cfg.yaw + pose.yaw);
    if (current) {
      current.rotation.set(deg(pose.x), deg(pose.y), 0);
      const knob = current.getObjectByName("knob");
      if (knob) knob.position.set(pose.knob.x * KNOB_X, knob.position.y, -pose.knob.y * KNOB_Z);
      for (const k of PEDAL_KEYS) {
        const pedal = current.getObjectByName(`pedal-${k}`);
        if (pedal) pedal.rotation.x = deg(pedalAngle(pose.pedals[k]));
      }
    }
    renderer.render(scene, camera);
    placeLabels();
    frames++;
    if (spinning) raf = requestAnimationFrame(tick);
  }
```

O que observar:

- `tick` **não se reagenda** (exceto o giro idle). O navegador fica ocioso até o próximo `request()`.
- Três guardas no início: janela escondida, teto de 30/s (`- 2` ms de folga) e `dt` limitado a 0,1 s.
- `placeLabels()` roda depois do `render`: rótulos herdam o teto e a pausa.

### 2.2 Aplicar um valor medido com limiar (epsilon)

Fonte: `hub/web/js/poses.js:56-58` e `hub/web/js/webgl.js:341-347`

```js
export function moved(prev, next, eps) {
  return Math.abs(next - prev) >= eps;
}
```

```js
  function rotateModel(x, y) {
    if (dead || !Number.isFinite(x) || !Number.isFinite(y)) return;
    if (!moved(pose.x, x, WHEEL_EPS_DEG) && !moved(pose.y, y, WHEEL_EPS_DEG)) return;
    pose.x = x;
    pose.y = y;
    request();
  }
```

Fonte: `hub/web/js/hero.js:46-53` (de onde o dado ao vivo chega ao modelo)

```js
  function update(live) {
    const r = heroReadout(live);
    status.textContent = r.status;
    for (const [k] of FIELDS) values[k].textContent = r[k];
    last = r.angleDeg;
    // eixo Y do modelo aponta para o piloto: virar para a direita (horário) é rotação negativa
    if (viewer && follow && last != null) viewer.rotateModel(0, -last);
  }
```

O que observar: `moved(prev, next, eps)` compara com o último valor **aplicado**; o ruído abaixo de 0,2 grau não gera quadro e a
deriva lenta nunca fica presa; o sinal negativo de `-last` vem da convenção do eixo Y do modelo.

### 2.3 Pose de pedal e visibilidade da aba

Fonte: `hub/web/js/poses.js:87-89` e `hub/web/js/webgl.js:181-183`

```js
export function pedalAngle(v) {
  return PEDAL_REST_DEG - (finite(v) ? clamp(v, 0, 1) : 0) * PEDAL_TRAVEL_DEG;
}
```

```js
  const onVisible = () => { if (!document.hidden) { lastFrame = 0; request(); } };
  document.addEventListener("visibilitychange", onVisible);
  resize();
```

---

## 3. Seções empilhadas e movimento

### 3.1 CSS: hero fixo, seções que sobem por cima e `--cover`

Fonte: `hub/web/css/app.css:50-54` (hero), `67-74` (seções)

```css
.hero { position: sticky; top: var(--topbar-h); height: calc(100vh - var(--topbar-h)); z-index: 0; overflow: hidden;
  display: grid; grid-template-columns: minmax(170px, 1fr) minmax(0, 2.6fr) minmax(170px, 1fr); align-items: end; padding: 0 28px 34px; }
body.has-titlebar .hero { top: calc(var(--topbar-h) + 36px); height: calc(100vh - var(--topbar-h) - 36px); }
.hero-stage { position: absolute; inset: 0; }
.hero-stage canvas { display: block; width: 100%; height: 100%; }
```

```css
.sections { position: relative; z-index: 1; }
.sec { background: color-mix(in srgb, var(--sec-mid) calc(var(--t, 0) * 100%), var(--sec-edge)); padding: 64px 28px 80px; min-height: 70vh; border-top: 1px solid var(--line); scroll-margin-top: var(--topbar-h); }
html.bg-cada .sec { background: var(--sec-grad); }          /* opção "Em cada painel": o degradê dentro de cada seção */
/* empilhadas (motion.js, só no desktop): cada seção gruda (o top vem do JS) e a seguinte sobe por cima;
   a coberta escurece com --cover (0 a 1, ligado à rolagem) */
.sections.stacked > .sec { position: sticky; }
.sections.stacked > .sec::after { content: ""; position: absolute; inset: 0; background: #000; pointer-events: none;
  opacity: calc(var(--cover, 0) * .6); }
```

### 3.2 JS: `stack` (sticky com `top` calculado) e `stickTop`

Fonte: `hub/web/js/motion.js:19-37` e `hub/web/js/poses.js:107-111`

```js
// seções empilhadas (desktop): cada uma gruda e a seguinte sobe por cima, como a primeira sobre o volante
function stack(sections) {
  const wide = window.matchMedia("(min-width: 761px)");
  const place = () => {
    for (const s of sections.querySelectorAll(":scope > .sec")) s.style.top = `${stickTop(s.offsetHeight, window.innerHeight, topbar())}px`;
  };
  const ro = new ResizeObserver(place);
  const watch = () => {
    ro.disconnect();
    for (const s of sections.querySelectorAll(":scope > .sec")) ro.observe(s);
    place();
  };
  const toggle = () => sections.classList.toggle("stacked", wide.matches);
  wide.addEventListener("change", toggle);
  window.addEventListener("resize", place);
  toggle();
  watch();
  return watch;
}
```

```js
// seções empilhadas: onde cada uma gruda (top do sticky). Curta: logo abaixo da barra. Mais alta que a tela: só
// quando o rodapé chega ao fim da tela (top negativo), para nada ficar escondido sob a seção seguinte
export function stickTop(height, viewport, bar) {
  return finite(height) && finite(viewport) ? Math.min(bar, viewport - height) : bar;
}
```

### 3.3 JS: Lenis + GSAP ScrollTrigger (parallax do hero, entrada das seções e `--cover`)

Fonte: `hub/web/js/motion.js:93-107` e `109-124`

```js
  const lenis = new Lenis({ autoRaf: false, lerp: 0.12 });
  lenis.on("scroll", ScrollTrigger.update);
  gsap.ticker.add((time) => lenis.raf(time * 1000));
  gsap.ticker.lagSmoothing(0);
  if (window.matchMedia("(pointer: fine)").matches) startCursor(gsap);

  const stage = hero.querySelector(".hero-stage");
  const edges = hero.querySelectorAll(".hero-left, .hero-right");
  gsap.matchMedia().add("(min-width: 761px)", () => {
    // o volante recua, diminui e esmaece enquanto a primeira seção sobe por cima dele
    gsap.to(stage, { yPercent: 14, scale: 0.86, opacity: 0.4, ease: "none",
      scrollTrigger: { trigger: sections, start: "top bottom", end: "top top", scrub: true } });
    gsap.to(edges, { yPercent: -30, opacity: 0, ease: "none",
      scrollTrigger: { trigger: sections, start: "top 85%", end: "top 35%", scrub: true } });
  });
```

```js
  // cada seção sobe e aparece ligada à rolagem e escurece quando a seguinte a cobre (só se vê com .stacked);
  // seções remontadas ganham gatilhos novos e os velhos morrem
  const restack = stack(sections);
  const reveal = () => {
    for (const t of ScrollTrigger.getAll()) if (!document.contains(t.trigger)) t.kill(true);
    for (const s of sections.querySelectorAll(".sec")) {
      if (s.dataset.motion) continue;
      s.dataset.motion = "1";
      gsap.fromTo(s.querySelector(".page-inner"), { y: 48, opacity: 0.25 }, { y: 0, opacity: 1, ease: "none",
        scrollTrigger: { trigger: s, start: "top bottom", end: "top 60%", scrub: true } });
      const next = s.nextElementSibling;
      if (next) gsap.fromTo(s, { "--cover": 0 }, { "--cover": 1, ease: "none",
        scrollTrigger: { trigger: next, start: "top bottom", end: () => `top ${topbar()}px`, scrub: true } });
    }
  };
  reveal();
```

O que observar:

- Tudo com `ease: "none"` e `scrub: true`: o progresso **é** a posição da rolagem. Nada anima por tempo.
- O `gsap.ticker` aciona o Lenis (`autoRaf: false`) e `lagSmoothing(0)` evita pulos.
- `matchMedia("(min-width: 761px)")` liga o parallax só no desktop e o GSAP desfaz sozinho ao sair da faixa.
- `reveal()` primeiro mata gatilhos órfãos, depois só cria os das seções sem `dataset.motion`: pode ser chamado a cada remontagem.
- O `end` de `--cover` é uma função (reavalia a altura da barra em cada refresh).

### 3.4 Fundo entre os painéis (`sectionTone` + `color-mix`)

Fonte: `hub/web/js/poses.js:193-199`, `hub/web/js/app.js:302-306` e `hub/web/css/app.css:68`

```js
// fundo entre os painéis: o quão claro é o painel i de n (0 = preto, 1 = o do meio, o mais claro); sin(π·i/(n−1)).
// Com menos de duas seções, índice fora da lista ou lixo, fica preto (0). Arredondado para não sair -0 nem 1e-17.
export function sectionTone(i, n) {
  if (!Number.isInteger(i) || !Number.isInteger(n) || n <= 1 || i < 0 || i >= n) return 0;
  const t = Math.round(clamp(Math.sin(Math.PI * i / (n - 1)), 0, 1) * 1e4) / 1e4;
  return t === 0 ? 0 : t;                      // Math.round(-0) seria -0
}
```

```js
  const plan = sectionPlan();
  const built = await Promise.all(plan.map(async (p, i) => {
    const el = ui.h("section", { class: "sec", id: sectionIdFor(`#/${p.key}`) });
    el.style.setProperty("--t", String(sectionTone(i, plan.length)));      // o quão claro é o fundo deste painel
    const inner = ui.h("div", { class: "page-inner" });
```

```css
.sec { background: color-mix(in srgb, var(--sec-mid) calc(var(--t, 0) * 100%), var(--sec-edge)); padding: 64px 28px 80px; min-height: 70vh; border-top: 1px solid var(--line); scroll-margin-top: var(--topbar-h); }
```

---

## 4. Integração HTML/CSS: tokens e overlay

### 4.1 Tokens (todo o `:root`)

Fonte: `hub/web/css/tokens.css:4-30`

```css
:root {
  --bg-base: #070707;                         /* sólido, no html: atrás de tudo (futuro canvas 3D) */
  --bg: rgba(10, 10, 10, .70);                /* translúcido, no body */
  --panel: rgba(255, 255, 255, .025);         /* cards e quadros: quase transparentes */
  --panel-solid: rgba(12, 12, 12, .94);       /* diálogos: legíveis por cima do 3D */
  --side: rgba(10, 10, 10, .55);
  --ink: #E8E8E5; --ink-2: #A9A9A5; --ink-3: #6F6F6B;
  --line: rgba(255, 255, 255, .10); --line-2: rgba(255, 255, 255, .05);
  --corner: rgba(255, 255, 255, .55);         /* marcas em L nos cantos (mira de HUD) */
  --accent: #EDEDEA; --accent-ink: #0A0A0A;
  --ok: #7FB487; --ok-bg: rgba(127, 180, 135, .12); --warn: #E0B45E; --warn-bg: rgba(224, 180, 94, .12);
  --err: #F08A80; --err-bg: rgba(240, 138, 128, .12);
  --bt-hit-bg: rgba(237, 237, 234, .14);     /* Botões: linha da tabela do botão apertado agora */
  --rpm-off: rgba(255, 255, 255, .07); --rpm-on: #E8E8E5;           /* telemetria: segmentos da barra de RPM (apagado, aceso) */
  --cal-grid: rgba(255, 255, 255, .08); --cal-band: rgba(237, 237, 234, .10); --cal-handle: #EDEDEA;   /* calibração dos pedais: grade, faixa gravada, alças e linhas */
  --pedal-gas: #7FB487; --pedal-brake: #F08A80; --pedal-clutch: #7FA8D0;   /* telemetria: barras dos pedais do jogo */
  --radius: 0; --radius-sm: 2px; --shadow: none;
  --blur: blur(6px);
  --topbar-h: 44px;
  --sec-grad: linear-gradient(180deg, #000 0%, #3a3a3a 50%, #000 100%);   /* fundo de toda seção: preto, cinza médio, preto */
  --sec-edge: #000;                                  /* fundo da primeira e da última seção */
  --sec-mid: #1c1c1c;                                /* o cinza do painel do meio, bem perto do preto */
  --font: "Segoe UI Variable Text", "Segoe UI", system-ui, sans-serif;
  --mono: "Cascadia Mono", Consolas, ui-monospace, monospace;
  --ease: cubic-bezier(.2, .7, .2, 1);
  color-scheme: dark;
}
```

### 4.2 A casca HTML (hero, seções, cursor, importmap)

Fonte: `hub/web/index.html:12-39`

```html
<body class="readonly">
<header class="titlebar pywebview-drag-region" id="titlebar" hidden>
  <span class="titlebar-name">Hub Sim Racing</span>
  <div class="titlebar-controls">
    <button type="button" data-win="minimize" aria-label="Minimizar"><svg viewBox="0 0 12 12"><path d="M2 6h8"/></svg></button>
    <button type="button" data-win="toggle_maximize" aria-label="Maximizar"><svg viewBox="0 0 12 12"><rect x="2.5" y="2.5" width="7" height="7"/></svg></button>
    <button type="button" data-win="hide" aria-label="Fechar para a bandeja" class="close"><svg viewBox="0 0 12 12"><path d="M3 3l6 6M9 3l-6 6"/></svg></button>
  </div>
</header>
<header class="topbar" id="topbar">
  <a class="brand" href="#/top">Hub Sim Racing</a>
  <nav class="status" id="status" aria-label="Dispositivos"></nav>
  <a class="update-notice admin-only" id="update-notice" href="#/settings" hidden></a>
  <button type="button" class="burger" id="burger" aria-label="Menu" aria-expanded="false" aria-controls="menu"><i></i><i></i></button>
</header>
<nav class="menu" id="menu" aria-label="Seções" data-lenis-prevent hidden></nav>
<section class="hero" id="hero" aria-label="Volante">
  <div class="hero-stage"></div>
  <div class="hero-left"></div>
  <div class="hero-right"></div>
  <div class="hero-tools"></div>
</section>
<main class="sections" id="page" tabindex="-1"></main>
<div class="cursor" id="cursor" aria-hidden="true"></div>
<div class="toasts" id="toasts" role="status" aria-live="polite"></div>
<script type="importmap">{"imports": {"three": "/vendor/three/three.module.js", "three/addons/": "/vendor/three/addons/", "lenis": "/vendor/lenis/lenis.mjs"}}</script>
<script type="module" src="/js/app.js"></script>
</body>
```

### 4.3 Camadas: fundo, barra, hero, `pointer-events`

Fonte: `hub/web/css/app.css:3`, `12`, `32-33` e `50-65`

```css
html { background: var(--bg-base); }      /* camada do futuro canvas 3D; o body translúcido fica por cima */
```

```css
body { margin: 0; background: var(--bg); color: var(--ink); font: 14px/1.45 var(--font); -webkit-font-smoothing: antialiased; }
```

```css
.topbar { position: fixed; top: 0; left: 0; right: 0; height: var(--topbar-h); z-index: 30; display: flex; align-items: center; gap: 16px;
  padding: 0 10px 0 20px; background: rgba(7, 7, 7, .78); backdrop-filter: var(--blur); border-bottom: 1px solid var(--line); }
```

```css
.hero { position: sticky; top: var(--topbar-h); height: calc(100vh - var(--topbar-h)); z-index: 0; overflow: hidden;
  display: grid; grid-template-columns: minmax(170px, 1fr) minmax(0, 2.6fr) minmax(170px, 1fr); align-items: end; padding: 0 28px 34px; }
body.has-titlebar .hero { top: calc(var(--topbar-h) + 36px); height: calc(100vh - var(--topbar-h) - 36px); }
.hero-stage { position: absolute; inset: 0; }
.hero-stage canvas { display: block; width: 100%; height: 100%; }
.hero-left, .hero-right { position: relative; z-index: 1; pointer-events: none; }
.hero-left { grid-column: 1; }
.hero-right { grid-column: 3; display: grid; gap: 10px; justify-items: end; text-align: right; }
.hero-kicker { font: 11px var(--mono); letter-spacing: .1em; text-transform: uppercase; color: var(--ink-3); }
.hero h1 { margin: 6px 0 4px; font: 300 clamp(40px, 6vw, 76px)/1 var(--font); letter-spacing: -.03em; }
.hero-sub { font: 12px var(--mono); color: var(--ink-2); }
.hero-field small { display: block; font: 10px var(--mono); letter-spacing: .1em; text-transform: uppercase; color: var(--ink-3); }
.hero-field b { font: 400 26px/1.1 var(--mono); font-variant-numeric: tabular-nums; }
.hero-tools { position: absolute; left: 28px; top: 16px; z-index: 2; max-width: min(360px, 32vw); }   /* fora do aro */
.hero-tools > .row { background: var(--panel-solid); border: 1px solid var(--line); padding: 8px 12px; }
.hero.no3d .hero-stage { display: none; }
```

### 4.4 Marcas em L (mira de HUD) sem pseudo-elementos

Fonte: `hub/web/css/app.css:108-121`

```css
/* blocos: wireframe técnico (bordas de 1px, sem fundo sólido, marcas em L nos cantos) */
/* marcas em L: 8 traços de 10 px desenhados no fundo do próprio elemento (sem pseudo-elementos) */
.card, .tile, dialog.modal {
  background-image:
    linear-gradient(var(--corner), var(--corner)), linear-gradient(var(--corner), var(--corner)),
    linear-gradient(var(--corner), var(--corner)), linear-gradient(var(--corner), var(--corner)),
    linear-gradient(var(--corner), var(--corner)), linear-gradient(var(--corner), var(--corner)),
    linear-gradient(var(--corner), var(--corner)), linear-gradient(var(--corner), var(--corner));
  background-size: 10px 1px, 1px 10px, 10px 1px, 1px 10px, 10px 1px, 1px 10px, 10px 1px, 1px 10px;
  background-position: left top, left top, right top, right top, left bottom, left bottom, right bottom, right bottom;
  background-repeat: no-repeat;
  background-origin: border-box;
}
.card { background-color: var(--panel); backdrop-filter: var(--blur); border: 1px solid var(--line); border-radius: var(--radius); padding: 14px 16px; box-shadow: var(--shadow); }
```

### 4.5 Responsivo (≤ 760 px)

Fonte: `hub/web/css/app.css:192-206`

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
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation: none !important; transition: none !important; }
}
```

O que observar nesta seção toda: o canvas é `absolute inset: 0` dentro de um hero `sticky`; o texto sobre ele tem `pointer-events: none`;
as seções têm `z-index: 1` contra `0` do hero; a barra fica a 30 e a folha a 40; toda cor sai dos tokens.

---

## 5. Servidor FastAPI mínimo com WebSocket ao vivo (escrito aqui, mesmo padrão)

> **Este bloco não é cópia literal.** É um servidor enxuto escrito neste repositório, seguindo o padrão de
> `hub/server/app.py`, `live.py`, `auth.py` e `runner.py`, e **testado** (HTTP, guard de Host/Origin/admin e o canal
> `full` + `diff`, tanto no `TestClient` quanto num uvicorn real). Os trechos centrais (`diff`, `serve_until_disconnect`,
> `live_loop`, `guard`, `require_admin`, `ws_open`, `ServerRunner.start`) são os mesmos do projeto, condensados.

Arquivo: `app/server.py`

```py
"""Servidor mínimo no padrão do Hub Sim Racing: fábrica create_app, guard de Host/Origin, admin por token,
canal ao vivo por WebSocket (mensagem "full" e depois só "diff"), estáticos por último."""
from __future__ import annotations

import secrets
import threading
from pathlib import Path

import anyio
import uvicorn
from fastapi import Depends, FastAPI, HTTPException, Request, WebSocket, WebSocketDisconnect
from fastapi.responses import JSONResponse
from fastapi.staticfiles import StaticFiles

LOCAL_HOSTS = ("127.0.0.1", "localhost", "[::1]")


class Core:
    """O 'núcleo' da aplicação: aqui só um dicionário de valores; no projeto real, o registro de plug-ins."""

    def __init__(self, admin_token: str | None = None):
        self.admin_token = admin_token or secrets.token_urlsafe(32)   # novo a cada execução
        self.values: dict[str, float] = {"angle_deg": 0.0}
        self._lock = threading.Lock()

    def snapshot(self) -> dict:              # tem de ser BARATO: roda 30 vezes por segundo por cliente
        with self._lock:
            return {k: {"live": {"v": v}} for k, v in self.values.items()}

    def set_value(self, key: str, v: float) -> None:
        with self._lock:
            self.values[key] = v


def diff(prev: dict, cur: dict) -> dict:
    changed = {k: v for k, v in cur.items() if prev.get(k) != v}
    changed.update({k: None for k in prev if k not in cur})          # None = saiu
    return changed


async def serve_until_disconnect(ws: WebSocket, sender) -> None:
    """Roda `sender()` e escuta a desconexão em paralelo (sem isso, um canal parado nunca percebe que o cliente saiu)."""
    async with anyio.create_task_group() as tg:
        async def send() -> None:
            try:
                await sender()
            except (WebSocketDisconnect, RuntimeError):
                pass
            finally:
                tg.cancel_scope.cancel()

        tg.start_soon(send)
        try:
            while (await ws.receive())["type"] != "websocket.disconnect":
                pass
        except (WebSocketDisconnect, RuntimeError):
            pass
        finally:
            tg.cancel_scope.cancel()


async def live_loop(ws: WebSocket, snapshot, interval: float = 1 / 30) -> None:
    async def loop() -> None:
        prev = snapshot()
        await ws.send_json({"t": "full", "devices": prev})
        while True:
            await anyio.sleep(interval)
            cur = snapshot()
            changed = diff(prev, cur)
            if changed:
                await ws.send_json({"t": "diff", "devices": changed})
            prev = cur

    await serve_until_disconnect(ws, loop)


def create_app(core: Core, web_dir: Path | None = None) -> FastAPI:
    app = FastAPI(title="Meu App", docs_url=None, redoc_url=None, openapi_url=None)

    def allowed_hosts(conn) -> set[str]:
        server = conn.scope.get("server")                       # (host, porta) em que o uvicorn escuta
        port = server[1] if server and server[1] else None
        return {f"{h}:{port}" for h in LOCAL_HOSTS} if port else set()

    def host_ok(conn) -> bool:                                  # barra DNS rebinding
        return (conn.headers.get("host") or "").strip().lower() in allowed_hosts(conn)

    def origin_ok(conn) -> bool:                                # barra CSRF de outro site
        origin = conn.headers.get("origin")
        return origin is None or origin.strip().lower() == "http://" + conn.headers.get("host", "").strip().lower()

    @app.middleware("http")
    async def guard(request: Request, call_next):
        if not host_ok(request):
            return JSONResponse({"error": "Endereço não permitido."}, status_code=403)
        if request.method not in ("GET", "HEAD") and not origin_ok(request):
            return JSONResponse({"error": "Pedido de outro site bloqueado."}, status_code=403)
        return await call_next(request)

    def require_admin(request: Request):
        given = (request.headers.get("X-Hub-Admin") or "").encode("utf-8", "surrogateescape")
        local = request.client is not None and request.client.host in ("127.0.0.1", "::1", "localhost")
        # compare_digest com bytes: com str não-ASCII levantaria TypeError (viraria 500)
        if not (local and secrets.compare_digest(given, core.admin_token.encode("utf-8"))):
            raise HTTPException(403, "Somente a janela do app no PC pode alterar.")

    @app.get("/api/state")
    def state():
        return core.snapshot()

    @app.post("/api/value/{key}", dependencies=[Depends(require_admin)])
    def set_value(key: str, body: dict):
        v = body.get("value")
        if isinstance(v, bool) or not isinstance(v, (int, float)):       # tipo estrito: bool é int em Python
            raise HTTPException(400, "Envie {\"value\": número}.")
        core.set_value(key, float(v))
        return {"ok": True, "error": None}

    @app.websocket("/ws/live")
    async def ws_live(ws: WebSocket):
        if not host_ok(ws) or not origin_ok(ws):
            await ws.close(code=4403)            # antes do accept: o handshake é recusado (HTTP 403)
            return
        await ws.accept()
        try:
            await live_loop(ws, core.snapshot)
        except (WebSocketDisconnect, RuntimeError):
            pass

    if web_dir is not None:                      # SEMPRE por último, ou engole as rotas da API
        app.mount("/", StaticFiles(directory=str(web_dir), html=True), name="web")
    return app


def run(core: Core, port: int = 8770, host: str = "127.0.0.1", web_dir: Path | None = None) -> uvicorn.Server:
    """uvicorn numa thread própria (o laço principal fica livre para a janela). Para parar: server.should_exit = True."""
    cfg = uvicorn.Config(create_app(core, web_dir), host=host, port=port, log_level="warning", lifespan="off",
                         ws="websockets-sansio", proxy_headers=False, timeout_graceful_shutdown=2,
                         log_config=None)           # log_config=None: sem console o dictConfig padrão quebra
    server = uvicorn.Server(cfg)
    threading.Thread(target=server.run, name="http", daemon=True).start()
    return server
```

Teste mínimo (pytest). Atenção: o WebSocket do `TestClient` precisa da URL **absoluta** para o cabeçalho `Host` bater com o guard.

```py
import warnings
with warnings.catch_warnings():
    warnings.filterwarnings("ignore")
    from fastapi.testclient import TestClient
from app.server import Core, create_app

BASE = "http://127.0.0.1:8770"
core = Core(admin_token="tok")
c = TestClient(create_app(core), base_url=BASE, client=("127.0.0.1", 50000))
assert c.get("/api/state").json() == {"angle_deg": {"live": {"v": 0.0}}}
assert c.post("/api/value/angle_deg", json={"value": 5}).status_code == 403          # sem token
assert c.post("/api/value/angle_deg", json={"value": 5}, headers={"X-Hub-Admin": "tok"}).status_code == 200
assert c.post("/api/value/angle_deg", json={"value": True}, headers={"X-Hub-Admin": "tok"}).status_code == 400
assert c.post("/api/value/angle_deg", json={"value": 1}, headers={"X-Hub-Admin": "tok", "Origin": "http://evil.example"}).status_code == 403
assert TestClient(create_app(core), base_url="http://evil.example:8770").get("/api/state").status_code == 403
remote = TestClient(create_app(core), base_url=BASE, client=("192.168.0.9", 50000))
assert remote.post("/api/value/angle_deg", json={"value": 1}, headers={"X-Hub-Admin": "tok"}).status_code == 403   # não local
with c.websocket_connect("ws://127.0.0.1:8770/ws/live") as ws:
    first = ws.receive_json()
    assert first == {"t": "full", "devices": {"angle_deg": {"live": {"v": 5.0}}}}, first
    core.set_value("angle_deg", 12.5)
    nxt = ws.receive_json()
    assert nxt == {"t": "diff", "devices": {"angle_deg": {"live": {"v": 12.5}}}}, nxt
print("ok")
```

Cliente JS do canal (copiado de `hub/web/js/api.js:52-66`):

```js
export function connectLive(onMessage, onStatus = () => {}) {
  let delay = 1000;
  const open = () => {
    const ws = new WebSocket(`${location.protocol === "https:" ? "wss" : "ws"}://${location.host}/ws/live`);
    ws.onopen = () => { delay = 1000; onStatus(true); };
    ws.onmessage = (ev) => { try { onMessage(JSON.parse(ev.data)); } catch { /* mensagem inválida */ } };
    ws.onclose = (ev) => {
      onStatus(false, ev.code);
      if (ev.code === 4401) return;          // aparelho não pareado (ou pareamento revogado): não insiste
      setTimeout(open, delay);
      delay = Math.min(delay * 2, 5000);
    };
  };
  open();
}
```

O que observar:

- **Fábrica** `create_app(core)`: nada global, testável com um núcleo falso.
- **Guard** de Host (DNS rebinding) e Origin (CSRF), admin só por token **e** cliente local, `compare_digest` em bytes.
- **`full` e depois `diff`**: o cliente recebe o estado inteiro uma vez e depois só o que mudou; `None` significa "saiu".
- **`serve_until_disconnect`**: escutar a desconexão em paralelo é o que deixa o servidor reiniciar sem esperar canais parados.
- **`StaticFiles` por último**, `log_config=None` e `lifespan="off"`.
- `snapshot()` é chamado 30x/s por cliente: mantenha-o barato (ler memória, nunca I/O).

---

## 6. Widget de canvas 2D com `devicePixelRatio`

### 6.1 O ajudante `criarCanvas` (buffer em pixels do aparelho, desenho só quando a chave muda)

Fonte: `hub/web/painel/widgets/canvas.js:34-78`

```js
export function criarCanvas(cell, env = {}) {
  const dprDe = () => (typeof env.dpr === "function" ? env.dpr() : 1);
  const canvas = document.createElement("canvas");
  canvas.className = "w-canvas";
  setStyle(canvas, "width", "100%");
  setStyle(canvas, "height", "100%");
  cell.append(canvas);
  const ctx = typeof canvas.getContext === "function" ? canvas.getContext("2d") : null;
  const size = { w: 0, h: 0, dpr: 0 };
  let ultima = null, sujo = true, paleta = null, paletaDe = null;

  const api = {
    canvas, ctx, size,
    // lê o tamanho da célula; true se o buffer mudou (e, portanto, precisa desenhar de novo)
    medir() {
      const w = cell.clientWidth || 0, h = cell.clientHeight || 0;
      const d = Number(dprDe()) > 0 ? Number(dprDe()) : 1;
      if (w === size.w && h === size.h && d === size.dpr) return false;
      size.w = w; size.h = h; size.dpr = d;
      if (!(w > 0 && h > 0) || !ctx) return false;
      canvas.width = Math.round(w * d);
      canvas.height = Math.round(h * d);
      ctx.setTransform(d, 0, 0, d, 0, 0);
      sujo = true;
      return true;
    },
    // true se desenhou
    desenhar(chave, fn) {
      if (!ctx || !(size.w > 0 && size.h > 0)) return false;
      const k = `${chave}|${estiloAtual()}`;
      if (!sujo && k === ultima) return false;
      ultima = k; sujo = false;
      ctx.clearRect(0, 0, size.w, size.h);
      fn(ctx, size, api.paleta());
      return true;
    },
    invalidar() { sujo = true; },
    paleta() {
      const e = estiloAtual();
      if (!paleta || paletaDe !== e) { paleta = lerPaleta(); paletaDe = e; }
      return paleta;
    },
  };
  return api;
}
```

### 6.2 Um widget real que o usa (desenho do mostrador omitido)

Fonte: `hub/web/painel/widgets/ponteiro.js:14-32` e `86-97`. O corpo do `cv.desenhar(...)` (linhas 33-83 do original) foi
substituído por `// ...`.

```js
export function mount(cell, widget, env) {
  const info = campoInfo(widget.campo), o = widget.opcoes || {};
  const rotulo = o.rotulo ?? info.rotulo ?? "", unidade = o.unidade ?? info.unidade ?? "";
  const fmt = { formato: info.formato, casas: o.casas ?? info.casas ?? 0, escala: info.escala ?? 1 };
  const cv = criarCanvas(cell, env);
  let ultimo = null;                      // o que a última atualização pediu para desenhar

  function escala(estado) {
    const min = o.min ?? info.min ?? 0;
    const max = o.max ?? (typeof info.max === "string" ? fieldValue(estado, info.max) : info.max) ?? null;
    return [min, max];
  }

  function desenhar() {
    if (!ultimo) return;
    const { v, min, max, texto, faixas } = ultimo;
    const ang = anguloDoValor(v, min, max);
    const chave = [ang === null ? "-" : Math.round((ang / RAD) * 2), texto, min, max, JSON.stringify(faixas)].join("|");
    cv.desenhar(chave, (ctx, size, p) => {
      // ... (linhas 33-83 do original: o desenho do mostrador, usando `ctx`, `size` e a paleta `p`)
    });
  }

  return {
    update(estado) {
      const v = fieldValue(estado, widget.campo);
      const [min, max] = escala(estado);
      const okMax = typeof max === "number" && Number.isFinite(max) ? max : null;
      const faixas = Array.isArray(o.faixas) && o.faixas.length ? o.faixas : widget.campo === "rpm" ? zonaVermelhaPadrao(okMax) : [];
      ultimo = { v: typeof v === "number" ? v : null, min, max: okMax, texto: formatValor(v, fmt) || DASH, faixas };
      desenhar();
    },
    resize() { if (cv.medir()) desenhar(); },
  };
}
```

O que observar:

- `medir()` só toca o buffer se largura, altura **ou densidade** mudaram; `canvas.width = ...` apaga o canvas.
- Depois do `setTransform(d, ...)` o widget desenha em pixels de CSS.
- A chave de redesenho (`chave`) é uma string barata com o que importa (aqui, o ângulo em passos de meio grau, o texto, a escala e as faixas).
- Nenhuma cor literal: a paleta `p` vem dos tokens do estilo.
- `resize()` redesenha só quando `medir()` devolveu `true`.

### 6.3 Buffer circular para séries ao vivo

Fonte: `hub/web/painel/buffer.js:3-19`

```js
export function criarBuffer(capacidade) {
  const cap = Number.isFinite(capacidade) && capacidade >= 1 ? Math.floor(capacidade) : 1;
  const dados = new Array(cap);
  let inicio = 0, n = 0;
  return {
    capacidade: cap,
    get tamanho() { return n; },
    push(item) {
      if (n < cap) { dados[(inicio + n) % cap] = item; n++; } else { dados[inicio] = item; inicio = (inicio + 1) % cap; }
    },
    limpar() { inicio = 0; n = 0; dados.fill(undefined); },
    primeiro() { return n ? dados[inicio] : undefined; },
    ultimo() { return n ? dados[(inicio + n - 1) % cap] : undefined; },
    paraCada(fn) { for (let i = 0; i < n; i++) fn(dados[(inicio + i) % cap], i); },
    itens() { const out = []; for (let i = 0; i < n; i++) out.push(dados[(inicio + i) % cap]); return out; },
  };
}
```

---

## 7. Carregamento de modelo contra corrida (`loadSeq`) e descarte

Fonte: `hub/web/js/webgl.js:43-51` (`disposeTree`) e `289-322` (`loadModel`, `loadExported`)

```js
function disposeTree(obj) {
  obj.traverse((o) => {
    o.geometry?.dispose();
    for (const m of [].concat(o.material || [])) {
      for (const v of Object.values(m)) if (v && v.isTexture) v.dispose();
      m.dispose();
    }
  });
}
```

```js
  // .glb listado em models.json, senão o modelo em código; o anterior é removido e descartado.
  // Com source ({url, frente, topo}): modelo exportado do Fusion; se não carregar, o volante em código.
  async function loadModel(name, source = null) {
    if (dead) return;
    if (source?.url) { await loadExported(name, source); return; }
    const url = modelUrl(name);
    if (!url) { clearModel(); return; }
    const seq = ++loadSeq;                       // antes do atalho: um carregamento antigo de outro modelo não entra
    if (name === currentName && !currentUrl) return;
    let model = null;
    if ((await availableModels()).includes(name)) {
      try {
        model = (await new GLTFLoader().loadAsync(url)).scene;
      } catch (e) {
        console.warn(`Modelo ${name} não carregou; usando o modelo em código.`, e);
      }
    }
    if (dead || seq !== loadSeq) { if (model) disposeTree(model); return; }   // saiu da página ou trocou de modelo
    swap(model || build(name), name);
  }

  async function loadExported(name, source) {
    const seq = ++loadSeq;
    if (name === currentName && source.url === currentUrl) return;
    let model = null;
    try {
      model = prepareExported((await new GLTFLoader().loadAsync(source.url)).scene, source);
    } catch (e) {
      console.warn(`Modelo ${name} não carregou; usando o volante em código.`, e);
    }
    if (dead || seq !== loadSeq) { if (model) disposeTree(model); return; }
    if (model) swap(model, name, source.url);
    else swap(build(source.fallback || "volante.glb"), source.fallback || "volante.glb");
  }
```

O que observar: `loadSeq` é incrementado **antes** do primeiro `await`; depois do `await`, `seq !== loadSeq` descarta e libera o
resultado atrasado; se o `.glb` falhar, entra um modelo em código (plano B visual) em vez de um erro na tela.

---

## 8. Carregar o 3D sob demanda (hero)

Fonte: `hub/web/js/hero.js:25-44`

```js
  // 3D sob demanda: sem Three.js ou sem WebGL ficam só as leituras
  const ready = (async () => {
    try {
      const mod = await import("./webgl.js");
      viewer = mod.createViewer(stage, { interactive: true, layout: "center",
        orbit: window.matchMedia("(pointer: fine)").matches, zoom: false, style });
      if (viewer) await load(model);
    } catch (e) {
      console.warn("Volante 3D desligado:", e);
      viewer = null;
    }
    root.classList.toggle("no3d", !viewer);
    return viewer;
  })();

  // sem modelo exportado (ou se ele não carregar), o volante em código
  async function load(m) {
    await viewer.loadModel(m?.key || "volante.glb", m?.url ? m : null);
    if (follow && last != null) viewer.rotateModel(0, -last);
  }
```

O que observar: `await import("./webgl.js")` dentro do `try`: sem Three.js ou sem WebGL, a página continua com as leituras em texto
(`root.classList.toggle("no3d", !viewer)` esconde o palco no CSS).
