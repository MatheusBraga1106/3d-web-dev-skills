# Fluxo de trabalho para uma feature 3D (a metodologia real do projeto de referência)

Este é o processo que o Hub Sim Racing seguiu de ponta a ponta para construir o fundo 3D, o volante do topo (cockpit), os
modelos do Fusion, as juntas móveis, o painel de telemetria e a distribuição. Não é um processo idealizado: cada passo
tem um artefato real no repositório de origem (`docs/superpowers/`), citado aqui. Onde o processo teve atalhos ou
mudou no meio do caminho, isso está dito.

## 1. Visão em uma página

```
 ideia do dono
     |
 1. BRAINSTORMING   (perguntas, 2-3 abordagens, escolhas do dono anotadas)           skill: superpowers:brainstorming
     |
 2. SPEC            docs/superpowers/specs/AAAA-MM-DD-<tema>-design.md                skill: (escrita direta)
     |
 3. PLANO           docs/superpowers/plans/AAAA-MM-DD-<tema>.md                       skill: superpowers:writing-plans
     |
 4. EXECUÇÃO        subagentes por tarefa (SDD) ou inline; TDD em cada tarefa         skills: subagent-driven-development, executing-plans, test-driven-development
     |  \__ ledger: docs/superpowers/sdd-estado/progress.md (decisões, correções, pendências)
     |
 5. CONFERÊNCIA     suíte inteira + hub de teste (--modo-teste) + painel de preview + prints + checklist de FPS/UI
     |
 6. ENTREGA         commit local sempre; push só com OK; .exe de teste; release assinado (ensaio antes)
     |
 7. RETOMADA        docs/superpowers/RETOMAR-*.md + etapas/*.md (para a próxima sessão e para o dono)
```

A decisão de produto fica com o dono; o assistente planeja, conferencia e revisa; a implementação em lote vai para
subagentes (modelo intermediário). Quem planeja nunca "pula" a spec.

## 2. Passo a passo com os artefatos

### 2.1 Brainstorming

Antes de qualquer arquivo: entender a intenção, listar requisitos e escolher entre abordagens. No projeto, o brainstorming
virou a **primeira seção da spec**, com as escolhas do dono em lista curta e as decisões do assistente marcadas.
Exemplo real (spec do cockpit): uma tabela "O que faz sentido da referência, e como" (Referência -> No hub), a lista "O que o
dono escolheu, nesta ordem" (escuro com um tom por seção, página única, barra fina com hambúrguer, cursor em ponto) e um aviso:
"Demais decisões: minhas, por autorização dele. Estão marcadas com **(decisão minha)**".

Regras práticas que o projeto adotou:

- **Mostrar antes de editar** quando a decisão depende de conhecimento só do dono (por exemplo, o formato de um arquivo de
  controles de um jogo). Faça a pergunta com uma recomendação, não em aberto.
- **O que a referência traz e o que não cabe** vira tabela explícita (ex.: "Fonte Inter: Não. Seria baixada da internet.").
- Se o dono não decide, o assistente decide e **escreve que decidiu**.

### 2.2 Spec

Arquivo `docs/superpowers/specs/AAAA-MM-DD-<tema>-design.md`. Estrutura que se repete (ex.: `2026-10-12-motor-3d-design.md`):

1. Objetivo. 2. Decisões. 3. Estrutura (arquivos, módulos, contratos, formatos de dados). 4. **Desempenho**. 5. **Segurança e falhas**
(com tabela "falha -> resultado"). 6. **Testes** (o que cada teste prova e a conferência visual). 7. Fora do escopo.

O que torna uma spec 3D boa (de specs reais):

- O contrato de desempenho é número: "desenha só quando o ângulo muda em mais de 0,2° ou a manopla anda mais de 0,005", "no
  máximo 30 quadros por segundo", "`document.hidden` pausa tudo", "um canvas e um renderizador durante toda a vida da página".
- A tabela de falhas cobre o **plano B de cada dependência**: WebGL indisponível (sem canvas, o app funciona), Three.js não carregou
  (sem 3D), `.glb` 404 (modelo em código), `.glb` corrompido (modelo em código e aviso no console), dados estranhos (mantém a última pose).
- A seção de segurança diz que o 3D **só lê** (nenhum módulo 3D importa a API de escrita, usa o token ou `innerHTML`).
- Fica registrado o que **depende do dono** ("conferir com o dono": girar o volante real, jogar com o app aberto e ver o FPS).

### 2.3 Plano

Arquivo `docs/superpowers/plans/AAAA-MM-DD-<tema>.md`, escrito como se o executor não conhecesse o projeto
(skill `superpowers:writing-plans`). Estrutura real (ex.: `2026-10-12-motor-3d.md`, `2026-10-13-movimento-juntas.md`):

- Cabeçalho: objetivo, arquitetura (3 bullets), stack, caminho da spec, quem executa.
- **Restrições globais** (lista curta e dura): versões exatas de bibliotecas, "nada de CDN", "o 3D só lê", o `z-index`/estrutura do canvas, teto de
  FPS, formato dos nomes de arquivo, comentários em português e identificadores em inglês, mensagem de commit, o que **não** entra nos commits.
- **Foco da revisão** (cinco perguntas de risco). Exemplo real: "troca rápida de página enquanto um modelo carrega: o modelo antigo não pode
  aparecer na página nova"; "janela escondida: nenhum quadro, mesmo com o canal ao vivo mexendo"; "chiado do ângulo parado: não pode
  redesenhar sem parar"; "WebGL indisponível: o app funciona igual".
- **Tarefas** (`### Task N`) com `Arquivos`, `Interfaces` (o que produz e o que consome), passos com caixa de seleção `- [ ]`, **o código dos testes
  já escrito no plano**, a regra "veja o teste falhar" e a mensagem de commit pronta.

Cada item do "Foco da revisão" corresponde a um item de conferência no último passo do plano (ex.: "Task 5, Step 3, item d").

### 2.4 Execução: subagentes, TDD, ledger

Dois modos foram usados:

- **Subagent-Driven Development** (plano das 20 tarefas da etapa 1): um subagente implementador por tarefa, um revisor depois, correções em rodadas
  ("fix round k/5"), uma **revisão final do branch inteiro** com modelo de maior capacidade e esforço alto.
- **Inline** (`superpowers:executing-plans`) nas etapas seguintes, e, quando o dono pediu, **subagente intermediário para a construção** depois de planejado
  (planejar e conferir ficam com o assistente principal).

Regras do implementador (arquivo `docs/superpowers/sdd-estado/implementer-common.md`), vale copiar:

1. Ler o briefing e as restrições globais; se algo contradiz as restrições, **parar** (`NEEDS_CONTEXT`) em vez de adivinhar.
2. **TDD:** escrever o teste, rodar e ver falhar (RED), implementar, ver passar (GREEN); rodar a suíte inteira uma vez antes do commit; saída "limpa" (sem warnings).
3. Commit com mensagem na língua do projeto e o trailer de coautoria; **nunca dar push**, nunca alterar commits de outras tarefas.
4. Auto-revisão do diff (completo? YAGNI? nomes? os testes verificam comportamento real?).
5. **Relatório** em arquivo (o que fez, evidência RED/GREEN, arquivos, desvios do briefing com o motivo) e resposta curta com `Status` (`DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT`).
6. Não despachar sub-subagentes; não abrir janelas nem simular teclado/mouse no computador do dono.

O **ledger** (`docs/superpowers/sdd-estado/progress.md`) é o diário de bordo:

- *Setup rulings:* decisões de processo (trabalhar no branch, política de modelos de revisor).
- *Pre-flight scan:* tabela "quem produz x quem consome" por interface entre tarefas, com o veredito `consistent` ou **`CONFLICT`** (um conflito real foi achado aqui, antes de escrever
  código: um `merge` que não aceitava `None` que a UI já enviava).
- Uma linha por evento: `Task N: Ruling: ... — cost if wrong: ...`, `minor (deferred): ...`, `fix round 1/5 (3 addressed, 0 open ...)`, `complete (commits a..b, review clean)`.
- `NEXT`, `CARRY` (itens para a tarefa seguinte), e as conferências locais com data.
- Cada decisão discutível tem a forma **"Ruling: <o que decidi> — <por que> — cost if wrong: <custo>"**. Isso permite ao dono revisar depois sem reler o código.

A conferência visual na nuvem/Linux usava um *harness* (`harness-nuvem/`): servidor com stubs de Windows + um script Playwright que tira prints (claro/escuro, 1200 e 375 px) e
relata erros de console, respostas HTTP >= 400 e overflow horizontal.

### 2.5 Conferência no hub de teste

O **modo de teste** é a peça que torna o 3D verificável sem hardware:

```
.venv\Scripts\python -m hub --modo-teste --sem-janela --porta-web 8771
# depois abrir http://127.0.0.1:8771/#admin=<token>     (fixe o token com a variável HUB_ADMIN_TOKEN antes de subir)
```

O que o modo de teste faz (e por isso é seguro): dispositivo principal **simulado** (nenhum HID aberto), teclas **nunca** enviadas
(`NullKeyboard`), pasta de dados própria que ignora a variável da configuração real, sem varredura de USB, sem rede de atualização, instalação do add-in só num diretório de teste.
Dispositivos auxiliares são simulados por scripts à parte (por exemplo, o câmbio por pacotes UDP numa porta de teste; pedais com valores sintéticos).

Conferência no painel de preview do assistente (navegador embutido, com uma entrada em `.claude/launch.json` apontando para o comando do hub de teste e a porta):

1. Abrir a URL do hub de teste **com** o fragmento `#admin=<token>` (admin) e **sem** ele (somente leitura, como o celular).
2. Largura de **1200** e **375 px** (`resize_window` presets desktop/mobile): sem overflow horizontal.
3. Console sem erros; sem respostas >= 400 que não sejam esperadas.
4. Para 3D: `viewer.stats().frames` **não cresce** com a página parada por 2 s; simular `document.hidden` e ver que não desenha; bloquear
   `/vendor/three/three.module.js` e ver que o app funciona sem o 3D; trocar de página rápido enquanto um `.glb` carrega.
5. Comparar com os **prints de referência** (`docs/superpowers/telas/` e `docs/prints/`).
6. Anotar o resultado no ledger com a data e o que ficou **pendente com o dono** (hardware real, jogo aberto, FPS no jogo).

### 2.6 Entrega: commit, push, .exe

- **Commit local sempre** (pequeno, mensagem no idioma do projeto, trailer de coautoria). **Push só com confirmação** do dono, exceto no gatilho combinado (ver 2.7).
- `.exe` de teste: `pyinstaller --noconfirm hub.spec`; se o app do dono estiver aberto (a pasta `dist` fica travada), gerar com `--distpath` e `--workpath` alternativos e avisar.
  Conferir o `.exe` em modo de teste (servidor sem janela, MIME dos `.js`/`.css`, `/ws/live`, log vazio) e depois com janela.
- Lançamento: `tools/release.py X.Y.Z --ensaio` (faz tudo, não publica nada) e depois sem `--ensaio`; só publica depois de um `s` na pergunta. Detalhes em
  `03-utilities-and-pipeline/python-to-web-pipeline.md`, seção 12.
- Qualquer ação externa (instalar um programa de terceiros, criar repositório, baixar uma biblioteca) **só com o OK do dono, com nome, origem e tamanho**. O download do Three.js, do GSAP e do Lenis seguiu isso: versão exata do registro
  oficial, conferida pelo `integrity`, SHA-256 gravado em `VERSION.txt`, e um teste que recalcula os hashes.

### 2.7 Retomada (sessões longas e limites de uso)

Para um trabalho que atravessa várias sessões, o projeto mantém:

- `docs/superpowers/RETOMAR-LOCAL.md`: onde está cada coisa, o que foi feito, as **pendências em ordem** (com riscado nas resolvidas e a data), e um **prompt pronto** para colar na sessão nova.
- `docs/superpowers/RETOMAR-AUTO.md`: escrito quando o limite de uso chegou a ~96%; descreve o **pedido do dono (resumo fiel)**, as **regras que não mudam**, e as etapas planejadas; uma tarefa agendada abre a sessão nova depois do reset.
- `docs/superpowers/etapas/NN-<tema>.md` + `etapas/README.md`: um documento por etapa, no formato "O que mudou / Como trocar / Os testes / Pendência", e um resumo em tabela para o dono com **"o que você precisa fazer ou conferir"**.
- Gatilhos combinados com o dono (no projeto de origem): em ~80% do limite de uso passar a sessão para a nuvem; em ~90%, fazer o push, gerar o `.exe` de teste e escrever o markdown de retomada, e depois seguir até acabar.

## 3. Checklist de FPS e de UI (para toda feature 3D)

Marque cada item **com evidência** (teste, número ou print). Os itens vêm de testes e revisões reais.

### 3.1 FPS e recursos

- [ ] **Desenha só quando muda.** Há um `request()` central; o `tick` não se reagenda (exceto uma animação contínua explícita). Prova: `stats().frames` parado por 2 s com dado repetido.
- [ ] **Epsilon por grandeza** (ângulo 0,2°, fração 0,005, junta 0,1°). Compara com o último valor **aplicado**.
- [ ] **Teto de quadros** (30/s) com folga de 2 ms para o jitter do `requestAnimationFrame`.
- [ ] **Pausa com a aba escondida** (`document.hidden`) e retomada no `visibilitychange`, zerando o relógio do último quadro.
- [ ] **`devicePixelRatio` com teto** (1,5 no 3D) e canvas 2D com buffer = tamanho x densidade.
- [ ] **Um canvas por uso e liberação ao sair** (`destroy`: `cancelAnimationFrame`, `ResizeObserver.disconnect`, `dispose` de geometrias/materiais/texturas, `renderer.dispose()`, `forceContextLoss()`).
- [ ] **Contagem de contextos WebGL** conhecida e justificada (3 no projeto de referência); reaproveitar o do hero em vez de criar mais um.
- [ ] **Carregamento contra corrida** (`loadSeq`) e plano B visual quando o `.glb` falha.
- [ ] **Raycast barato:** arestas em outra camada, poucos rótulos, nada de picking por quadro.
- [ ] **Three.js sob demanda** (`await import(...)` em `try`); sem WebGL a página continua útil.
- [ ] **FPS conferido com o app em uso real** (por exemplo, com o jogo aberto), e anotado como pendência do dono se não foi possível.
- [ ] Animação ligada à rolagem usa `scrub: true` e `ease: "none"`; nada anima por tempo sem motivo; `prefers-reduced-motion` cai no modo nativo.

### 3.2 UI

- [ ] **Tokens, nunca cor literal** (CSS, JS e canvas 2D); toda cor nova ganha token. Teste estático nos arquivos de widget.
- [ ] **Sem `innerHTML` com dados** (texto sempre como texto). Teste estático proíbe `innerHTML` nos módulos de UI e 3D; exceção única e comentada para SVG fixo do próprio código.
- [ ] **Nenhum recurso externo** (fonte, CDN, imagem). Teste estático; `vendor/` com hashes.
- [ ] **Travas `busy`** em todo botão que envia algo (clique duplo não duplica a ação).
- [ ] **Controles só alteram um rascunho;** só "Aplicar" envia (teste: o handler de `input` do controle nunca chama a API).
- [ ] **Somente leitura no celular:** classe `admin-only` escondida em `body.readonly`.
- [ ] **Responsivo em 375 px** sem overflow horizontal; breakpoint único (760/761) repetido de forma coerente em CSS e JS.
- [ ] **Foco por teclado visível** (`:focus-visible`); camada de fundo `inert` quando uma folha/diálogo abre; `Esc` fecha.
- [ ] **Estados de erro em português** e sem rastreamento na tela.
- [ ] **O 3D só lê:** nenhum módulo 3D importa a API de escrita, o token ou armazenamento.
- [ ] **Console limpo** e **nenhuma resposta HTTP >= 400** inesperada no hub de teste.

### 3.3 Testes mínimos de uma feature 3D

1. Funções puras em módulo sem DOM e sem Three.js, testadas com `node --test` (modelo -> pose, fórmulas, epsilons, escalas).
2. Teste estático de texto para as regras de arquitetura (o que não pode aparecer no código, o que precisa aparecer).
3. Para módulos com Three.js: gancho de resolução do `import "three"` para a cópia local (ver `python-to-web-pipeline.md`, seção 9.1).
4. Conferência visual no hub de teste, com prints antes/depois.

## 4. Regras de segurança com hardware (versão genérica)

O projeto de referência controla uma base de volante *direct drive* de 10 Nm, cujo erro de software pode machucar uma pessoa. As
regras dele (S1 a S10) foram generalizadas abaixo para qualquer projeto que escreva em hardware que possa causar dano físico.

| # | Regra | Como aplicar |
|---|---|---|
| H1 | **Comandos perigosos não existem no código.** Nenhuma função monta força direta, reinício, atualização de firmware, escrita de pinos ou de hardware. | Se não precisa, não escreva. Uma lista branca de **o que pode ser escrito**, nunca uma lista de proibidos. Exceções entram na spec, com data, e com limites e testes próprios. |
| H2 | **Escrita só por clique explícito do usuário no computador principal.** Nunca automática, nunca ao iniciar, nunca enquanto um controle se move, nunca por aparelho remoto. | Rotas de escrita exigem admin (token + cliente local); o front só altera um **rascunho** até o botão "Aplicar". |
| H3 | **Lista branca e limites no servidor.** Cada campo com tipo e faixa validados no backend, independentemente da interface. | O front pode errar; o servidor recusa. |
| H4 | **Mudanças que aproximam limites mecânicos** só com o sistema numa posição segura (folga em graus). | O estado usado na decisão não pode ser velho: reconferir a idade da leitura logo antes de escrever. |
| H5 | **Aumentos grandes de força pedem confirmação, e a percepção é multiplicativa.** Confirmar quando sobe mais de X pontos **ou** quando sobe pouco em pontos mas dobra em proporção (1% -> 21% é "só" 20 pontos, mas 21x mais força). | Mostrar cada mudança com o multiplicador e o aviso "pode ficar mais forte de repente". |
| H6 | **Reler e comparar depois de escrever.** Se diferir, mostrar erro e oferecer restaurar. | Escrita sem leitura de volta é aposta. |
| H7 | **Aplicar não é salvar; desfazer devolve o que havia.** Gravar na memória permanente do aparelho é uma ação à parte, com confirmação. | Cancelar, fechar a tela e encerrar o app devolvem os valores originais de tudo que foi tocado. |
| H8 | **Trava por versão de firmware.** Escrita só nas versões em que o formato dos dados foi conferido; nas outras, a tela fica só leitura com aviso. | |
| H9 | **Testes automáticos nunca abrem o hardware real.** Usam falsos e **gravações reais** guardadas como fixture (somente leitura). Testes que leem hardware ficam atrás de um marcador (`hardware`), fora da execução padrão. | `@pytest.mark.hardware` e `addopts = -m "not hardware"`. |
| H10 | **A primeira escrita real é uma sessão manual com o dono presente,** começando pelo campo mais inofensivo (variar 0 -> 1% e voltar) e nunca por força total ou limites. | Registrar a sessão no ledger. |
| H11 | **Interfaces só de leitura são abertas só para leitura** (flag de leitura, nada de feature report). | Uma interface que pertence a outro programa (jogos, drivers) nunca recebe escrita sua. |
| H12 | **Portas e dispositivos por lista conhecida.** Portas seriais só de VID/PID conhecidos e confirmadas por *handshake*; nunca abrir um dispositivo que **reinicia ao abrir** (alguns microcontroladores resetam). | A lista vive em um lugar só e tem teste. |
| H13 | **Testes nunca apertam teclas reais** (teclado nulo injetado) nem leem pastas reais do usuário. | `conftest.py` com fixtures `autouse` que bloqueiam rede, processos e pastas. |
| H14 | **Telemetria e leitores são só leitura** e nada que vem deles pode virar comando ao hardware. | Nenhum caminho de código liga leitura de jogo a escrita de força. |
| H15 | **Tudo que muda a base é registrável:** cada exceção a uma regra fica escrita no ponto do código onde vive (data e motivo) e na spec. | Ver o comentário da exceção de escrita de dashboards em `hub/server/app.py:473-475`. |

Como adaptar: copie esta tabela para o `CLAUDE.md` do seu projeto, troque os termos, e **crie um teste para cada regra** (os
testes do projeto de referência têm um para cada: lista branca, S4/S5, falha na releitura, fixture sem hardware, etc.).

## 5. Quando o processo muda

- **Funcionalidade pequena e bem entendida:** pode ir direto a spec curta + plano de poucas tarefas executado inline (foi assim nas etapas 2 a 8 de 13/10).
- **Mudança de visual:** tire prints "antes" para `docs/.../telas/` **antes de mexer**; sem isso não há como dizer o que mudou.
- **Pedido do dono no meio da execução:** vira "adendo" na spec (§8 da spec do cockpit tem dois: as seções empilhadas e a seção dos Pedais) e uma tarefa nova no plano, não uma exceção silenciosa.
- **Achado de revisão que contradiz o briefing:** a regra é corrigir e registrar com `Ruling:` e o custo do erro; só pare e pergunte se a correção muda comportamento prometido ao dono.
