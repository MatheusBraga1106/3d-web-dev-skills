---
description: Planeja uma feature 3D (brainstorming, spec e plano) seguindo o blueprint, sem escrever código de produção
argument-hint: [descrição da feature]
---

Você vai PLANEJAR (não implementar) esta feature 3D: $ARGUMENTS

## 1. Leia antes de qualquer coisa

Localize a raiz do blueprint (a pasta que contém `SKILL.md`; em um projeto costuma ser `3d-web-dev-skills/` ou
`.claude/skills/3d-interactive-web-blueprint/`) e leia, nesta ordem:

1. `SKILL.md` (índice mestre e a lista do que NÃO existe no projeto de referência).
2. `01-planning/3d-feature-workflow.md` (o fluxo: brainstorming, spec, plano, execução, conferência; o checklist de FPS e UI; as regras H1-H15 de segurança com hardware).
3. `01-planning/prompt-templates.md` (o template que mais se parece com a tarefa).
4. Conforme o assunto: `02-design-and-3d-scenes/interactive-physics-parallax.md`, `02-design-and-3d-scenes/3d-lighting-and-materials.md`,
   `02-design-and-3d-scenes/ui-tokens-and-overlay.md`, `03-utilities-and-pipeline/webgl-utils-and-perf.md`,
   `03-utilities-and-pipeline/python-to-web-pipeline.md`.

Depois leia o PROJETO ATUAL: o `CLAUDE.md` da raiz, a estrutura de `docs/` (se existir `docs/superpowers/`, leia a spec e o plano mais
recentes e o fim do ledger) e os arquivos que a feature vai tocar. Não assuma que o projeto atual segue o de referência: confira.

## 2. Faça

1. **Brainstorming:** pergunte UMA coisa por vez, sempre com a sua recomendação, até fechar: objetivo, dados de entrada (de onde vem o valor ao vivo),
   comportamento 3D, o que acontece sem WebGL / sem dado / com dado estranho, impacto em desempenho (quantos contextos WebGL, quadros por segundo) e o que
   precisa de hardware ou do dono para conferir. Registre as escolhas do dono e marque as suas decisões com **(decisão minha)**.
2. **Spec** em `docs/superpowers/specs/AAAA-MM-DD-<tema>-design.md` (ou o equivalente do projeto), com as seções: Objetivo, Decisões, Estrutura, Desempenho
   (números: epsilon, teto de FPS, pausa com aba escondida), Segurança e falhas (tabela falha -> resultado), Testes (puros, estáticos, conferência no modo de teste), Fora do escopo.
   Mostre e PARE para o meu OK.
3. **Plano** (só depois do OK) em `docs/superpowers/plans/AAAA-MM-DD-<tema>.md`: Objetivo, Arquitetura, Restrições globais, **Foco da revisão** (5 perguntas de risco), tarefas
   com Arquivos, Interfaces (produz/consome), passos `- [ ]`, o código dos testes e a mensagem de commit. Uma tarefa por commit.
4. Faça o **pre-flight de interfaces**: uma tabela "quem produz x quem consome" entre as tarefas, com `consistent` ou `CONFLICT`. Corrija conflitos no plano antes de entregá-lo.

## 3. Regras

- Não crie nem edite código de produção neste comando. Só documentos em `docs/`.
- Use as constantes e as fórmulas reais do blueprint (epsilons, FOV, luzes, `z-index`) como ponto de partida e diga onde você se afasta delas e por quê.
- Se um tópico não existe no projeto de referência (sombras, shaders próprios, flutuação por senoide), diga isso explicitamente e proponha, marcando como PROPOSTA.
- Para hardware, aplique as regras H1-H15. Nunca planeje um teste que abra hardware real nem uma escrita automática.
- Cada tarefa do plano termina com a conferência do checklist de FPS e UI aplicável, com a evidência esperada (teste, número ou print).

## 4. Entregue

Um resumo curto: caminhos da spec e do plano, as decisões em aberto que dependem de mim, e a lista do que precisa ser conferido por mim fora do ambiente de teste.
