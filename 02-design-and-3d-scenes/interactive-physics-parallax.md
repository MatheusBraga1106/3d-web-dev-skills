# Interação, "física" e parallax

A matemática real do projeto de referência: como o volante 3D segue o ângulo medido, como as juntas e os pedais se mexem,
como o mouse e a rolagem entram, e como o desenho só acontece quando algo muda. Referências `arquivo:linhas` valem para o
commit `ff34fab` (versão 0.2.1).

## 1. O que existe e o que não existe neste projeto

| Tópico | Situação |
|---|---|
| Seguir um valor medido (ângulo, pedais, posição da manopla) | Existe e é o centro do projeto. **Sem suavização própria**: o valor é aplicado direto, com um limiar (epsilon) para ignorar chiado. |
| Física de verdade (massa, mola, amortecimento) | **Não existe neste projeto.** O "amortecimento" que existe é o do Lenis (rolagem) e o do `gsap.quickTo` (cursor). Os controles de força/mola/amortecimento que aparecem no hub são ajustes da base de volante, enviados ao hardware; nada a ver com o 3D. |
| Flutuação por `Math.sin` (objeto "respirando") | **Não existe neste projeto.** O único `Math.sin` ligado à interface está em `sectionTone` (tom do fundo de cada seção, `poses.js:197`); os outros dois (`painel/gauge.js`, `painel/widgets/ponteiro.js`) são geometria do ponteiro analógico. Nada flutua com o tempo. Na seção 8 há uma **proposta** de como encaixar uma flutuação no laço existente. |
| Giro automático (idle) | Existe no `webgl.js` (`IDLE_DEG_PER_S = 10`, `setIdle`), mas **nenhuma página o liga hoje**: `poseFor` ainda devolve `{ idle: true }` para o aro sem base conectada, porém ninguém chama `viewer.setIdle`. É código disponível, não comportamento ativo. |
| Inércia no OrbitControls | Não: `enableDamping = false`, de propósito. |
| Parallax de mouse (cena reage à posição do cursor) | Não. O mouse só (a) orbita a câmera quando se arrasta e (b) move o cursor em ponto. |
| Parallax de rolagem | Existe (GSAP ScrollTrigger com `scrub`). |
| Rolagem suave | Existe (Lenis 1.3.26, `lerp: 0.12`). |
| Animação por tempo (`setTimeout`, `gsap.to` sem scroll) | Quase nada. Regra do projeto: "nada é animado por tempo: tudo segue a rolagem ou o mouse" (comentário de `motion.js:1-4`). Exceções: o fade `.18s` em CSS e o giro idle. |

## 2. Constantes de epsilon e limites (a tabela que importa)

| Constante | Valor | Onde | Significado |
|---|---|---|---|
| `WHEEL_EPS_DEG` | `0.2` | `poses.js:7` | Variação de ângulo do volante menor que 0,2 graus não redesenha. |
| `KNOB_EPS` | `0.005` | `poses.js:8` | Variação da manopla (fração de curso) menor que isso não redesenha. |
| `PEDAL_EPS` | `0.005` | `poses.js:73` | Variação de pedal menor que 0,5% não redesenha. |
| `JOINT_EPS` | `0.001745` rad (0,1 grau) | `joints.js:9` | Variação de junta menor que 0,1 grau não redesenha. |
| `MAX_FPS` | `30` | `webgl.js:18` | Teto de quadros por segundo do 3D. |
| `MAX_DPR` | `1.5` | `webgl.js:18` | Teto do `devicePixelRatio` do renderer. |
| `IDLE_DEG_PER_S` | `10` | `webgl.js:18` | Giro automático (disponível, sem uso ativo). |
| `KNOB_X`, `KNOB_Z`, `PLATE_Y` | `0.8`, `0.6`, `0.07` | `poses.js:9` | Meia largura entre colunas, meio curso, topo da placa. |
| `PEDAL_REST_DEG`, `PEDAL_TRAVEL_DEG` | `14`, `30` | `poses.js:73` | Pedal solto a +14 graus; curso de 30 graus. |
| `PEDAL_VIEW_DEG` | `-32` | `poses.js:75` | Pedais em 3/4 (grupo interno, giro em Z). |

A função de comparação, `poses.js:56-58`:

```js
export function moved(prev, next, eps) {
  return Math.abs(next - prev) >= eps;
}
```

Detalhe sutil e correto: o `prev` é o último valor **aplicado**, não o último recebido. Se o ângulo deriva devagar (0,05 grau
por mensagem), o `pose` não muda até a diferença acumulada chegar a 0,2 graus; aí atualiza e o ciclo recomeça. O ruído
não gera quadros e a deriva real nunca fica "presa".

## 3. O volante seguindo `angle_deg`

### 3.1 Caminho do dado

```
base (HID) --> plug-in ffbeast.live()["angle_deg"] --> /ws/live (diff a até 30 Hz) --> app.js onLive()
   --> hero.update(state.live) --> heroReadout(live).angleDeg --> viewer.rotateModel(0, -angleDeg)
   --> (rotateModel: moved? sim -> pose.y = valor; request()) --> tick: current.rotation.set(deg(pose.x), deg(pose.y), 0)
```

- `heroReadout` (`poses.js:116-130`) só aceita `lv.angle_deg` numérico finito e só com a base conectada; senão devolve
  `angleDeg: null` e o hero **mantém a última pose**. Dados estranhos (`NaN`, string, `null`) são ignorados; há testes para
  isso em `tests/js/poses.test.mjs` (`poseFor`/`heroReadout`).
- `hero.js:46-53`:

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

O sinal negativo vem da convenção do modelo: o eixo Y aponta para o piloto, então girar no sentido horário (visto pelo
piloto) é uma rotação **negativa** em Y. Foi conferido na prática contra o volante físico.

- `rotateModel` (`webgl.js:341-347`):

```js
function rotateModel(x, y) {
  if (dead || !Number.isFinite(x) || !Number.isFinite(y)) return;
  if (!moved(pose.x, x, WHEEL_EPS_DEG) && !moved(pose.y, y, WHEEL_EPS_DEG)) return;
  pose.x = x;
  pose.y = y;
  request();
}
```

- E no laço (`webgl.js:267-269`): `root.rotation.y = deg(cfg.yaw + pose.yaw); current.rotation.set(deg(pose.x), deg(pose.y), 0);`.

Não há `lerp` entre o ângulo antigo e o novo: a base entrega até 30 amostras por segundo (o canal ao vivo é um laço de
`interval = 1/30`, `server/live.py:52`), e o 3D redesenha a até 30 quadros por segundo, então o movimento já parece contínuo.

### 3.2 O modo de edição para o seguimento

`setFollow(on)` (`hero.js:60-63`) interrompe o seguimento enquanto o usuário arrasta um rótulo sobre o volante:
`viewer.rotateModel(0, follow && last != null ? -last : 0)`. Voltar a seguir reaplica o último ângulo guardado em `last`.

### 3.3 A manopla do câmbio (código)

`poses.js:149-157` converte a leitura normalizada `norm = (u, v)` (coluna 0 a 2, linha -1 a +1) em posição:

```js
return { knob: { x: clamp(n[0] - 1, -1.6, 1.6), y: clamp(n[1], -1.4, 1.4) } };
```

O laço aplica (`webgl.js:270-271`): `knob.position.set(pose.knob.x * KNOB_X, knob.position.y, -pose.knob.y * KNOB_Z)`. Os limites
±1,6 e ±1,4 deixam passar um pouco além das marchas (a manopla "estica" além do engate sem sair da placa).
`gearPoints()` (`poses.js:60-62`) calcula os pontos dos rótulos das marchas na mesma conta:
`[(u - 1) * KNOB_X, PLATE_Y, -v * KNOB_Z]`.

## 4. Pedais

### 4.1 Pedais em código (ângulo fixo por pedal)

```js
export const PEDAL_REST_DEG = 14, PEDAL_TRAVEL_DEG = 30, PEDAL_EPS = 0.005;
export function pedalAngle(v) {
  return PEDAL_REST_DEG - (finite(v) ? clamp(v, 0, 1) : 0) * PEDAL_TRAVEL_DEG;
}
```

`poses.js:73,87-89`. Solto (v = 0) a +14 graus; apertado (v = 1) a -16 graus. O laço aplica por nome do objeto
(`webgl.js:272-275`): `current.getObjectByName("pedal-" + k).rotation.x = deg(pedalAngle(pose.pedals[k]))`, com
`PEDAL_KEYS = ["clutch", "brake", "accelerator"]`.

`pedalValues(entry)` (`poses.js:78-85`) transforma as leituras em frações:

```js
const inp = entry?.health?.connected ? entry.live?.inputs : null;
if (!inp?.ok) return null;                       // base desconectada ou sem leitura -> nada se mexe
return Object.fromEntries(PEDAL_KEYS.map((k) => {
  const v = inp.pedals?.[k];
  return [k, finite(v) ? clamp(v / 100, 0, 1) : null];   // a base entrega 0..100
}));
```

`setPedals(values)` (`webgl.js:357-365`) limita a `[0, 1]` e só pede quadro se algum pedal passou do `PEDAL_EPS`.

### 4.2 Pedais exportados (juntas)

Ver seção 5: com `movimento` no perfil, a página chama `viewer.setJoints(jointTargets(movimento, "pedais", v || {}))`
(`pages/pedals.js:43`) em vez de `setPedals`.

## 5. As juntas: `jointTargets`, `setJointAngle`, repouso e limites

### 5.1 Dado de entrada: o bloco `movimento` do `perfil.json`

Cada junta: `{ nome, papel, pai, pecas, origem, eixo, limites: [min, max], atual, repouso }` (ângulos em rad, origem em metros
no referencial do `.glb`). Papéis: câmbio `lados` e `frente`; pedais `embreagem`, `freio`, `acelerador`
(`PEDAL_ROLE = { embreagem: "clutch", freio: "brake", acelerador: "accelerator" }`).

### 5.2 `jointTargets` (puro, testado): dado ao vivo -> ângulo alvo por junta

`poses.js:169-191`:

```js
export function jointTargets(movimento, tipo, valores, inverter = []) {
  ...
  for (const j of juntas) {
    ...
    const lim = j.limites;
    if (!Array.isArray(lim) || lim.length !== 2 || !finite(lim[0]) || !finite(lim[1])) continue;
    if (tipo === "cambio" && (j.papel === "lados" || j.papel === "frente")) {
      const eixo = j.papel === "lados" ? val.x : val.y;
      let f = finite(eixo) ? clamp(eixo, -1, 1) : 0;
      if (inv.includes(j.papel)) f = -f;
      out[j.nome] = (lim[0] + lim[1]) / 2 + f * (lim[1] - lim[0]) / 2;
    } else if (tipo === "pedais" && PEDAL_ROLE[j.papel] && finite(j.repouso)) {
      const v = val[PEDAL_ROLE[j.papel]];
      const f = finite(v) ? clamp(v, 0, 1) : 0;
      const outro = Math.abs(lim[0] - j.repouso) >= Math.abs(lim[1] - j.repouso) ? lim[0] : lim[1];
      out[j.nome] = j.repouso + f * (outro - j.repouso);
    }
  }
  return out;
}
```

Fórmulas:

- **Câmbio:** `alvo = meio + f * (máx − mín) / 2`, `meio = (mín + máx) / 2`, `f = clamp(eixo, -1, 1)` (negado se a inversão
  daquele papel estiver ligada). O **ponto morto é o meio dos limites** e as marchas vão até os limites. Exemplo real do
  câmbio testado: `lados` com limites `[-0,2618; +0,2618]` rad (±15 graus): `x = 1` dá `+0,2618`, `x = -1` dá `-0,2618`;
  `frente` com `[-0,6109; -0,0873]` (de -35 a -5 graus) tem meio em `-0,3491` rad (-20 graus).
- **Pedais:** `alvo = repouso + f * (outro − repouso)`, onde `outro` é o limite **mais distante** do repouso (o pedal é
  modelado solto, então apertado é o limite oposto) e `f` é a fração `[0, 1]`. Exemplo: repouso `0`, limites
  `[-0,4; 0]`: `outro = -0,4`; `f = 0,5` dá `-0,2`.
- Valor que não é número vira `f = 0`: câmbio no ponto morto, pedal solto. Dado estranho no todo devolve `{}`.

### 5.3 `mountJoints` e `setJointAngle`: onde o ângulo vira rotação

`joints.js` (todo o arquivo, 80 linhas). Montagem, uma vez, na carga do modelo (`prepareExported`, **antes** de orientar e
escalar):

1. Para cada junta válida (mãe antes da filha; ciclos e dados quebrados são ignorados), cria um `Group` `junta:<nome>` em
   `position = origem`.
2. Se tem mãe, `mae.obj.attach(obj)` (mantém a posição no mundo): a filha anda junto com a mãe.
3. O eixo é levado do referencial do modelo ao do pai da articulação (`transformDirection` duas vezes) e guardado em
   `joints[nome] = { obj, eixo, atual, quat0, ang: 0 }`.
4. Cada corpo do glTF (o nó com `extras.caminho`) vai para a articulação **mais funda** cujo conjunto de peças o contém
   (`joint.obj.attach(corpo)`).

Aplicação, por quadro de dado (`joints.js:73-80`):

```js
export function setJointAngle(joint, alvo) {
  if (!joint || !finite(alvo)) return false;
  const ang = alvo - joint.atual;                       // o modelo foi exportado na posição "atual"
  if (Math.abs(ang - joint.ang) < JOINT_EPS) return false;
  joint.ang = ang;
  joint.obj.quaternion.copy(q.setFromAxisAngle(joint.eixo, ang)).multiply(joint.quat0);
  return true;
}
```

- A rotação é `alvo − atual`, em torno do eixo, pelo pivô (a posição do grupo), a partir da posição desenhada. `atual` é o
  ângulo da junta no momento da exportação.
- Retorna `true` só se mexeu ≥ 0,1 grau; `webgl.js:369-383` (`setJoints`/`applyJoints`) só chama `request()` se algum
  retornar `true`. `setJoints` a 100 Hz com o mesmo valor não gera quadro nenhum.
- Os alvos pedidos ficam guardados em `targets`; se o modelo chegar depois, `swap()` chama `applyJoints()` e a pose já
  nasce certa.
- Nomes de junta que o modelo não tem são ignorados.

## 6. OrbitControls e o mouse

`webgl.js:148-156`:

```js
const controls = interactive && orbit ? new OrbitControls(camera, canvas) : null;   // sem orbit (toque): a rolagem da página passa
if (controls) {
  controls.enablePan = false;
  controls.enableZoom = zoom;            // sem zoom: a roda do mouse rola a página
  controls.enableDamping = false;        // sem inércia: nada fica desenhando sozinho depois de soltar
  controls.minDistance = cfg.dist * 0.5;
  controls.maxDistance = cfg.dist * 1.6;
  controls.addEventListener("change", request);
}
```

- **Quem liga o orbit:** o hero passa `orbit: window.matchMedia("(pointer: fine)").matches` e `zoom: false` (`hero.js:29-30`).
  Em telas de toque não há OrbitControls, para o gesto de rolar passar. Os cards do Câmbio e dos Pedais usam o padrão
  (`orbit = interactive`, `zoom = true`).
- **Sem pan; zoom entre 50% e 160% da distância do layout** (`center`: de 4 a 12,8).
- **Sem inércia:** `enableDamping = false` garante que, ao soltar o mouse, o 3D para de pedir quadros na hora; só o evento
  `change` pede um quadro (`controls.addEventListener("change", request)`).
- O orbit move a **câmera**; o ângulo do volante gira o **modelo**. As duas coisas são independentes e se combinam.
- Cartão do visualizador: `touch-action: none` (`.viewer-card`, `app.css:6`) para o navegador não roubar o gesto.
- Seleção de ponto com o mouse (editor de posições): `pickPart(clientX, clientY)` converte o pixel em NDC
  (`ndc.set(((x − left) / width) * 2 − 1, −((y − top) / height) * 2 + 1)`), faz `raycaster.setFromCamera`, e usa a primeira
  malha atingida (`meshHit`); fora das peças cai no plano da face. Resultado em coordenadas do modelo arredondadas a 3 casas
  e com limite de ±10 (`PICK_LIMIT`).

## 7. Lenis, GSAP ScrollTrigger e o cursor

Tudo em `hub/web/js/motion.js`, carregado sob demanda por `app.js:98-106` (`await import("./motion.js")`).

### 7.1 Lenis (rolagem suave)

```js
const lenis = new Lenis({ autoRaf: false, lerp: 0.12 });
lenis.on("scroll", ScrollTrigger.update);
gsap.ticker.add((time) => lenis.raf(time * 1000));
gsap.ticker.lagSmoothing(0);
```

- `autoRaf: false`: o Lenis não cria o próprio `requestAnimationFrame`; o **`gsap.ticker`** o aciona, para o ScrollTrigger e o
  Lenis andarem no mesmo relógio. `lagSmoothing(0)` desliga a compensação de lag do GSAP (não "pula" tempo).
- `lerp: 0.12`: no código vendorizado (`vendor/lenis/lenis.mjs:37-38,86`) isso vira
  `valor = lerp(valor, alvo, 1 − exp(−0,12 · 60 · dt))`. A 60 quadros por segundo, cada quadro percorre cerca de 11% do que falta
  (`1 − e^(−0,12)`); a fórmula usa `dt`, então a suavização é independente da taxa de quadros.
- `scrollTo`: `lenis.scrollTo(topOf(target), { immediate: !smooth, force: true })`. Passa-se um **número** em pixels, não o
  elemento, porque o Lenis descontaria o `scroll-margin-top` da seção uma segunda vez.
- Posição da seção no fluxo (`motion.js:10-17`): as seções estão `sticky`, então `getBoundingClientRect().top` não diz onde
  elas começam. `flowTop` soma as alturas dos irmãos anteriores: `y = pai.top + scrollY + Σ offsetHeight(irmãos anteriores)`;
  `topOf(target) = flowTop(target) − topbar()`.
- `lock(on)`: com a folha aberta, `lenis.stop()`; ao fechar, `lenis.start()`.

### 7.2 ScrollTrigger: parallax do hero (só ≥ 761 px)

```js
gsap.matchMedia().add("(min-width: 761px)", () => {
  // o volante recua, diminui e esmaece enquanto a primeira seção sobe por cima dele
  gsap.to(stage, { yPercent: 14, scale: 0.86, opacity: 0.4, ease: "none",
    scrollTrigger: { trigger: sections, start: "top bottom", end: "top top", scrub: true } });
  gsap.to(edges, { yPercent: -30, opacity: 0, ease: "none",
    scrollTrigger: { trigger: sections, start: "top 85%", end: "top 35%", scrub: true } });
});
```

`motion.js:101-107`. `stage` é `.hero-stage` (o canvas) e `edges` são `.hero-left` e `.hero-right`. Valores do parallax:

| Alvo | Propriedades (início para fim) | Faixa da rolagem |
|---|---|---|
| Canvas do hero | `yPercent` 0 -> 14, `scale` 1 -> 0,86, `opacity` 1 -> 0,4 | `top bottom` -> `top top` do bloco de seções |
| Texto das bordas | `yPercent` 0 -> -30, `opacity` 1 -> 0 | `top 85%` -> `top 35%` |

`ease: "none"` e `scrub: true`: o progresso é exatamente a posição da rolagem, sem tempo próprio. A suavização vem do Lenis, não do
`scrub` (que aqui é `true`, não um número).

### 7.3 ScrollTrigger: entrada das seções e `--cover`

```js
for (const s of sections.querySelectorAll(".sec")) {
  if (s.dataset.motion) continue;                 // já tem gatilho
  s.dataset.motion = "1";
  gsap.fromTo(s.querySelector(".page-inner"), { y: 48, opacity: 0.25 }, { y: 0, opacity: 1, ease: "none",
    scrollTrigger: { trigger: s, start: "top bottom", end: "top 60%", scrub: true } });
  const next = s.nextElementSibling;
  if (next) gsap.fromTo(s, { "--cover": 0 }, { "--cover": 1, ease: "none",
    scrollTrigger: { trigger: next, start: "top bottom", end: () => `top ${topbar()}px`, scrub: true } });
}
```

`motion.js:112-123`. O conteúdo de cada seção sobe 48 px e vai de 25% a 100% de opacidade entre "o topo da seção toca o fim da
tela" e "o topo está a 60% da tela". `--cover` vai de 0 a 1 enquanto a **seguinte** sobe por cima (o `end` é uma função, para
reavaliar a altura real da barra). Antes disso, `reveal()` mata os gatilhos órfãos: `for (const t of ScrollTrigger.getAll())
if (!document.contains(t.trigger)) t.kill(true)`. É o que permite remontar as seções (a lista de aparelhos mudou) sem
vazar gatilhos. `refresh()` = `restack(); reveal(); ScrollTrigger.refresh()`.

### 7.4 `stickTop`

`poses.js:109-111`, função pura testada: `Math.min(bar, viewport − height)` (ou `bar` se algum valor não for finito). Seção
curta: gruda em `bar`. Seção mais alta que a janela: `viewport − height` é negativo, e ela só "gruda" quando o rodapé chega ao
fim da tela. Veja a seção 5.5 de `ui-tokens-and-overlay.md`.

### 7.5 Cursor em ponto

`motion.js:65-78`:

```js
const toX = gsap.quickTo(dot, "x", { duration: 0.2, ease: "power3" });
const toY = gsap.quickTo(dot, "y", { duration: 0.2, ease: "power3" });
window.addEventListener("pointermove", (e) => {
  if (e.pointerType !== "mouse") return;
  toX(e.clientX); toY(e.clientY);
  dot.classList.toggle("off", Boolean(e.target.closest?.(INTERACTIVE)));   // sobre controles volta o cursor normal
}, { passive: true });
document.documentElement.addEventListener("pointerleave", () => dot.classList.add("off"));
```

Só com `pointer: fine`. O ponto tem 8 px, `mix-blend-mode: difference` (visível sobre claro e escuro) e fica a `z-index: 100`. A lista
`INTERACTIVE` (`a, button, input, select, textarea, label, summary, dialog, [role=button], .chip, .tile.editable`) devolve o
cursor nativo sobre controles.

### 7.6 Sem as bibliotecas ou com "reduzir movimento"

`startMotion` retorna `nativeMotion(sections, smoothOk)` em duas situações: `prefers-reduced-motion: reduce` (aí
`smoothOk = false`) ou falha ao carregar GSAP/Lenis (`smoothOk = true`). Nele: rolagem nativa, sem cursor próprio, seções
empilhadas continuam (`stack()`), e cada seção ganha um fade quando entra na tela via `IntersectionObserver({ threshold: 0.05 })`.
A interface devolvida é a mesma (`refresh`, `scrollTo`, `lock`), então `app.js` não precisa saber qual está ativa.

## 8. Desenhar só quando muda, 30 quadros por segundo, e a pausa com a aba escondida

O laço de render do visualizador não é contínuo. É um `requestAnimationFrame` **sob demanda** (`webgl.js:158-160,260-281`):

```js
function request() {
  if (!dead && !raf) raf = requestAnimationFrame(tick);
}

function tick(t) {
  raf = 0;
  if (dead || document.hidden) return;         // escondida: nada é desenhado; volta no visibilitychange
  if (lastFrame && t - lastFrame < 1000 / MAX_FPS - 2) { raf = requestAnimationFrame(tick); return; }   // até 30/s
  const spinning = idle && current && !reduceMotion;
  if (spinning && lastFrame) pose.yaw += IDLE_DEG_PER_S * Math.min(0.1, (t - lastFrame) / 1000);
  lastFrame = t;
  root.rotation.y = deg(cfg.yaw + pose.yaw);
  if (current) { /* aplica pose, manopla, pedais */ }
  renderer.render(scene, camera);
  placeLabels();
  frames++;
  if (spinning) raf = requestAnimationFrame(tick);
}
```

Regras:

1. **Quem pede quadro:** `request()` é chamado por tudo que muda algo visível (`rotateModel`, `setKnob`, `setPedals`,
   `setJoints`, `setGlow`, `setStyle`, `setLabels`, `moveLabel`, `resize`, evento `change` do OrbitControls, troca de modelo).
   Se já há um `raf` pendente, não agenda outro.
2. **Quem mantém o laço vivo:** só o giro idle (`if (spinning) raf = requestAnimationFrame(tick)`). Sem ele, o quadro termina e
   o navegador fica **100% ocioso** até o próximo `request()`.
3. **Teto de 30/s:** se o quadro chegou antes de `1000/30 − 2` ms desde o último, reagenda sem desenhar (o `− 2` absorve o jitter
   do relógio do rAF para não cair para 20 fps).
4. **Aba escondida:** `document.hidden` -> não desenha e não reagenda; ao voltar, `visibilitychange` zera `lastFrame` e chama
   `request()` (`webgl.js:181-182`). O intervalo `lastFrame = 0` evita um "salto" no giro idle.
5. **`dt` limitado:** `Math.min(0.1, (t − lastFrame) / 1000)` impede que um engasgo (aba em segundo plano) gire o modelo de
   uma vez.
6. **`reduceMotion`** (`webgl.js:30`, lido uma vez) desliga só o giro idle.
7. **Contadores para teste:** `stats().frames` conta quadros desenhados; é o que permite provar que "dado repetido não
   desenha".

No painel de telemetria (canvas 2D), a mesma ideia: o laço (`painel/panel.js`) desenha no máximo um quadro por `tick`, só chama
`update()` dos widgets cujos campos mudaram, e `desenhar(chave, fn)` do `widgets/canvas.js` só redesenha quando a chave dos
dados muda. Ver `webgl-utils-and-perf.md`.

## 9. PROPOSTA (não existe no projeto): flutuação por senoide no laço existente

> **Isto é uma proposta, não código do projeto de referência.** Os números abaixo (amplitude, frequência) são escolhas
> sugeridas, não valores medidos de lá.

Para um objeto "respirando" (por exemplo um modelo flutuante num hero), encaixe uma fase acumulada no mesmo `tick`, ao lado do
giro idle, respeitando as quatro invariantes do projeto: teto de 30/s, pausa com a aba escondida, `reduceMotion` e descarte do
`dt` grande.

```js
// PROPOSTA: constantes novas ao lado de MAX_FPS / IDLE_DEG_PER_S
const FLOAT_AMP = 0.08;     // unidades de cena (o modelo tem ~3,4 de maior dimensão): ~2%
const FLOAT_HZ = 0.25;      // um ciclo a cada 4 s
let floating = false, floatPhase = 0;

// dentro de tick(t), depois de calcular `dt` como no giro idle:
const bobbing = floating && current && !reduceMotion;
if (bobbing && lastFrame) floatPhase += 2 * Math.PI * FLOAT_HZ * Math.min(0.1, (t - lastFrame) / 1000);
stage.position.y = bobbing ? Math.sin(floatPhase) * FLOAT_AMP : 0;     // no `stage`, NUNCA em `current`
// ...render como hoje...
if (spinning || bobbing) raf = requestAnimationFrame(tick);

// API, no mesmo estilo de setIdle:
function setFloat(on) {
  if (dead || floating === Boolean(on)) return;
  floating = Boolean(on);
  lastFrame = 0;
  request();
}
```

Por que assim:

- **Fase acumulada, não `Math.sin(t)`**: com `t` absoluto, depois de uma pausa (aba escondida) o objeto "teleporta" para outra
  altura. Acumulando `fase += 2π·f·dt` com `dt` limitado, ele retoma de onde parou.
- **No `stage` (ou no `root`), não no `current`**: o `tick` reescreve `current.rotation` todo quadro, e `current` é o objeto
  que `prepareExported` embrulhou; a posição do `stage` não interfere na rotação do volante nem nos pivôs das juntas. Os
  rótulos acompanham porque `placeLabels()` roda depois do `render`.
- **Custo:** quebra a regra "ocioso quando nada muda": com flutuação ligada o laço roda a até 30 quadros por segundo o tempo
  todo. Ligue-a só no objeto que de fato precisa (um `viewer` por vez), desligue-a (`setFloat(false)`) quando o objeto estiver
  coberto pelas seções (por exemplo, no `onUpdate` do ScrollTrigger quando o progresso do parallax chegar a 1), e confira o
  FPS no cenário de uso real (ver o checklist em `01-planning/3d-feature-workflow.md`).
- **Alternativa sem JS por quadro:** para flutuação de um elemento HTML (não do 3D), prefira uma animação CSS
  `@keyframes` com `transform: translateY(...)` no compositor; ela já respeita `prefers-reduced-motion` com a regra global do
  `app.css:204-206`.
- **Teste:** como `moved`/`pedalAngle`, deixe a conta da senoide numa função pura (`floatOffset(phase, amp)`) e teste no
  `node --test`; a parte do laço fica coberta por um teste estático de texto como os de `tests/test_web_static.py`.
