# Templates de prompt

Quatro prompts prontos para colar numa sessão do Claude Code. Cada um:

1. diz **o que ler primeiro** (caminhos deste repositório; troque `<blueprint>` pelo caminho onde você copiou esta pasta, por exemplo
   `3d-web-dev-skills/` ou `.claude/skills/3d-interactive-web-blueprint/`);
2. fixa as **restrições que não mudam** (as do projeto de referência);
3. pede **entregáveis verificáveis** (testes e conferência no modo de teste), não só "código que parece certo".

Preencha o que está entre `{{chaves duplas}}` e apague as linhas que não se aplicam. Se um item não existir no seu projeto, diga isso no prompt
("não temos shaders próprios") em vez de deixar o assistente inventar.

Todos assumem português do Brasil para textos de interface e comentários, e identificadores em inglês (a convenção do projeto de referência).
Troque se a sua convenção for outra.

---

## (a) Projeto novo neste padrão

```text
Vou começar um projeto novo: {{nome}} — {{uma frase do que ele faz e para quem}}.

Quero o mesmo padrão do projeto de referência descrito em <blueprint>:
 - Python + FastAPI + uvicorn servindo uma interface web de JS puro (sem framework e sem bundler);
 - Three.js {{0.186.1 ou a versão que eu fixar}} copiado em vendor/ (importmap local), GSAP ScrollTrigger e Lenis locais;
 - janela de desktop com pywebview {{ou: só navegador}}, canal ao vivo por WebSocket, plug-ins por tipo de dispositivo;
 - página única com um herói 3D fixo ao topo e seções empilhadas que sobem por cima.

ANTES DE ESCREVER CÓDIGO:
1. Leia, nesta ordem: <blueprint>/SKILL.md, <blueprint>/01-planning/3d-feature-workflow.md,
   <blueprint>/03-utilities-and-pipeline/python-to-web-pipeline.md (seções 3, 4, 5, 7 e 9) e
   <blueprint>/02-design-and-3d-scenes/ui-tokens-and-overlay.md.
2. Faça o BRAINSTORMING comigo: uma pergunta por vez, com a sua recomendação, até fecharmos as decisões. Não crie arquivos antes disso.
3. Escreva a SPEC em docs/superpowers/specs/AAAA-MM-DD-{{tema}}-design.md no formato do fluxo (objetivo, decisões, estrutura, DESEMPENHO,
   SEGURANÇA E FALHAS com tabela falha->resultado, TESTES, fora do escopo). Pare e me mostre.
4. Só depois do meu OK, escreva o PLANO em docs/superpowers/plans/ com "Restrições globais" e "Foco da revisão" e tarefas com o teste já escrito.

Restrições que valem desde o dia 1:
 - estrutura de pastas da seção 3.2 de python-to-web-pipeline.md;
 - tokens CSS em um único :root; nenhuma cor literal fora dele; texto dinâmico sempre como texto (nunca innerHTML com dados);
 - nenhum recurso externo (sem CDN, sem fonte remota); bibliotecas em vendor/ com VERSION.txt e SHA-256, e um teste que recalcula os hashes;
 - o 3D só lê: módulos 3D não importam a API de escrita nem o token;
 - desenha só quando muda, no máximo 30 quadros por segundo, pausa com document.hidden, devicePixelRatio com teto de 1,5;
 - rotas de escrita só para admin (token + cliente local), guard de Host e Origin, corpo com limite de tamanho;
 - modo de teste (--modo-teste) com dispositivos simulados desde a primeira tarefa; testes nunca tocam rede, hardware ou pastas reais;
 - download de qualquer biblioteca só depois do meu OK (nome do arquivo, origem e tamanho).

Entregável da primeira sessão: spec + plano aprovados e a tarefa 1 (esqueleto que sobe em --modo-teste, GET /api/hub, WebSocket /ws/live
com full+diff, index.html com hero vazio, e uma suíte que roda com pytest e node --test).
Critério de aceite: suíte verde; abrir http://127.0.0.1:{{porta}}/ no modo de teste mostra a página sem erros no console.
```

Exemplo de preenchimento: `{{nome}}` = "Painel de bancada", `{{uma frase}}` = "mostra em 3D o estado de uma esteira e deixa o operador ajustar
parâmetros". `{{tema}}` = "esqueleto".

---

## (b) Modelo 3D novo, flutuante ou reativo

```text
Quero um modelo 3D novo: {{nome do modelo}} em {{onde: no herói do topo / em um card da seção X}}, que deve {{ficar flutuando / reagir a: a leitura Y}}.

LEIA ANTES: 
 - <blueprint>/02-design-and-3d-scenes/3d-lighting-and-materials.md (câmera, luzes, estilos, EDGE_LAYER, orientFor/fitScale/centrar, setGlow);
 - <blueprint>/02-design-and-3d-scenes/interactive-physics-parallax.md (epsilons, rotateModel, juntas, laço sob demanda e a PROPOSTA de flutuação na seção 9);
 - <blueprint>/03-utilities-and-pipeline/webgl-utils-and-perf.md (loadSeq, disposeTree, contextos WebGL, custo de rótulos);
 - <blueprint>/04-templates/golden-snippets.md (seções 1, 2 e 7).

Dados do modelo:
 - origem: {{arquivo .glb exportado do CAD, ou "montar em código"}}; unidades: {{m/cm/mm}}; frente = {{±x|±y|±z}}, topo = {{±x|±y|±z}};
 - peças que se movem: {{nenhuma / juntas com nome, eixo, limites, papel}};
 - tem arestas do CAD como linhas? {{sim/não}}; triângulos aproximados: {{n}};
 - girar por: {{origem do CAD (não centrar) / centro da caixa (centrar)}}.

Comportamento:
 - fonte do dado: {{campo ao vivo, ex.: live.angle_deg}}; fórmula: {{ex.: rotateModel(0, -angulo)}}; epsilon: {{ex.: 0,2°}};
 - {{se flutuante: amplitude em unidades de cena, frequência em Hz, pausar quando coberto pelas seções? sim/não}};
 - estados sem dado (desconectado, NaN, string): manter a última pose.

Restrições:
 - reaproveitar o createViewer existente; NÃO criar um contexto WebGL novo se o herói já serve (contexts atuais: {{n}});
 - estilo wireframe E renderizado devem funcionar; GLOW 0x7fb487 se acender peças;
 - se o .glb falhar: modelo em código como plano B, sem erro na tela; loadSeq contra corrida; descarte com disposeTree;
 - sem sombras e sem shader próprio, a menos que eu peça (hoje o padrão não os usa);
 - desenha só quando muda (request()), teto de 30/s, pausa com a aba escondida.

Entregáveis:
 1. a função pura da pose (módulo sem DOM, ex.: poses.js) com teste em node --test, incluindo dado estranho;
 2. o carregamento/montagem do modelo no visualizador, com teste estático do que não pode aparecer;
 3. a integração na página (mount/update/unmount, destroy no unmount);
 4. conferência no modo de teste: stats().frames não cresce com dado repetido por 2 s, document.hidden não desenha, 375 px e 1200 px sem overflow, console limpo.
Critério de aceite: suíte verde + os quatro itens da conferência descritos no relatório, com números.
```

---

## (c) Ajuste fino de física e parallax

```text
Preciso ajustar a sensação de {{o que: giro do volante / rolagem / parallax do herói / cursor / flutuação}}. Hoje está {{muito rígido/lento/rápido/pesado}}.

LEIA ANTES: <blueprint>/02-design-and-3d-scenes/interactive-physics-parallax.md inteiro e a seção 5 de <blueprint>/02-design-and-3d-scenes/ui-tokens-and-overlay.md.

Valores atuais (do projeto de referência; confira nos meus arquivos antes de assumir):
 - epsilons: WHEEL_EPS_DEG 0,2; KNOB_EPS 0,005; PEDAL_EPS 0,005; JOINT_EPS 0,001745 rad; MAX_FPS 30; MAX_DPR 1,5;
 - Lenis: lerp 0,12 (autoRaf false, acionado pelo gsap.ticker); cursor: quickTo duration 0,2 ease power3;
 - parallax do herói: yPercent 0->14, scale 1->0,86, opacity 1->0,4, start "top bottom", end "top top", scrub true, ease "none";
 - bordas: yPercent 0->-30, opacity 1->0, start "top 85%", end "top 35%";
 - entrada das seções: y 48->0, opacity 0,25->1, start "top bottom", end "top 60%"; --cover 0->1 (até 60% de preto);
 - OrbitControls: enableDamping false, minDistance 0,5x, maxDistance 1,6x.

Mudança desejada: {{descreva em termos do que se vê, ex.: "o volante parece ter atraso; quero acompanhar o sinal sem tremer"}}.

Regras:
 - mude UM parâmetro por vez e me diga o antes/depois em número; explique o efeito de cada um na sensação;
 - não adicione suavização que esconda ruído no lugar de tratá-lo (preferir epsilon a lerp);
 - toda animação continua ligada à rolagem (scrub) ou ao mouse; nada novo por tempo, exceto se eu pedir (nesse caso use a PROPOSTA da seção 9 de interactive-physics-parallax.md, com fase acumulada e dt limitado);
 - preservar: desenhar só quando muda, 30/s, pausa com aba escondida, reduced-motion cai no modo nativo;
 - qualquer constante nova vai para o topo do módulo com um comentário de uma linha dizendo a unidade;
 - se a conta é pura, deixe numa função testada com node --test.

Entregáveis: o diff mínimo; testes das contas puras; um roteiro de 5 passos para eu sentir a diferença no hub de teste (o que girar/rolar e o que observar);
os valores antigos e novos numa tabela. NÃO mude o design/CSS nem a estrutura de camadas.
```

---

## (d) Rotas e automação em Python

```text
Preciso de {{rotas novas / uma automação / um plug-in novo}} no backend: {{descrição curta do comportamento}}.

LEIA ANTES: <blueprint>/03-utilities-and-pipeline/python-to-web-pipeline.md (seções 4 a 9 e 13) e <blueprint>/04-templates/golden-snippets.md (seção 5).

Contrato:
 - método e caminho: {{ex.: POST /api/devices/{id}/actions/{name}}}; autenticação: {{viewer / admin / só local}};
 - corpo (JSON) e tipos estritos (bool não é número; `confirmed` só vale se for o booleano `true`, nunca a string "false");
 - resposta: {"ok": bool, "error": str|null, ...}; erros em português, sem rastreamento; exceção de plug-in vira {"ok": false, "error": "Erro interno no dispositivo: ..."} com log.exception;
 - limites: tamanho do corpo {{ex. 64 KB}}, nomes de arquivo com fullmatch, listas fixas para destinos/redirecionamentos.

Se for um plug-in novo: DevicePlugin com kind/page, describe(), live() BARATO (roda 30x/s por cliente), health(), start()/stop() idempotentes,
fábrica no registry (a ordem importa), um fake.py para o modo de teste e a página JS com mount(root, ctx) -> {update, unmount}.

Segurança (não negociável):
 - toda rota que muda estado leva Depends(require_admin); nenhuma escrita por aparelho remoto;
 - se mexer em hardware: aplicar as regras H1-H15 de <blueprint>/01-planning/3d-feature-workflow.md (lista branca no servidor, releitura, desfazer, trava por firmware, sem teste que abra hardware real);
 - o front só altera um RASCUNHO; o envio é por um botão "Aplicar" com trava busy.

Testes primeiro (TDD, com RED visível):
 - servidor: TestClient(create_app(FakeHub), base_url="http://127.0.0.1:8770") — casos: sem token (403), token em cliente remoto (403), Host errado (403),
   Origin errada (403), corpo grande (413), tipo errado (400), caminho feliz;
 - canal ao vivo: websocket_connect com URL ABSOLUTA; recv com prazo (falha em vez de travar);
 - plug-ins: fakes e fixtures gravadas; nada de rede, USB, serial, teclado real ou %APPDATA% real (conftest com fixtures autouse).

Entregáveis: testes que falham -> implementação -> suíte inteira verde; mensagem de commit pequena; uma linha no ledger/relatório com decisões ("Ruling: ... — cost if wrong: ...").
NÃO faça push nem rode testes de hardware.
```

---

## Dica de uso

- Cole só o template que se aplica. Se a tarefa mistura (por exemplo um modelo novo que também precisa de uma rota), faça dois prompts em sequência: primeiro (d), depois (b).
- Use os comandos do repositório (`.claude/commands/`) quando quiser que o assistente leia os documentos sozinho: `/plan-3d-feature`, `/new-floating-model` e `/review-design-and-perf`.
- Em todos: peça que o relatório final traga **números** (quadros, milissegundos, linhas de teste), não adjetivos.
