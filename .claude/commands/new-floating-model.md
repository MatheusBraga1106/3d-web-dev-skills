---
description: Adiciona um modelo 3D flutuante ou reativo a dados ao vivo, reaproveitando o visualizador e as regras de desempenho do blueprint
argument-hint: [nome do modelo, onde aparece e o que ele reage]
---

Adicione este modelo 3D ao projeto: $ARGUMENTS

## 1. Leia antes de gerar qualquer código

Na raiz do blueprint (a pasta com `SKILL.md`):

1. `02-design-and-3d-scenes/3d-lighting-and-materials.md` - câmera (FOV 30, near 0.1, far 100, posição (0,0,8)), luzes (ambiente 0.35, principal 1.9 em (3,4,6), contraluz 0.7
   em (-5,2,-4)), estilos wireframe e renderizado, `EDGE_LAYER`, `orientFor`/`fitScale`/`FIT_SIZE` (3.4)/centrar, `setGlow`.
2. `02-design-and-3d-scenes/interactive-physics-parallax.md` - epsilons, `rotateModel`, `jointTargets`/`setJointAngle`, o laço sob demanda e a PROPOSTA de flutuação (seção 9).
3. `03-utilities-and-pipeline/webgl-utils-and-perf.md` - carregamento com `loadSeq`, `disposeTree`, contextos WebGL, custo de rótulos e raycast.
4. `04-templates/golden-snippets.md` - seções 1 (cena), 2 (laço e pose), 7 (carregamento contra corrida) e 8 (3D sob demanda).
5. `01-planning/prompt-templates.md` - template (b), para saber quais perguntas faltam.

Depois leia, no projeto atual: o módulo do visualizador (algo como `webgl.js`), o módulo de poses puras, a página onde o modelo vai entrar e o teste estático que protege os módulos 3D.
Confira também quantos contextos WebGL já existem hoje.

## 2. Se faltar informação, pergunte (uma pergunta por vez, com recomendação)

- Fonte do modelo: `.glb` do CAD (frente, topo, unidade, tem arestas?) ou montado em código?
- Qual valor ao vivo ele segue, com que fórmula e com que epsilon? O que ele faz sem dado?
- Flutuante: amplitude, frequência e se pausa quando coberto. Reativo: qual propriedade (rotação, posição, junta, brilho).
- Em que página/card ele aparece e se pode compartilhar o contexto WebGL do herói.

## 3. Implemente (TDD, na ordem)

1. **Função pura da pose** em um módulo sem DOM e sem Three.js (ex.: `poses.js`), com testes `node --test`: caso normal, limites, `NaN`, string, `null`, valor repetido (não deve gerar mudança acima do epsilon).
2. **Montagem do modelo** no visualizador existente (não crie outro): modelo exportado via `prepareExported` (orientar -> medir -> centrar -> escalar -> materiais `wire`/`original` por malha)
   ou modelo em código no estilo de `models3d.js` (`MeshStandardMaterial` com `SOLID` + `LineSegments` com `EdgesGeometry(geom, 25)`). Arestas na camada `EDGE_LAYER`.
3. **Carregamento robusto:** `loadSeq` incrementado antes do primeiro `await`, descarte do resultado atrasado com `disposeTree`, plano B em código se o `.glb` falhar, sem erro na tela.
4. **Reação ao dado:** método do visualizador com a guarda `if (dead) return`, validação `Number.isFinite`, comparação `moved(prev, next, eps)` e `request()` só quando mudou.
5. **Flutuação (se pedida):** use a PROPOSTA do blueprint (fase acumulada, `dt` limitado a 0,1 s, aplicada no `stage`/`root` e nunca no objeto que o laço reescreve, respeitando `reduceMotion`),
   exponha `setFloat(on)` no estilo de `setIdle`, e desligue quando o modelo estiver coberto. Isso quebra "ocioso quando nada muda": declare o custo.
6. **Página:** `await import("../webgl.js")` dentro de `try`; `viewer?.destroy()` no `unmount`.

## 4. Regras que não podem quebrar

- Sem sombras e sem shaders próprios (o padrão não os usa). Se eu pedir, trate como extensão e meça o custo.
- Nenhuma cor literal no JS fora das constantes `SOLID`/`EDGE`/`GLOW`; cores de UI vêm dos tokens.
- O 3D só lê: nada de API de escrita, token, `innerHTML`, `localStorage`.
- Desenhar só quando muda, 30 quadros por segundo no máximo, pausar com a aba escondida, `devicePixelRatio` com teto de 1,5.

## 5. Verifique e relate com números

- `node --test` e a suíte do projeto verdes (cole o resumo).
- No modo de teste do projeto: `stats().frames` não cresce em 2 s com dado repetido; com `document.hidden` simulado nenhum quadro é desenhado;
  375 px e 1200 px sem overflow horizontal; console limpo; o que acontece com o `.glb` ausente (deve cair no modelo em código).
- Liste o que só eu posso conferir (hardware, FPS em uso real). Não faça push.
