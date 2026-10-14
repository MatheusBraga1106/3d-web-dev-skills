# 3d-web-dev-skills

Documentação técnica reutilizável, extraída de um projeto real (**Hub Sim Racing**: Python + FastAPI + pywebview + JS puro, com
Three.js 0.186.1, GSAP ScrollTrigger e Lenis locais). Serve como **skill do Claude Code**, como **base de conhecimento** para qualquer
sessão de IA ou pessoa, e como **ponto de partida** para projetos novos no mesmo padrão: interface web interativa com cena 3D, servida por Python e empacotada como
app de desktop.

Nada aqui é resumo genérico. Cada valor (câmera, luzes, epsilons, `z-index`, constantes de parallax, rotas) foi lido no código e vem com
o caminho e as linhas de origem (`arquivo:linhas`, commit `ff34fab`, versão 0.2.1 do projeto de referência). Tudo em **português do Brasil**.
Quando um tópico não existe no projeto de referência (sombras, shaders próprios, flutuação por senoide), o documento diz isso e mostra o que o projeto faz no lugar.

## Estrutura

```
3d-web-dev-skills/
  SKILL.md                         # skill: frontmatter + índice mestre + cola rápida (mesma estrutura em CLAUDE.md)
  CLAUDE.md                        # cópia de SKILL.md, para quando a pasta for lida como instruções do projeto
  README.md                        # este arquivo
  .claude/commands/
    plan-3d-feature.md             # /plan-3d-feature: brainstorming, spec e plano (não escreve código)
    new-floating-model.md          # /new-floating-model: modelo 3D flutuante ou reativo
    review-design-and-perf.md      # /review-design-and-perf: revisão de design e desempenho
  01-planning/
    3d-feature-workflow.md         # o fluxo real: brainstorming -> spec -> plano -> subagentes + TDD -> conferência -> push/.exe; checklists de FPS e UI; regras de hardware
    prompt-templates.md            # 4 templates: projeto novo, modelo 3D, ajuste de física/parallax, rotas e automação Python
  02-design-and-3d-scenes/
    ui-tokens-and-overlay.md       # paleta, tipografia, bordas, camadas HTML sobre canvas, hero fixo, seções empilhadas, folha, responsivo
    3d-lighting-and-materials.md   # câmera, luzes, estilos, arestas, materiais, brilho, modelos do CAD
    interactive-physics-parallax.md# seguir o ângulo, juntas, pedais, OrbitControls, Lenis, ScrollTrigger, laço sob demanda
  03-utilities-and-pipeline/
    python-to-web-pipeline.md      # FastAPI, WebSocket, plug-ins, segurança, pywebview, testes, PyInstaller, Inno Setup, release assinado
    webgl-utils-and-perf.md        # GLB, loadSeq, disposeTree, resize, DPR, contextos WebGL, raycast, canvas 2D
  04-templates/
    golden-snippets.md             # blocos de código reais (e um servidor FastAPI mínimo testado)
```

## Como usar em um projeto novo

Escolha uma das três formas. Em todas, o conteúdo é o mesmo.

### 1. Copiar a pasta para dentro do projeto

```bash
cp -r 3d-web-dev-skills ./meu-projeto/3d-web-dev-skills
mkdir -p ./meu-projeto/.claude/commands
cp ./meu-projeto/3d-web-dev-skills/.claude/commands/*.md ./meu-projeto/.claude/commands/
```

No PowerShell:

```powershell
Copy-Item -Recurse 3d-web-dev-skills .\meu-projeto\3d-web-dev-skills
New-Item -ItemType Directory -Force .\meu-projeto\.claude\commands | Out-Null
Copy-Item .\meu-projeto\3d-web-dev-skills\.claude\commands\*.md .\meu-projeto\.claude\commands\
```

Depois aponte o `CLAUDE.md` do projeto para ela com uma linha, por exemplo:
`Padrões de 3D, UI e pipeline Python: ver 3d-web-dev-skills/SKILL.md (leia antes de mexer em 3D).`
Os comandos `/plan-3d-feature`, `/new-floating-model` e `/review-design-and-perf` passam a existir no projeto.

### 2. Usar como skill em `~/.claude/skills`

```bash
mkdir -p ~/.claude/skills
cp -r 3d-web-dev-skills ~/.claude/skills/3d-interactive-web-blueprint
# comandos disponíveis em todos os projetos (opcional):
mkdir -p ~/.claude/commands
cp ~/.claude/skills/3d-interactive-web-blueprint/.claude/commands/*.md ~/.claude/commands/
```

A pasta precisa se chamar como o `name` do `SKILL.md` (`3d-interactive-web-blueprint`). O Claude carrega a skill quando o pedido combina com a
`description` (Three.js/WebGL, FastAPI + WebSocket, parallax, GLB, etc.) e segue o índice mestre para ler só os documentos necessários.

No PowerShell (Windows): `Copy-Item -Recurse 3d-web-dev-skills "$env:USERPROFILE\.claude\skills\3d-interactive-web-blueprint"`.

### 3. Como submódulo do git

Depois que você publicar este repositório (a publicação é um passo seu; este repositório foi criado só localmente):

```bash
git submodule add <URL-do-repositorio> 3d-web-dev-skills
git commit -m "docs: adiciona o blueprint 3D como submódulo"
# atualizar depois:
git submodule update --remote 3d-web-dev-skills
```

Os comandos continuam em `3d-web-dev-skills/.claude/commands/`; copie-os (ou crie links) para `.claude/commands/` do projeto, porque o Claude Code só lê esse diretório na raiz do projeto.
Quem clonar o seu projeto precisa de `git clone --recurse-submodules`.

## Um primeiro passeio (10 minutos)

1. Leia `SKILL.md` (o índice e a "cola rápida").
2. Leia `01-planning/3d-feature-workflow.md`, seções 1 e 3.
3. Abra `04-templates/golden-snippets.md`, seção 5, e rode o servidor mínimo (cabe em um arquivo; precisa de `fastapi`, `uvicorn` e `websockets`).
4. Rode `/plan-3d-feature <sua ideia>` no seu projeto e responda às perguntas.

## Convenções deste repositório

- O projeto de referência é citado como `arquivo:linhas` no commit `ff34fab`. Se você copiar um trecho, copie o comentário também.
- Não há segredos nem dados pessoais aqui: chaves de atualização, caminhos de usuário, e-mails e identificadores de perfil do projeto original foram omitidos ou generalizados (`<...>`).
- O que foi **escrito aqui** (e não copiado) está avisado: o servidor FastAPI mínimo de `golden-snippets.md` (testado), a PROPOSTA de flutuação por senoide e a tabela H1-H15 de segurança com hardware (generalização das regras do projeto original).
- Sugestões de melhoria entram como PROPOSTA ou "oportunidade", marcadas, nunca como fato do projeto de referência.
