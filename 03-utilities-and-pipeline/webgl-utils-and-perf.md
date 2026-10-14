# Utilidades de WebGL e desempenho

Carregamento de `.glb`, laço de render sob demanda, resize, `devicePixelRatio`, descarte de recursos, custo de vários
contextos WebGL, rótulos e raycast, e o canvas 2D do painel de telemetria. Todo o código citado é do projeto de referência
(`hub/web/js/webgl.js` e `hub/web/painel/`), commit `ff34fab` (versão 0.2.1).

## 1. O que existe e o que não existe neste projeto

| Tópico | Situação |
|---|---|
| `GLTFLoader` simples (`loadAsync`) | Existe. Sem Draco, KTX2 ou Meshopt. |
| Pausa de render com a aba escondida | Existe (`document.hidden`). |
| Pausa de render quando o canvas está **fora da tela** (coberto por outras seções ou rolado para fora) | **Não existe neste projeto.** Só `document.hidden` pausa. O hero continua redesenhando enquanto o ângulo muda, mesmo coberto pelas seções empilhadas. Veja a seção 8, oportunidade 1. |
| `InstancedMesh`, LOD, BVH (`three-mesh-bvh`) | Não existem. O custo do raycast é contido por outras vias (seção 7). |
| Descarte do material "reserva" de cada malha | **Não completo**: `disposeTree` descarta `o.material` (o atual); o outro material guardado em `userData` (`wire` ou `original`) só é alcançado se for o atual no momento (ver seção 5). |
| Canvas 2D nítido em tela de alta densidade | Existe (`criarCanvas`, com `devicePixelRatio`). |
| Buffer circular para séries ao vivo | Existe (`painel/buffer.js`). |

## 2. Carregar um GLB

### 2.1 Dois caminhos: modelo em código e modelo exportado

`loadModel(name, source = null)` (`webgl.js:289-308`) decide:

- Com `source.url` (modelo exportado do Fusion): `loadExported(name, source)`.
- Sem `source`: o nome precisa passar por `modelUrl(name)` (`poses.js:45-47`): só aceita `^[\w-]+\.glb$` e devolve
  `/assets/models/<nome>`. `"../x.glb"`, URL absoluta, `a/b.glb`, `x.gltf` e `x.glb?y` dão `null` (há teste). Depois, só tenta
  baixar se o nome estiver em `/assets/models/models.json`; senão (ou se o `GLTFLoader` falhar) usa o modelo montado em código
  (`build(name)`).

```js
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
```

### 2.2 `loadSeq` contra corrida

`loadSeq` é um contador de "geração". Cada carregamento pega o seu número **antes** do primeiro `await` e, depois de cada espera,
confere `seq !== loadSeq`: se outro carregamento começou (o usuário trocou o modelo), ou o visualizador foi destruído, o
resultado atrasado é **descartado e liberado** (`disposeTree(model)`) em vez de entrar na cena. Também incrementam `loadSeq`:
`clearModel()` e `destroy()`. Sem isso, um `.glb` grande que termina depois de um pequeno sobrescreveria o escolhido por último.
Detalhe: `++loadSeq` vem **antes** do atalho `name === currentName`, para um carregamento antigo de outro modelo também ser
invalidado quando se volta ao modelo atual.

### 2.3 `availableModels()`: uma busca para todas as instâncias, sem guardar falha

```js
let available = null;          // lista de models.json (promessa), lida uma vez para todas as instâncias
function availableModels() {
  if (!available) {
    available = fetch("/assets/models/models.json", { cache: "no-store" })
      .then((r) => { if (!r.ok) throw new Error(`models.json: ${r.status}`); return r.json(); })
      .then((d) => (Array.isArray(d.available) ? d.available : []))
      .catch(() => { available = null; return []; });   // hub reiniciando: tenta de novo no próximo modelo
  }
  return available;
}
```

Guarda a **promessa** (várias instâncias pedindo ao mesmo tempo compartilham uma só requisição) e, na falha, zera `available`
para a próxima tentativa (um teste protege: "a falha ao ler o models.json não fica guardada até recarregar").

### 2.4 Modelo exportado: `loadExported` e o `fallback`

```js
async function loadExported(name, source) {
  const seq = ++loadSeq;
  if (name === currentName && source.url === currentUrl) return;     // mesmo arquivo: nada a fazer
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

- `source = { url, frente, topo, movimento?, centrar?, fallback? }`. O `fallback` é o nome do modelo em código usado se o
  exportado falhar (câmbio -> `"cambio.glb"`, pedais -> `"pedais.glb"`, aro -> `"volante.glb"`).
- A URL do arquivo traz `?v=<rev>` (`modelFileUrl`, `poses.js:50-54`): quando o add-in regrava o modelo, a revisão muda, a URL
  muda, e o navegador não reaproveita o arquivo velho; e o `currentUrl` diferente força o recarregamento. O servidor também
  responde `Cache-Control: no-cache` para esses arquivos.
- A página também protege o carregamento: depois de `await viewer.loadModel(...)`, confere `viewer.stats().joints > 0` e `alive`
  antes de assumir o modo de juntas (`pages/pedals.js:75-76`).

### 2.5 Escala e pivô (`prepareExported`)

Ver `02-design-and-3d-scenes/3d-lighting-and-materials.md`, seção 6, para a matemática de `orientFor`/`fitScale`/centrar. O que
interessa para desempenho e correção:

- A caixa envolvente é calculada **uma vez**, na carga (`Box3.setFromObject`).
- O `holder` devolvido é o objeto que o laço gira (`current.rotation.set(...)`); o **pivô do giro é a origem do projeto de CAD**
  no aro (não centrado) ou o centro da caixa em câmbio e pedais (centrado).
- Se o perfil não tem `frente`/`topo` válidos, `prepareExported` lança, o `catch` cai e o `fallback` entra.

### 2.6 Arquivos "completo" e "leve"

O add-in exporta duas versões do mesmo modelo (`completo.glb`, `leve.glb`) e o perfil escolhe em `usar` (padrão `"leve"`). A
leve deixa de fora parafusos, encoders e peças pequenas (regex `_HARDWARE` em `hubexport/perfil.py`) que quase não aparecem
e pesam. No aro de referência: cerca de 280 mil triângulos no completo e cerca de 156 mil no leve. O servidor ainda recusa `.glb`
com mais de 80 MB, sem cabeçalho glTF 2.0 válido ou com `uri` externa (`models3d.inspect_glb`).

## 3. O laço rAF sob demanda e o resize

O laço está descrito com o código na seção 8 de `02-design-and-3d-scenes/interactive-physics-parallax.md`. Resumo operacional:

- Nada roda sem motivo: `request()` agenda **um** `requestAnimationFrame` se não houver um pendente; o `tick` desenha e **não** se
  reagenda (a não ser o giro idle).
- Teto de 30 quadros por segundo (`MAX_FPS`), pausa com `document.hidden`, retomada em `visibilitychange`.
- Para saber se algo "não deveria" desenhar, use `viewer.stats().frames` num teste: o contador sobe a cada `renderer.render`.

### 3.1 Resize

```js
function resize() {
  if (dead) return;
  const w = Math.max(1, container.clientWidth), h = Math.max(1, container.clientHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, MAX_DPR));
  renderer.setSize(w, h, false);
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  /* ...posição e escala do conjunto conforme o layout... */
  request();
}
const ro = new ResizeObserver(resize);
ro.observe(container);
```

- `ResizeObserver` no **container** (não em `window`): reage a mudança de layout (menu aberto, folha, orientação do celular).
- `Math.max(1, ...)` impede tamanho zero (container escondido), que geraria `aspect = NaN`.
- `setSize(w, h, false)`: o terceiro argumento `false` **não** mexe no estilo CSS do canvas; o CSS (`width: 100%; height: 100%`)
  continua mandando.
- `resize()` termina com `request()`: redimensionar limpa o framebuffer, então é preciso desenhar de novo.

### 3.2 `devicePixelRatio`

- 3D: `renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, MAX_DPR))` com **`MAX_DPR = 1.5`** (`webgl.js:18,165`). Um telefone
  com DPR 3 desenharia 9 vezes mais pixels que um monitor comum; o teto de 1,5 custa um pouco de nitidez e economiza muito.
  `setPixelRatio` é reaplicado a cada `resize()`, então arrastar a janela entre monitores de densidades diferentes funciona.
- Canvas 2D: sem teto, mas com a conta certa (seção 9).

## 4. Rótulos (HTML presos a pontos do modelo) e picking

Os rótulos ficam numa camada HTML (`.viewer-labels`, `pointer-events: none`) sobre o canvas. A cada quadro desenhado,
`placeLabels()` (`webgl.js:240-258`) percorre todos:

```js
for (const lb of labels) {
  if (!current) { lb.el.hidden = true; continue; }
  tmp.copy(lb.point);
  current.localToWorld(tmp);                                     // ponto do modelo -> mundo
  const dist = camPos.distanceTo(tmp);
  raycaster.set(camPos, dir.copy(tmp).sub(camPos).normalize());  // raio da câmera ao ponto
  const hit = meshHit();                                         // primeira malha no caminho
  lb.el.classList.toggle("behind", Boolean(hit && hit.distance < dist - 0.05));   // atrás de outra peça: apagado
  tmp.project(camera);
  const s = toScreen(tmp.x, tmp.y, tmp.z, w, h);                 // NDC -> pixels (função pura testada)
  lb.el.hidden = !s.visible;
  lb.el.style.left = `${s.x}px`;
  lb.el.style.top = `${s.y}px`;
}
```

`toScreen` (`poses.js:65-67`): `x = ((x + 1) / 2) * w`, `y = ((1 - y) / 2) * h`, visível se `z` entre -1 e 1 e `|x|,|y| <= 1.05`.
Vetores temporários (`camPos`, `tmp`, `dir`) são **reutilizados**: nenhuma alocação por quadro.

## 5. `disposeTree` e o ciclo de vida de recursos da GPU

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

(`webgl.js:43-51`.) Geometria, texturas (qualquer propriedade do material que seja `Texture`) e o material. Usado em três lugares:
`swap()` (descarta o modelo anterior), carregamento descartado por corrida (seção 2.2) e `destroy()`.

**Ponto de atenção (observação sobre o código):** `disposeTree` alcança só `o.material`, o material **atualmente atribuído**. Cada
malha exportada guarda dois materiais em `userData` (`wire` e `original`) e usa um por vez; o que não está em uso não é percorrido. Se o
usuário alternou o estilo durante a vida do modelo, o material reserva pode ter chegado a ser enviado à GPU e não é descartado
na troca de modelo. Se você copiar esse padrão, descarte também `o.userData.wire`/`o.userData.original`.

`destroy()` (`webgl.js:440-459`) é o contrato de saída de uma página:

```js
dead = true; loadSeq++;
if (raf) cancelAnimationFrame(raf); raf = 0;
ro.disconnect();
document.removeEventListener("visibilitychange", onVisible);
if (controls) { controls.removeEventListener("change", request); controls.dispose(); }
if (current) { stage.remove(current); disposeTree(current); }
envMap?.dispose();
renderer.dispose();
renderer.forceContextLoss();                 // devolve o contexto WebGL já (o navegador limita quantos existem)
canvas.remove(); labelLayer?.remove();
```

Cada método público começa com `if (dead) return;`, então uma chamada tardia de uma página já desmontada (um `await` que
resolveu depois do `unmount`) é inofensiva. Testes estáticos garantem `forceContextLoss()`, `renderer.dispose()` e
`viewer?.destroy()` no `unmount()` de cada página 3D.

## 6. Os três contextos WebGL

Na página única do projeto de referência existem **três** visualizadores ao mesmo tempo:

| Contexto | Onde | Origem |
|---|---|---|
| 1 | Volante do topo (hero) | `hero.js:29` |
| 2 | Card do Câmbio | `pages/shifter.js:261` |
| 3 | Seção dos Pedais | `pages/pedals.js:67` |

Todas as seções são montadas juntas, então os três contextos coexistem. Decisões que seguram o custo:

1. **O editor do aro não cria um quarto:** reutiliza o do hero (`ctx.hero`), com `setLabels`, `setGlow`, `setFollow`, e solta no
   `unmount` sem destruí-lo.
2. **Three.js sob demanda:** `await import("../webgl.js")` dentro do `try`; sem WebGL ou com falha, a página fica sem 3D (só
   os números ou o desenho 2D). `createViewer` devolve `null` se `new THREE.WebGLRenderer` lançar.
3. **`forceContextLoss()`** no `destroy()`: navegadores limitam os contextos ativos (na ordem de dezenas); remontar seções
   várias vezes sem liberar estouraria o limite e o navegador começaria a derrubar contextos antigos.
4. **Cada um desenha só quando algo muda**, no máximo 30/s.
5. **Sem tone mapping nem PMREM no wireframe**: o `RoomEnvironment` só é gerado se o usuário escolher o estilo renderizado.
6. **`MAX_DPR = 1.5`** vale para os três.

Medida a repetir antes de adicionar um quarto contexto: abrir o app com o jogo/aplicativo pesado em primeiro plano e olhar o FPS
(ver o checklist em `01-planning/3d-feature-workflow.md`).

## 7. Custo de rótulos e raycast

- **Raycast contra malhas é o custo dominante.** `meshHit()` faz `raycaster.intersectObject(current, true)` na árvore inteira,
  **sem BVH**. Num modelo de CAD com centenas de milhares de triângulos, isso custa por rótulo e por quadro desenhado.
- Contenções reais no projeto:
  1. **Arestas fora do raycast** (`EDGE_LAYER = 1`): ~40 mil segmentos de linha no aro ficam de fora (ver
     `3d-lighting-and-materials.md`, seção 5.2).
  2. **Poucos rótulos:** fora do modo de edição, o aro só cria rótulo para os botões **apertados agora** (`pages/rim.js:135`).
  3. **Sem rótulo, sem custo:** `placeLabels` retorna na primeira linha se a lista está vazia ou não há camada de rótulos
     (`if (!labelLayer || !labels.length) return;`); visualizadores não interativos nem criam a camada.
  4. **Só quando desenha:** `placeLabels()` roda dentro do `tick`, depois do `render`, e portanto herda o teto de 30/s e a pausa.
  5. **Versão leve do modelo** como padrão.
  6. **Picking só em eventos de ponteiro** (`pickPart` ao arrastar um rótulo), nunca por quadro.
- Se precisar de muitos rótulos: considere raycast contra uma malha proxy simplificada, `three-mesh-bvh` ou esconder rótulos por
  zona em vez de testar oclusão (nada disso existe no projeto; é direção, não código).

## 8. Oportunidades de desempenho que o projeto ainda não aproveita

1. **Pausar visualizadores fora da tela.** Um `IntersectionObserver` por `container` chamando um `setVisible(false)` que faça
   `tick` retornar cedo (como o `document.hidden`) pouparia os dois cards rolados para fora e o hero coberto pelas seções
   empilhadas. Hoje só `document.hidden` pausa.
2. **Descartar o material reserva** no `disposeTree` (seção 5).
3. **BVH** para o raycast dos rótulos em modelos de CAD grandes.

## 9. O canvas 2D do painel

### 9.1 `criarCanvas`: buffer em pixels do aparelho, desenho em pixels de CSS

`hub/web/painel/widgets/canvas.js:34-78`:

```js
export function criarCanvas(cell, env = {}) {
  const dprDe = () => (typeof env.dpr === "function" ? env.dpr() : 1);
  const canvas = document.createElement("canvas");
  canvas.className = "w-canvas";
  setStyle(canvas, "width", "100%");
  setStyle(canvas, "height", "100%");
  cell.append(canvas);
  const ctx = canvas.getContext("2d");
  const size = { w: 0, h: 0, dpr: 0 };
  let ultima = null, sujo = true, paleta = null, paletaDe = null;

  const api = {
    canvas, ctx, size,
    medir() {                                           // true se o buffer mudou (e precisa desenhar de novo)
      const w = cell.clientWidth || 0, h = cell.clientHeight || 0;
      const d = Number(dprDe()) > 0 ? Number(dprDe()) : 1;
      if (w === size.w && h === size.h && d === size.dpr) return false;     // nada mudou: não toca no canvas
      size.w = w; size.h = h; size.dpr = d;
      if (!(w > 0 && h > 0) || !ctx) return false;
      canvas.width = Math.round(w * d);                 // buffer = tamanho CSS x densidade
      canvas.height = Math.round(h * d);
      ctx.setTransform(d, 0, 0, d, 0, 0);               // daqui em diante se desenha em pixels de CSS
      sujo = true;
      return true;
    },
    desenhar(chave, fn) {                               // true se desenhou
      if (!ctx || !(size.w > 0 && size.h > 0)) return false;
      const k = `${chave}|${estiloAtual()}`;
      if (!sujo && k === ultima) return false;          // mesma chave e mesmo estilo: o canvas não é tocado
      ultima = k; sujo = false;
      ctx.clearRect(0, 0, size.w, size.h);
      fn(ctx, size, api.paleta());
      return true;
    },
    invalidar() { sujo = true; },
    paleta() { /* lê os tokens do <html> uma vez por estilo */ },
  };
  return api;
}
```

Pontos que valem copiar:

- **`canvas.width = ...` apaga o canvas e é caro**, então `medir()` só o faz quando largura, altura **ou densidade** mudaram.
- O widget desenha em coordenadas de CSS (`ctx.setTransform(d, 0, 0, d, 0, 0)`); a nitidez em tela de DPR 2 ou 3 vem do buffer
  maior. O `dpr` vem de `env.dpr()`, que o `createPanel` define como `devicePixelRatio` (ou 1) e os testes substituem.
- **A chave de redesenho** é uma string barata montada pelo widget com o que importa (valores já arredondados). Ex. do
  G-meter: ``cv.desenhar(`${versao}|${max}|${atual.texto}`, ...)``. A chave é combinada com o estilo atual
  (`data-estilo` do `<html>`), então trocar de estilo redesenha sem o widget saber.
- **Cores nunca no JS:** `paleta()` lê `--p-ink`, `--p-line`, `--tom-*` com `getComputedStyle` e **guarda por estilo**
  (`paletaDe`); `p.tom("ok")` devolve a cor do tom.
- O `ResizeObserver` do painel (um só, observando cada célula) chama `impl.resize()` -> `cv.medir()`; se mudou, redesenha.

### 9.2 Buffer circular (`painel/buffer.js`)

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

Memória fixa (`new Array(cap)` uma vez), sem `shift()` O(n). Usos reais: rastro do G-meter (`TAMANHO_RASTRO = 45`, "~1,5 s a 30
quadros por segundo") e o histórico do gráfico (`criarBuffer(janela * 65 + 10)`, com 65 amostras/s de folga para o canal de 60 Hz).

### 9.3 O gráfico que rola

`painel/widgets/grafico.js`: `update(estado)` empurra `{t, v}` no buffer (e, se o intervalo passou de `LACUNA_MS = 66`, empurra o
valor anterior um instante antes para o traço virar **degrau** em vez de rampa falsa) e desenha; `tick(t)` redesenha só quando passou
um **passo de um pixel** de tempo: `passoMs = max(50, janelaMs / larguraÚtil)`. Ou seja, mesmo sem dado novo o gráfico rola
suavemente, mas nunca mais que 1 redesenho por pixel de deslocamento.

### 9.4 O laço do painel (`panel.js`)

Um `requestAnimationFrame` só, e três regras que impedem o atraso de se acumular num celular lento:

1. `push()` só guarda o quadro mais novo; o que não foi desenhado é descartado e contado (`descartados`).
2. Cada `tick` desenha **no máximo um** quadro e só chama `update()` dos widgets cujos **campos usados** mudaram
   (`sameValue` sobre `inst.campos`).
3. Sem quadro por `staleMs = 3000`, o estado vira `null` e todos os widgets mostram "—" (`setSignal(false)`).

A pausa com a aba escondida no painel vem do próprio navegador (o rAF não dispara) e, para manter a tela acesa, o módulo
`tela_acesa.js` usa a Wake Lock API e libera/readquire em `visibilitychange`.

Há ainda uma terceira família de canvas 2D (`pages/pedal_cal.js`, gráfico da calibração) com o mesmo princípio feito à mão:
`fit()` compara `{w, h, dpr}` e só então muda `canvas.width/height`; `scheduleDraw()` agenda um único rAF (`if (raf || !alive) return`).
