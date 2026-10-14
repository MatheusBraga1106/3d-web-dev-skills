---
name: 3d-interactive-web-blueprint
description: Blueprint para construir ou revisar uma interface web interativa com cena 3D (Three.js/WebGL) servida por Python (FastAPI + uvicorn + WebSocket) e empacotada como app de desktop (pywebview, PyInstaller, Inno Setup). Use quando o pedido envolver Three.js ou WebGL em JS puro sem bundler, herói 3D fixo com seções empilhadas por cima, parallax ligado à rolagem (GSAP ScrollTrigger, Lenis), tokens CSS e camadas HTML sobre canvas, carregar modelos .glb exportados de CAD (orientação, escala, centrar, juntas móveis, peças que acendem), fazer um modelo seguir um valor medido ao vivo (ângulo, pedais), desenhar só quando muda e pausar com a aba escondida, canvas 2D com devicePixelRatio e buffer circular, canal ao vivo por WebSocket (full + diff), plug-ins de dispositivos, segurança local (Host, Origin, admin por token, pareamento), testes com pytest e node --test, ou um fluxo de trabalho spec -> plano -> subagentes com TDD e regras de segurança para hardware. Contém valores reais (FOV, luzes, epsilons, z-index, constantes de parallax), trechos de código copiados de um projeto em produção e o que o projeto NÃO tem (sombras, shaders próprios, flutuação por senoide), para não inventar.
---

# 3D interactive web blueprint

Documentação técnica extraída de um projeto real (**Hub Sim Racing**: Python + FastAPI + pywebview + JS puro, com Three.js 0.186.1, GSAP
ScrollTrigger e Lenis copiados localmente). Tudo aqui tem valor exato e origem (`arquivo:linhas` no commit `ff34fab`, versão 0.2.1 do
projeto de referência). **Textos em português do Brasil.** Quando um tópico pedido não existe no projeto, o documento diz isso e mostra o que o
projeto faz no lugar (ou marca como PROPOSTA).

## Como usar (leia nesta ordem)

1. Descobriu o que precisa? Use o **índice mestre** abaixo.
2. Vai começar uma feature 3D? Rode `/plan-3d-feature <descrição>` (ou leia `01-planning/3d-feature-workflow.md` e use o template certo de `01-planning/prompt-templates.md`).
3. Vai criar um modelo flutuante/reativo? `/new-floating-model <descrição>`.
4. Vai revisar? `/review-design-and-perf <arquivos ou o diff>`.
5. Antes de escrever código, **leia os documentos citados** pelo comando; não copie constantes de memória.

## Índice mestre

| Quero... | Leia | O que tem |
|---|---|---|
| Entender o processo (brainstorming, spec, plano, subagentes, TDD, ledger, retomada, push e .exe) | `01-planning/3d-feature-workflow.md` | Fluxo real com artefatos, **checklist de FPS e UI**, regras H1-H15 de segurança com hardware |
| Prompts prontos | `01-planning/prompt-templates.md` | (a) projeto novo, (b) modelo 3D novo, (c) ajuste fino de física/parallax, (d) rotas e automação em Python |
| Paleta, tipografia, bordas, marcas em L, blur, camadas HTML sobre canvas, hero fixo, seções empilhadas (`--cover`), folha, responsivo, fundo entre painéis | `02-design-and-3d-scenes/ui-tokens-and-overlay.md` | Todos os tokens com valores, `z-index`, `pointer-events`, `sectionTone` + `color-mix` |
| Câmera, luzes, RoomEnvironment, ACES, wireframe com arestas, materiais, `setGlow`, modelos do CAD (`orientFor`, `fitScale`, centrar) | `02-design-and-3d-scenes/3d-lighting-and-materials.md` | FOV/near/far, posições e intensidades, `EDGE_LAYER`, o que não existe (sombras, shaders) |
| Volante seguindo o ângulo, epsilons, juntas, pedais, OrbitControls, Lenis, ScrollTrigger, cursor, laço sob demanda, flutuação | `02-design-and-3d-scenes/interactive-physics-parallax.md` | Fórmulas, constantes, código real, **proposta de senoide** marcada como tal |
| FastAPI, uvicorn, rotas, WebSocket, plug-ins, segurança, pywebview, bandeja, testes, PyInstaller, Inno Setup, release assinado, árvore de pastas | `03-utilities-and-pipeline/python-to-web-pipeline.md` | Rotas reais, `full`+`diff`, guard, janela, `hub.spec`, atualização assinada |
| Carregar GLB, `loadSeq`, `disposeTree`, resize, DPR, 3 contextos WebGL, custo de raycast, canvas 2D, buffer circular | `03-utilities-and-pipeline/webgl-utils-and-perf.md` | Padrões de desempenho e oportunidades ainda não aproveitadas |
| Copiar código pronto | `04-templates/golden-snippets.md` | Blocos reais comentados: cena, laço, seções empilhadas, HTML/CSS, FastAPI mínimo testado, widget de canvas |

Comandos (`.claude/commands/`): `plan-3d-feature.md`, `new-floating-model.md`, `review-design-and-perf.md`.

## Cola rápida (confira nos documentos antes de usar)

**Cena** (`webgl.js:21-24,105-135`): `PerspectiveCamera` `fov 30`, `near 0.1`, `far 100`, posição `(0, 0, 8)` no layout `center`
(`tilt 70`, `yaw 0`); layout `background`: `fov 32`, `dist 10`, `tilt 62`, `yaw -24`. Renderer `alpha: true`, `setClearColor(0x000000, 0)`,
`scene.background = null`, `MAX_DPR 1.5`.

**Luzes:** `AmbientLight 0xffffff 0.35`; `DirectionalLight 0xffffff 1.9` em `(3, 4, 6)` (1.2 no renderizado); contraluz `0xc8d0d8 0.7` em `(-5, 2, -4)`.
Estilo renderizado: `RoomEnvironment` via PMREM (`sigma 0.04`) + `ACESFilmicToneMapping`; wireframe: `NoToneMapping`.

**Materiais:** sólido `0x161616`, `metalness 0.5`, `roughness 0.45`, `opacity 0.6`; aresta `0xdcdcd8` a `0.85`; brilho `GLOW 0x7fb487` (cor + emissão a `0.6`);
`EDGE_LAYER = 1`; `EdgesGeometry(geom, 25)`. `FIT_SIZE = 3.4`.

**Desempenho:** `MAX_FPS 30`; epsilons: ângulo do volante `0.2°`, manopla `0.005`, pedal `0.005`, junta `0.001745 rad`; pausa com `document.hidden`; `loadSeq` contra corrida.

**Movimento:** Lenis `lerp 0.12`, `autoRaf: false` (acionado pelo `gsap.ticker`, `lagSmoothing(0)`); parallax do herói `yPercent 14`, `scale 0.86`, `opacity 0.4`, `scrub: true`,
`ease: "none"`; entrada das seções `y 48 -> 0`, `opacity 0.25 -> 1`; `--cover 0 -> 1` (até 60% de preto); cursor `quickTo 0.2 s power3`.

**Camadas:** hero `sticky` `z 0`; seções `z 1`; menu `29`; barra `30`; barra de título `31`; folha `40`; avisos `50`; cursor `100`. Breakpoint: `<= 760 px` (mobile) / `>= 761 px` (empilhar, parallax).

**Cores:** fundo `#070707`, tinta `#E8E8E5 / #A9A9A5 / #6F6F6B`, linhas `rgba(255,255,255,.10)`, acento `#EDEDEA`, ok `#7FB487`, warn `#E0B45E`, err `#F08A80`, `--sec-mid #1c1c1c`.

## O que NÃO existe no projeto de referência (não invente)

- **Sombras projetadas** (CSS ou 3D) e **shaders próprios**: ausentes. Nem pós-processamento, névoa ou HDR externo.
- **Glassmorphism clássico**: só fundo translúcido + `blur(6px)` + borda de 1 px, raio 0, sem sombra.
- **Flutuação por `Math.sin`** de objetos: ausente. Há uma PROPOSTA de encaixe em `interactive-physics-parallax.md`, seção 9.
- **Tema claro/escuro real**: o app é sempre escuro (o atributo `data-theme` existe, mas nada varia com ele).
- **Giro automático (idle)**: o código existe (`setIdle`, 10 graus/s), mas nenhuma página o liga hoje.
- **Pausa de render fora da tela**: só a aba escondida pausa; veja `webgl-utils-and-perf.md`, seção 8.
- **Física** (massa, mola): ausente; "amortecimento" é só o do Lenis e o do cursor.

## Regras que valem em tudo

1. Português do Brasil nos textos e comentários; identificadores em inglês (convenção do projeto de referência; troque se a sua for outra).
2. Sem framework, sem bundler, sem CDN: bibliotecas em `vendor/` com versão e SHA-256 conferidos por teste.
3. Toda cor vem de token; texto dinâmico sempre como texto; o 3D só lê.
4. Desenhar só quando muda; 30 quadros por segundo no máximo; pausar com a aba escondida.
5. Escrita só por admin (token + cliente local), por clique explícito; testes nunca tocam hardware, rede ou pastas reais.
6. Spec e plano antes do código; um teste que falha antes de cada implementação; commits pequenos; **push só com confirmação**.
7. Mostrar números (quadros, milissegundos, linhas de teste) no relatório, não adjetivos.
