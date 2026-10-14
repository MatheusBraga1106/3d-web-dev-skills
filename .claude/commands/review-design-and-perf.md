---
description: Revisa o design (tokens, camadas, responsivo) e o desempenho 3D de uma mudança contra o blueprint, e devolve achados com evidência
argument-hint: [arquivos, branch ou "o diff atual"]
---

Revise esta mudança: $ARGUMENTS (se vazio, use o diff atual: `git diff` e `git diff --staged`).

## 1. Leia antes de revisar

Na raiz do blueprint (a pasta com `SKILL.md`):

1. `01-planning/3d-feature-workflow.md` - o **checklist de FPS e de UI (seção 3)** é o seu roteiro, item por item, e as regras de hardware H1-H15 (seção 4) se a mudança escreve em dispositivos.
2. `02-design-and-3d-scenes/ui-tokens-and-overlay.md` - tokens, camadas e `z-index`, `pointer-events`, seções empilhadas, folha, responsivo (<= 760 px).
3. `02-design-and-3d-scenes/interactive-physics-parallax.md` - epsilons, laço sob demanda, ScrollTrigger com `scrub`, cursor.
4. `03-utilities-and-pipeline/webgl-utils-and-perf.md` - `loadSeq`, `disposeTree`, contextos WebGL, raycast e rótulos, canvas 2D com `devicePixelRatio`.
5. Se houver backend: `03-utilities-and-pipeline/python-to-web-pipeline.md`, seções 4 a 7 (guard de Host/Origin, admin, canal ao vivo, limites).

Depois leia os arquivos alterados inteiros (não só o diff), o `CLAUDE.md` do projeto e os testes estáticos que protegem a arquitetura.

## 2. Revise nestas frentes, com um veredito por item (ok / problema / não se aplica)

**Design**
- Cores só por token (CSS, JS e canvas 2D); nenhum valor literal novo fora do `:root` e das constantes de material 3D.
- Texto dinâmico como texto (sem `innerHTML`, `insertAdjacentHTML`); sem recursos externos.
- Camadas: o `z-index` novo cabe na pilha (0 hero, 1 seções, 29 menu, 30 barra, 31 barra de título, 40 folha, 50 avisos, 100 cursor)? `pointer-events: none` onde algo cobre o canvas?
- Seções empilhadas: o `top` do `sticky` vem de `stickTop`? Há `ResizeObserver`? Seção mais alta que a tela funciona?
- Responsivo: breakpoint 760/761 coerente entre CSS e JS; sem overflow em 375 px; toque sem OrbitControls.
- Movimento: `scrub: true` e `ease: "none"`; `prefers-reduced-motion` cai no modo nativo; nada anima por tempo sem motivo.
- Foco visível, `inert`/`Esc` em folhas e diálogos, textos de erro em português.

**Desempenho 3D e 2D**
- Desenha só quando muda: há `request()` central? O `tick` se reagenda sem motivo? Epsilon por grandeza e comparação com o último valor **aplicado**?
- Teto de 30 quadros por segundo, pausa com `document.hidden`, retomada com `lastFrame = 0`.
- `devicePixelRatio` com teto (3D) e buffer = tamanho x densidade (2D); `canvas.width` só muda quando tamanho ou densidade mudam.
- Ciclo de vida: `destroy()` completo (`cancelAnimationFrame`, `disconnect`, `dispose`, `forceContextLoss`); métodos com `if (dead) return`; chamada no `unmount`.
- Carregamento: `loadSeq` antes do primeiro `await`, descarte do atrasado, plano B em código, `availableModels` sem guardar falha.
- Contextos WebGL: quantos agora? Reaproveita o do herói? Cada um desenha só quando muda?
- Raycast: arestas em `EDGE_LAYER`; número de rótulos; nada de picking por quadro; `disposeTree` cobre os materiais reservas?
- Buffers de séries ao vivo são circulares e de tamanho fixo; widgets de canvas redesenham por chave.

**Segurança e testes** (se tocar backend, dados ao vivo ou hardware)
- Toda rota que muda estado usa `require_admin`; Host e Origin protegidos; limites de corpo; nomes com `fullmatch`.
- Tipos estritos (`confirmed is True`); erro de plug-in vira `{"ok": false, "error": ...}` sem rastreamento.
- Regras H1-H15 respeitadas; o front só altera um rascunho; nenhum teste abre hardware, rede ou pastas reais.
- Há teste para cada regra nova (puro, estático ou de servidor)? Algum teste ficou sem ver falhar (RED)?

## 3. Como medir, em vez de supor

Se o projeto tem modo de teste, suba-o e confira: `stats().frames` parado com dado repetido por 2 s, nenhum quadro com `document.hidden`,
console limpo, sem respostas HTTP >= 400 inesperadas, largura de 375 e 1200 px. Rode a suíte inteira. Não rode testes de hardware e não envie nada a dispositivos.

## 4. Formato da resposta

1. **Resumo** em 3 linhas (a mudança está pronta? o que bloqueia?).
2. **Achados**, do mais grave ao menor, cada um com: arquivo:linha, o problema, a evidência (trecho, teste ou medida), a correção sugerida e o item do checklist a que se refere.
   Marque separadamente o que é **bloqueante**, o que é **recomendado** e o que é **ideia futura**.
3. **O que está bom** (curto), para o autor saber o que manter.
4. **O que só o dono pode conferir** (hardware real, FPS em uso real).

Não edite arquivos neste comando, a não ser que eu peça. Se um achado contradiz uma decisão registrada no ledger/spec, cite a decisão e pergunte antes de insistir.
