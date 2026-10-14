# Cena 3D, câmera, luzes e materiais

Valores reais do visualizador do projeto de referência (`hub/web/js/webgl.js`, `hub/web/js/models3d.js`,
`hub/web/js/poses.js`), com Three.js **0.186.1** vendorizado em `hub/web/vendor/three/` (sem bundler, via `importmap`).
Referências `arquivo:linhas` valem para o commit `ff34fab` (versão 0.2.1).

## 1. O que existe e o que não existe neste projeto

| Tópico | Situação |
|---|---|
| Luzes `AmbientLight` + 2 `DirectionalLight` | Existe (seção 3). |
| `RoomEnvironment` + PMREM | Existe, **só** no estilo "renderizado" e só para modelos exportados do Fusion. |
| Tone mapping ACES | Existe, **só** no estilo renderizado. No wireframe é `NoToneMapping`. |
| Wireframe com arestas | Existe (`LineSegments`, camada `EDGE_LAYER = 1`). Não usa `wireframe: true` do material. |
| Brilho da peça acesa | Existe (`setGlow`), por troca de cor e emissão do material. **Não** há bloom. |
| **Sombras projetadas** | **Não existe neste projeto.** Nenhum `castShadow`, `receiveShadow` ou `renderer.shadowMap`. Os objetos "flutuam" sem sombra no chão. |
| **Shaders próprios** | **Não existe neste projeto.** Nenhum `ShaderMaterial`, `RawShaderMaterial` ou `onBeforeCompile`. Só `MeshStandardMaterial` e `LineBasicMaterial`. |
| Pós-processamento (bloom, SSAO, FXAA) | Não existe. Não há `EffectComposer`. O antialiasing é o do contexto (`antialias: true`). |
| Névoa (`fog`), mapa de ambiente HDR externo, texturas próprias | Não existem. O ambiente é gerado em código (RoomEnvironment). |
| Compressão (Draco/KTX2/Meshopt) | Não. `GLTFLoader` simples, sem decodificadores. |

Quando o seu projeto precisar de sombra ou shader, esses são os pontos onde encaixar; o projeto de referência não os usa
de propósito, para ficar leve (3 contextos WebGL na mesma página).

## 2. Renderer, cena e câmera

`webgl.js:105-135` (`createViewer`):

```js
renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true });
renderer.setClearColor(0x000000, 0);          // transparente: a interface fica "por trás" do 3D
renderer.outputColorSpace = THREE.SRGBColorSpace;
const scene = new THREE.Scene();
scene.background = null;
const camera = new THREE.PerspectiveCamera(cfg.fov, 1, 0.1, 100);   // aspect é corrigido no resize()
camera.position.set(0, 0, cfg.dist);
camera.layers.enable(EDGE_LAYER);
```

- **FOV, near, far:** `fov` vem do layout (30 ou 32), **near = 0.1**, **far = 100**. A câmera nunca chama `lookAt`: olha
  para -Z a partir de `(0, 0, dist)`; o alvo do OrbitControls é a origem.
- **Pixel ratio:** `renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, MAX_DPR))` com `MAX_DPR = 1.5`
  (`webgl.js:18,165`). Teto para não pagar 4x os pixels num celular de densidade 3.
- **Tamanho:** `renderer.setSize(w, h, false)` (o `false` deixa o CSS mandar no tamanho do canvas) a partir de
  `container.clientWidth/Height`, num `ResizeObserver` (`webgl.js:162-180`).

### 2.1 Layouts (posição e enquadramento)

`webgl.js:21-24`:

| Layout | `tilt` | `yaw` | `fov` | `dist` | Uso |
|---|---|---|---|---|---|
| `center` | 70 | 0 | 30 | 8 | Hero do topo, cards do Câmbio e dos Pedais. |
| `background` | 62 | -24 | 32 | 10 | Fundo à direita e afastado (usado antes do hero; ainda existe no código). |

No layout `background`, `resize()` desloca o conjunto para a direita (`viewH * aspect * 0.24`) e, em tela estreita
(`w <= 760`), centraliza, abaixa (`-viewH * 0.12`) e escala `0.7`. No layout `center`, `root.scale = centerScale(aspect)`
com `centerScale(a) = a > 0 ? Math.min(1, a) : 1` (`poses.js:16-18`): num quadro mais alto que largo (celular) o aro
encolhe na proporção do quadro para não passar das bordas.

### 2.2 Grafo de cena

```
scene
 |- AmbientLight, DirectionalLight (key), DirectionalLight (rim)
 `- root  (Group)  rotation.y = deg(cfg.yaw + pose.yaw);  position/scale do layout
     `- stage (Group)  rotation.x = deg(cfg.tilt)
         `- model (Group "holder" ou o modelo em código)  rotation = (pose.x, pose.y, 0)  <- o giro do volante
```

Por que `stage.rotation.x = tilt`? Os modelos são montados "deitados" no plano XZ, com o eixo Y apontando para o piloto
(`models3d.js:1-2`). Inclinar o conjunto em 70 graus o mostra de frente para a câmera. É uma convenção do projeto: tudo,
inclusive os modelos exportados (seção 6), é reorientado para ela.

## 3. Luzes

`webgl.js:124-130`:

```js
scene.add(new THREE.AmbientLight(0xffffff, 0.35));
const key = new THREE.DirectionalLight(0xffffff, 1.9);
key.position.set(3, 4, 6);
scene.add(key);
const rim = new THREE.DirectionalLight(0xc8d0d8, 0.7);
rim.position.set(-5, 2, -4);
scene.add(rim);
```

| Luz | Tipo | Cor | Intensidade | Posição |
|---|---|---|---|---|
| Ambiente | `AmbientLight` | `0xffffff` | **0.35** | n/a |
| Principal (`key`) | `DirectionalLight` | `0xffffff` | **1.9** (wireframe) / **1.2** (renderizado) | `(3, 4, 6)` |
| Contraluz (`rim`) | `DirectionalLight` | `0xc8d0d8` (cinza-azulado frio) | **0.7** | `(-5, 2, -4)` |

- A `key` muda de intensidade com o estilo: `key.intensity = rendered ? 1.2 : 1.9` (`webgl.js:214`). No renderizado, a
  luz de ambiente (RoomEnvironment) já ilumina, então a direcional baixa.
- As duas direcionais usam o `target` padrão (origem). Elas não projetam sombra.
- A luz de contraluz fria e atrás (`z = -4`) desenha o contorno dos aros; a `key` quente-neutra à frente e acima dá o
  volume. Esse par (key forte + rim fraca e fria + ambiente baixo) é o que dá o aspecto "técnico".

## 4. Dois estilos: wireframe e renderizado

O estilo é uma preferência (`wireframe` | `rendered`) trocada em runtime por `viewer.setStyle(next)` (`webgl.js:324-329`) sem
recarregar o modelo. A decisão mora em `applyStyle()` (`webgl.js:196-216`):

```js
function applyStyle() {
  const wire = style === "wireframe";
  const exported = Boolean(current?.userData.exported);
  if (exported) {
    current.traverse((o) => {
      if (o.isLineSegments) o.visible = wire;
      else if (o.isMesh && o.userData.original) o.material = wire ? o.userData.wire : o.userData.original;
    });
  }
  const rendered = exported && !wire;
  if (rendered && !envMap) {
    const pmrem = new THREE.PMREMGenerator(renderer);
    envMap = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;
    pmrem.dispose();
  }
  scene.environment = rendered ? envMap : null;
  renderer.toneMapping = rendered ? THREE.ACESFilmicToneMapping : THREE.NoToneMapping;
  key.intensity = rendered ? 1.2 : 1.9;
  applyGlow();
}
```

| | Wireframe | Renderizado |
|---|---|---|
| Material das malhas | `MeshStandardMaterial` do hub (`o.userData.wire`) | Material PBR vindo do Fusion (`o.userData.original`) |
| Arestas | Visíveis (`LineSegments.visible = true`) | Ocultas |
| `scene.environment` | `null` | `PMREM(RoomEnvironment)` |
| `renderer.toneMapping` | `NoToneMapping` | `ACESFilmicToneMapping` (exposição padrão, 1) |
| `key.intensity` | 1.9 | 1.2 |
| Vale para modelos em código? | Sim (sempre wireframe) | Não: `rendered = exported && !wire`. |

Detalhes que importam:

- **RoomEnvironment criado uma vez, sob demanda.** `envMap` só é gerado na primeira vez que o estilo renderizado é usado
  com um modelo exportado. O `0.04` é o argumento `sigma` de `PMREMGenerator.fromScene(scene, sigma = 0, near = 0.1, far = 100)`
  (um leve desfoque do ambiente). O gerador é descartado logo (`pmrem.dispose()`); o `envMap` só é liberado no `destroy()`
  (`webgl.js:453`).
- **ACES só no renderizado.** Em wireframe o tone mapping desligado mantém os cinzas exatamente como definidos nos
  materiais (`0x161616`, `0xdcdcd8`); ligar ACES mudaria o contraste e "apagaria" a diferença de cor do brilho (a própria
  nota do `applyGlow`, seção 7).
- **Troca de material sem reconstruir.** Cada malha guarda os dois materiais em `userData` (seção 6.4); alternar o estilo é
  só atribuir `o.material`.

## 5. Materiais e arestas do wireframe

Constantes (`webgl.js:25-27`, repetidas em `models3d.js:6-7`; o comentário diz "= models3d.js" para lembrar de manter
iguais):

```js
const SOLID = { color: 0x161616, metalness: 0.5, roughness: 0.45, transparent: true, opacity: 0.6 };
const EDGE = { color: 0xdcdcd8, transparent: true, opacity: 0.85 };
const GLOW = 0x7fb487;
const EDGE_LAYER = 1;
```

- **Sólido:** `MeshStandardMaterial(SOLID)`: cinza quase preto, semimetálico, 60% opaco. Sem `wireframe: true`: o "wireframe"
  é uma malha translúcida **mais** linhas separadas nas arestas.
- **Aresta:** `LineBasicMaterial(EDGE)` cinza-claro a 85%. Largura de linha é sempre 1 px em WebGL (limitação da plataforma).

### 5.1 De onde vêm as arestas

- **Modelos em código** (`models3d.js:9-14`):

```js
function part(geometry, at = [0, 0, 0]) {
  const mesh = new THREE.Mesh(geometry, new THREE.MeshStandardMaterial(SOLID));
  mesh.add(new THREE.LineSegments(new THREE.EdgesGeometry(geometry, 25), new THREE.LineBasicMaterial(EDGE)));
  mesh.position.set(...at);
  return mesh;
}
```

  `EdgesGeometry(geometry, 25)`: só desenha arestas onde o ângulo entre faces passa de 25 graus (um cilindro liso não mostra
  linhas laterais).

- **Modelos exportados do Fusion** (`webgl.js:80-100`): o `.glb` já traz as **arestas reais do CAD** como primitivas de
  linha (modo 1, material `hub-arestas`); o hub só troca o material por um único `LineBasicMaterial(EDGE)` compartilhado.
  Se o modelo **não** tiver linhas, calcula pela malha com `EdgesGeometry(o.geometry, 25)`.

### 5.2 `EDGE_LAYER`: desenhar arestas sem pagar por elas no raycast

```js
const EDGE_LAYER = 1;   // arestas: a câmera desenha, os raios (rótulos, pick) não testam
camera.layers.enable(EDGE_LAYER);                 // webgl.js:123
if (o.isLineSegments) { o.material = edgeMat; o.layers.set(EDGE_LAYER); lines++; }   // webgl.js:84
```

O `Raycaster` do Three.js só testa objetos da camada 0 (padrão). Um modelo exportado tem dezenas de milhares de segmentos de
linha (o aro do projeto tem cerca de 40 mil); testar raios contra eles a cada quadro (para esconder rótulos atrás de
peças, para escolher o ponto com o mouse) seria inviável. Colocando as linhas na camada 1 e habilitando a camada 1 **só na
câmera**, a câmera as vê e os raios não. Um teste estático protege isso:
`tests/test_web_static.py::test_edges_are_drawn_but_not_raycast`. Nota de manutenção: no `webgl.js:28` o comentário do
`EDGE_LAYER` ficou misturado com o do `GLOW` (a frase "peça de um botão apertado" pertence a `GLOW`, na linha 27); ao
copiar, separe os dois comentários.

### 5.3 Geometrias dos modelos em código (`models3d.js`)

Úteis como material de partida quando não há CAD:

| Modelo | Peças |
|---|---|
| Volante (`wheel()`) | `TorusGeometry(1.6, 0.11, 10, 48).rotateX(PI/2)` (aro), `CylinderGeometry(0.42, 0.42, 0.26, 24)` (cubo), 3 raios `BoxGeometry(1.0, 0.08, 0.2)` em `(-0.84, 0, 0)`, `(0.84, 0, 0)`, `(0, 0, 0.84)`, 2 paddles `BoxGeometry(0.42, 0.05, 0.8)` em `x = ±0.7, y = -0.3, z = -0.35`, eixo `CylinderGeometry(0.16, 0.16, 0.7, 16)` em `(0, -0.5, 0)`. |
| Câmbio (`shifter()`) | placa `BoxGeometry(2.6, 0.08, 1.9)`, 3 canais `BoxGeometry(0.12, 0.03, 1.2)` em `x = -0.8, 0, 0.8`, travessa `BoxGeometry(1.72, 0.03, 0.12)`, grupo `knob` (haste `CylinderGeometry(0.05, 0.05, 1.1, 12)` + esfera `SphereGeometry(0.24, 16, 12)` em `y = 1.2`). |
| Pedais (`pedals()`) | base `BoxGeometry(2.6, 1.3, 0.08)`; cada pedal é um grupo `pedal-<chave>` na dobradiça `(PEDAL_X[k], 0, 1.3)` com dobradiça (cilindro), braço `BoxGeometry(0.1, 0.07, 1.6)` e sapata (acelerador mais estreita e alta). O conjunto fica num grupo interno girado `PEDAL_VIEW_DEG = -32` em Z. |

O grupo interno dos pedais existe porque o visualizador **reescreve a rotação do modelo inteiro a cada quadro**
(`current.rotation.set(...)`, `webgl.js:269`): qualquer rotação fixa do modelo precisa ficar num grupo filho.

## 6. Modelos exportados do Fusion

Fluxo: o add-in (`hub/fusion_addin/HubSimRacing/hubexport/`) lê o projeto do Fusion (só leitura) e grava na pasta de dados
`modelos/<id>/` os arquivos `completo.glb`, `leve.glb`, `perfil.json`, `origem.json` e `miniatura.png`. Escreve numa pasta
temporária e troca no fim (`exportar.py`), para o hub nunca ler um arquivo pela metade. O `.glb` usa **metros** (o Fusion
entrega cm; `glb.py` aplica `CM = 0.01`), um nó por corpo com `extras.caminho` (o caminho da peça no Fusion) e as arestas reais do CAD.
`pbr.py` converte a aparência do Fusion para `baseColor` linear + metalness + roughness (por exemplo, metal polido = 0,2 de
rugosidade, acetinado = 0,35, oxidado = 0,55, metal padrão = 0,5; opaco fosco = 0,8, "brilho" = 0,25, padrão = 0,6).

O `perfil.json` diz para onde apontam a **frente** (lado do piloto) e o **topo** do modelo (`±x`, `±y`, `±z`), o tipo
(`aro`, `cambio`, `pedais`, `outro`) e, para câmbio e pedais, o bloco `movimento` com as juntas.

### 6.1 `orientFor`: reorientar para a convenção do hub

`poses.js:20-34`. A convenção do hub é **frente em +Y** e **topo em -Z**. A função devolve a matriz 3x3 (linha a linha)
que leva o par `(frente, topo)` do modelo para essa convenção, completando o terceiro eixo pelo produto vetorial nos dois
lados:

```js
const AXIS = { "+x": [1, 0, 0], "-x": [-1, 0, 0], "+y": [0, 1, 0], "-y": [0, -1, 0], "+z": [0, 0, 1], "-z": [0, 0, -1] };
const cross = (a, b) => [a[1] * b[2] - a[2] * b[1], a[2] * b[0] - a[0] * b[2], a[0] * b[1] - a[1] * b[0]];

export function orientFor(frente, topo) {
  const f = AXIS[frente], t = AXIS[topo];
  if (!f || !t || frente[1] === topo[1]) return null;      // eixo inválido ou paralelo
  const F = [0, 1, 0], T = [0, 0, -1];
  const r = cross(f, t), R = cross(F, T);                  // terceiro eixo, nos dois lados (bases ortonormais e destras)
  const m = [];
  for (let i = 0; i < 3; i++) for (let j = 0; j < 3; j++) m.push(F[i] * f[j] + T[i] * t[j] + R[i] * r[j] || 0);
  return m;
}
```

É a soma dos produtos externos `F·fᵀ + T·tᵀ + R·rᵀ`; `|| 0` evita `-0`. Casos testados em `tests/js/poses.test.mjs:158-167`:
`orientFor("-y", "+z")` (Fusion com Z para cima) dá `[1, 0, 0, 0, -1, 0, 0, 0, -1]` (180 graus em X); `orientFor("+z", "+y")` leva
`+Z` para `+Y` e `+Y` para `-Z`; eixos paralelos ou inválidos dão `null`.

### 6.2 `fitScale` e `FIT_SIZE`

```js
export const FIT_SIZE = 3.4;          // a maior dimensão de um modelo exportado, em unidades da cena
export function fitScale(size) {
  const m = Array.isArray(size) ? Math.max(...size) : Number.NaN;
  return Number.isFinite(m) && m > 0 ? FIT_SIZE / m : 1;
}
```

A maior dimensão da caixa envolvente vira **3,4 unidades de cena**, qualquer que seja a unidade do arquivo. Medida estranha
(zero, NaN, nulo) devolve escala 1 em vez de quebrar. Como o volante em código tem diâmetro de aproximadamente 3,4
(`Torus` de raio 1,6 mais o tubo), os modelos exportados ficam com o mesmo tamanho aparente dos de código.

### 6.3 `prepareExported`: orientar, centrar, escalar, embrulhar

`webgl.js:53-102`. A ordem é a parte importante:

```js
const orient = new THREE.Group(), fit = new THREE.Group(), holder = new THREE.Group();
orient.add(gltfScene);
orient.quaternion.setFromRotationMatrix(new THREE.Matrix4().set(m[0], m[1], m[2], 0, m[3], m[4], m[5], 0, m[6], m[7], m[8], 0, 0, 0, 0, 1));
fit.add(orient);
fit.updateMatrixWorld(true);
const box = new THREE.Box3().setFromObject(orient);        // caixa JÁ orientada, ainda sem escala
if (source.centrar) {                                      // câmbio e pedais
  orient.position.sub(box.getCenter(new THREE.Vector3()));
  orient.updateMatrixWorld(true);
}
fit.scale.setScalar(fitScale(box.getSize(new THREE.Vector3()).toArray()));
holder.add(fit);
```

1. `orient` gira a cena do glTF para a convenção do hub.
2. A caixa é medida **depois** de orientar e **antes** de escalar.
3. **O centrar:** com `source.centrar` (câmbio e pedais), o centro da caixa vai para a origem; senão o modelo "cresce só
   para cima" e sai do quadro. O **aro não centraliza**, de propósito: ele gira pela origem do projeto do CAD (o centro do
   engate), e centralizar mudaria o eixo de giro. É uma decisão de domínio guardada em
   `tests/test_web_static.py::test_exported_shifter_and_pedals_are_centered_but_the_wheel_is_not`.
4. `fit` aplica a escala uniforme; `holder` é o objeto que o laço gira e que recebe `userData` (`exported`, `joints`).
5. Se o perfil não tem `frente`/`topo` válidos, lança erro e o chamador usa o volante em código (`fallback`).

### 6.4 Materiais guardados por malha

```js
for (const o of meshes) {
  o.userData.original = o.material.clone();                    // PBR do Fusion
  o.userData.originalEmissive = o.userData.original.emissive?.getHex() ?? 0;
  o.userData.originalColor = o.userData.original.color?.getHex() ?? null;
  o.userData.wire = new THREE.MeshStandardMaterial(SOLID);     // wireframe do hub
  o.userData.part = ...caminho do corpo no Fusion...
}
```

Cada malha tem a própria cópia dos dois materiais. É isso que permite acender **uma peça** sem acender as iguais (o clone
quebra o compartilhamento do material que o `GLTFLoader` faria entre malhas parecidas). O custo é memória: `disposeTree`
libera tudo (ver `03-utilities-and-pipeline/webgl-utils-and-perf.md`, seção do `disposeTree`).

## 7. Brilho da peça acesa (`setGlow`)

`webgl.js:218-233` (aplicação) e `331-339` (API):

```js
function applyGlow() {
  if (!current?.userData.exported) return;
  current.traverse((o) => {
    if (!o.isMesh || !o.userData.original) return;
    const on = glow.has(o.userData.part);
    // cor e emissão: só emissão somava verde ao amarelo e o tone mapping do renderizado quase apagava a diferença
    for (const [mat, emissive, color] of [[o.userData.wire, 0, SOLID.color],
      [o.userData.original, o.userData.originalEmissive, o.userData.originalColor]]) {
      if (mat.emissive) {
        mat.emissive.setHex(on ? GLOW : emissive);
        mat.emissiveIntensity = on ? 0.6 : 1;
      }
      if (mat.color && color !== null) mat.color.setHex(on ? GLOW : color);
    }
  });
}

function setGlow(paths) {                          // caminhos do Fusion dos botões apertados
  if (dead) return;
  const next = new Set(paths || []);
  if (next.size === glow.size && [...next].every((p) => glow.has(p))) return;   // nada mudou: não repinta
  glow = next;
  applyGlow();
  request();
}
```

- Cor de brilho: `GLOW = 0x7fb487` (o mesmo `--ok` do CSS). Aceso: `color = GLOW`, `emissive = GLOW`, `emissiveIntensity = 0.6`.
  Apagado: volta a `originalColor`/`originalEmissive` (e `SOLID.color` / emissivo 0 no wireframe).
- **Por que cor e emissão juntas?** Comentário do código: só emissão somava verde à cor original (amarelo, por exemplo)
  e o tone mapping do estilo renderizado quase apagava a diferença. Trocar a cor também garante um verde legível.
- O conjunto é comparado antes de repintar (`setGlow` a 30 Hz com o mesmo conjunto não custa nada) e só então pede um
  quadro. Caminhos que o modelo não tem são ignorados (`glow.has(...)` nunca casa).
- Aplica-se nos dois materiais (wireframe e original), então o brilho sobrevive à troca de estilo (`applyStyle` termina
  chamando `applyGlow`).
- Limitação conhecida: **não é bloom**. É só troca de cor/emissão; a "luz" não vaza para os lados.

## 8. Receita para um modelo novo

1. Gere o `.glb` em metros, com um nó por peça e `extras.caminho`; inclua as arestas do CAD como linhas (modo 1) se quiser o
   wireframe bonito. Sem arestas, o hub calcula com `EdgesGeometry(geom, 25)`.
2. Informe `frente` e `topo` no perfil; teste com `orientFor`.
3. Decida `centrar`: `true` se o objeto é livre e deve ficar no meio do quadro; `false` se ele gira por uma origem do CAD.
4. Escolha o layout (`center` na maioria dos casos) e confira o enquadramento com `dist` e `fov` do layout; não mude a
   câmera por modelo, mude a escala (`FIT_SIZE`) ou a posição do `stage`.
5. Se for acender peças, mantenha `GLOW` e a regra "cor e emissão"; se for usar sombra ou shader, é uma extensão fora do
   padrão deste projeto (ver seção 1).
