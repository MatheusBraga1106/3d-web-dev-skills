# Pipeline Python -> web: FastAPI, WebSocket, plug-ins, janela, testes e distribuição

Como o projeto de referência (Hub Sim Racing) leva dados de Python (dispositivos, jogos, arquivos) até uma interface web
com 3D, dentro de um aplicativo de desktop. Referências `arquivo:linhas` valem para o commit `ff34fab` (versão 0.2.1).
Os nomes de usuário, o endereço do repositório público e a chave de atualização do projeto original **foram deixados de
fora** de propósito; onde aparecem, estão como `<...>`.

## 1. Visão geral

```
[ hardware / jogos / arquivos ]
        |  (conectores: HID, serial, UDP, memória compartilhada)
        v
 plug-ins (DevicePlugin)  --registry-->  hub.registry.snapshot()  ---> live_loop() ---> WebSocket /ws/live
        |                                                                         \
        |                                  build_frame() ---> painel_loop() ---> WebSocket /ws/painel
        v
 FastAPI (create_app) -- REST /api/... -- StaticFiles("/") = hub/web (HTML + JS puro + Three.js local)
        ^
 uvicorn (thread "http") <-- ServerRunner  <-- Hub (composição)  <-- __main__.py
        |
 pywebview (janela WebView2 sem borda) + pystray (bandeja)  --> carrega http://127.0.0.1:<porta>/#admin=<token>
```

Ponto central: **um único processo Python** serve a interface por HTTP local; a janela é só um navegador embutido apontando
para o servidor. Isso dá o mesmo código para a janela do PC (admin, pode escrever) e para o celular na rede local
(somente leitura, pareado).

## 2. Stack e versões fixadas

`requirements.txt` do projeto (todas fixadas com `==`):

| Pacote | Versão | Papel |
|---|---|---|
| fastapi | 0.136.1 | API REST e WebSocket |
| uvicorn | 0.47.0 | Servidor ASGI (numa thread) |
| websockets | 17.2 | Implementação de WebSocket do uvicorn (`ws="websockets-sansio"`) |
| pywebview | 6.2.1 | Janela (WebView2/Edge Chromium) |
| pythonnet | 3.2.1 | Ponte .NET do pywebview no Windows |
| pystray | 0.19.5 | Ícone na bandeja |
| pillow | 12.3.0 | Validar imagens enviadas e gerar o ícone |
| qrcode | 8.2 | QR de pareamento |
| cryptography | 50.0.2 | Só no release (assinar). O hub só **verifica** (Ed25519 em Python puro) |

Dev (`requirements-dev.txt`): `pytest==9.1.1`, `httpx==0.28.1` (para o `TestClient`), `pyinstaller==6.22.3`,
`pyinstaller-hooks-contrib==2026.8`. Front-end: Three.js 0.186.1, GSAP 3.15.0 (ScrollTrigger), Lenis 1.3.26, todos
**copiados para `hub/web/vendor/`** com `VERSION.txt` e SHA-256; o teste `test_vendor_*_matches_recorded_hashes` confere os hashes.
`.gitattributes` marca `hub/web/vendor/** -text` para os bytes nunca serem alterados por conversão de fim de linha.

## 3. Estrutura de pastas

### 3.1 A do projeto de referência (resumo)

```
projeto/
  CLAUDE.md  README.md  hub.spec  pytest.ini  requirements*.txt  run_hub.pyw  iniciar.bat
  hub/
    __main__.py            # argparse, logging, instância única, composição, janela
    paths.py               # pastas de dados (%APPDATA%\<App>), logs, modelos
    core/                  # hub.py (composição), plugin.py (contrato), registry.py, config.py,
                           # updater.py, update_manifest.py, ed25519.py, models3d.py, paineis.py
    server/                # app.py (create_app), live.py, painel.py, auth.py, runner.py
    plugins/<kind>/        # um pacote por tipo de dispositivo: plugin.py + __init__.py (factory())
    connectors/            # hid_win.py, joystick_winmm.py, serial_win.py (ctypes, só Windows)
    bindings/              # leitores de arquivos de configuração dos jogos (só leitura)
    shell/                 # window.py (pywebview), tray.py, icon.py, single_instance.py, autostart.py
    web/                   # css/, js/ (+ js/pages/), painel/, vendor/, assets/, index.html
    fusion_addin/          # add-in do Fusion (exporta modelos .glb)
  installer/hub.iss        # Inno Setup
  tools/                   # release.py, chave_release.py, gerar_logo.py ...
  tests/                   # test_*.py, js/*.test.mjs, js/helpers/, fixtures/
  docs/superpowers/        # specs/, plans/, etapas/, sdd-estado/ (ver 01-planning)
  assets/marca/            # logo (fonte única)
```

### 3.2 Árvore recomendada para projetos híbridos (Python + web + 3D)

```
meu-app/
  CLAUDE.md                      # regras que não mudam + comandos + onde está o histórico
  pyproject.toml | requirements*.txt   # versões FIXADAS
  app/
    __main__.py                  # CLI: --modo-teste, --sem-janela, --porta-web, --dados
    paths.py                     # única fonte das pastas de dados
    core/                        # composição, contrato de plug-in, registry, config
    server/                      # create_app(core), live.py (WS), auth.py, runner.py
    plugins/<tipo>/              # um pacote por fonte de dados; fake.py ao lado (para testes e modo de teste)
    shell/                       # janela, bandeja, instância única
    web/
      index.html                 # importmap -> /vendor/...
      css/tokens.css  css/app.css
      js/ (api.js, ui.js, app.js, webgl.js, poses.js, motion.js, hero.js, pages/*.js)
      vendor/<lib>/ (+ VERSION.txt com sha256)
      assets/models/
  tests/
    conftest.py                  # fixtures autouse que BLOQUEIAM rede, hardware e pastas reais
    test_*.py                    # unidade, servidor (TestClient), "estáticos" do front
    js/*.test.mjs  js/helpers/   # node --test com DOM falso
    fixtures/                    # gravações reais (somente leitura) dos dispositivos
  installer/ tools/ docs/
```

Regras que fazem essa árvore funcionar: (1) `poses.js`-style: toda conta do front em **módulos puros** (sem DOM, sem
Three.js) para testar com `node --test`; (2) cada plug-in tem um `fake.py` para o **modo de teste**; (3) `web/` é servida como
arquivos estáticos, sem build.

## 4. FastAPI e uvicorn

### 4.1 Fábrica da aplicação

`hub/server/app.py:137-148`: `create_app(hub)` recebe o objeto de composição e devolve o `FastAPI`. Nada é global; os
testes constroem a app com um `FakeHub`.

```python
def create_app(hub) -> FastAPI:
    mimetypes.add_type("text/javascript", ".js")      # o Windows pode mapear .js para text/plain pelo registro
    mimetypes.add_type("text/javascript", ".mjs")
    mimetypes.add_type("text/css", ".css")
    app = FastAPI(title=APP_NAME, version=__version__, docs_url=None, redoc_url=None, openapi_url=None)
    app.add_middleware(BodyLimit)                       # por dentro do guard: o 413 chega direto na rota
    ...
    app.mount("/", StaticFiles(directory=str(hub.web_dir), html=True), name="web")   # SEMPRE por último
    return app
```

Lições embutidas:

- **Tipos MIME fixos.** No Windows o `mimetypes` lê o registro, e módulos ES servidos como `text/plain` são recusados pelo
  navegador: janela em branco. Por isso as três linhas `add_type`.
- **Documentação desligada** (`docs_url=None`, `redoc_url=None`, `openapi_url=None`): é um app local, sem superfície extra.
- **O `StaticFiles` em `"/"` vai por último**, depois de todas as rotas, ou engoliria as rotas da API.
- **Ordem das rotas:** `/api/paineis/presets` é declarada **antes** de `/api/paineis/{pid}` para a rota fixa vencer a dinâmica.
- Rotas síncronas (`def`) rodam no pool de threads do Starlette; as `async def` que gravam arquivo chamam
  `run_in_threadpool(store.criar, ...)` para não bloquear o laço de eventos.
- Plug-in com defeito não vira 500: `_call_plugin` e `_describe` capturam, registram com `log.exception` e devolvem
  `{"ok": False, "error": "Erro interno no dispositivo: ..."}` com HTTP 200 (`app.py:63-96`). O front mostra a mensagem em português.

### 4.2 Rotas reais

Auth: **V** = qualquer viewer (PC ou celular pareado), **A** = admin (`X-Hub-Admin` + cliente local), **L** = só cliente
local. Todas passam antes pelo `guard` (host, origem, pareamento).

| Método e caminho | Auth | O que faz |
|---|---|---|
| `GET /api/hub` | V | Nome, versão, `test_mode`, `admin` (true só com o token), preferências, aviso de carga. |
| `GET /api/devices` | V | Lista de dispositivos (`describe()` + `health`). |
| `GET /api/devices/{id}` | V | Um dispositivo com `live` e `values`. |
| `POST /api/devices/{id}/settings` | A | Aplica ajustes (`{"changes": {...}, "confirmed": true}`); só o booleano `true` confirma. |
| `POST /api/devices/{id}/actions/{name}` | A | Executa uma ação do plug-in. |
| `POST /api/devices/{id}/rename` | A | Renomeia. |
| `DELETE /api/devices/{id}` | A | Remove (só dispositivos adicionados manualmente). |
| `GET /api/candidates` / `POST /api/candidates/add` | V / A | Dispositivos não reconhecidos / adicionar um com nome. |
| `POST /api/images` / `GET /api/images/{name}` | A / V | Envio de imagem (base64, máx. 5 MB, validada com Pillow) / leitura (nome `[0-9a-f]{16}\.(png|jpg)` com `fullmatch`). |
| `GET /api/models`, `GET /api/models/{mid}/{name}` | V | Lista e arquivo dos modelos 3D (`Cache-Control: no-cache`, porque o add-in regrava). |
| `POST /api/fusion/addin` | A | Instala o add-in do Fusion. |
| `GET /api/buttons` | V (`?recarregar=1` exige A) | Botões de cada aparelho e binds dos jogos. |
| `GET /api/update`, `POST /api/update/check`, `POST /api/update/install` | V / A / A | Estado, procurar, instalar (409 se algo crítico estiver aberto). |
| `GET /api/prefs`, `POST /api/prefs` | V / A | Preferências validadas no servidor. |
| `GET /api/pairing`, `POST /api/pairing/code`, `GET /api/pairing/qr.png`, `DELETE /api/pairing/{pid}` | A / A / L / A | Pareamento do celular. |
| `GET /pair?code=...&to=...` | público (fora do guard de pareamento) | Troca o código por um cookie e redireciona (destino numa lista fixa). |
| `POST /api/show`, `POST /api/open-logs` | L / A | Traz a janela para a frente (usado pela segunda instância) / abre os logs. |
| `GET /marca/{name}` | V | A logo, só por nomes exatos da lista. |
| `GET /painel`, `GET /painel/manifest.json` | V (manifest e ícones públicos) | Página do painel de telemetria e manifest do "Adicionar à tela inicial". |
| `GET /api/paineis/presets`, `GET/POST /api/paineis`, `GET/PUT/DELETE /api/paineis/{pid}`, `POST /api/paineis/{pid}/duplicar` | V (**escrita permitida a viewer**) | Dashboards do painel (a única escrita permitida ao celular; corpo máx. 64 KB; versão otimista com 409). |
| `WS /ws/live` | V | Estado ao vivo de todos os dispositivos (seção 5). |
| `WS /ws/painel` | V | Quadros do painel de telemetria (seção 5). |
| `GET /` e estáticos | V | `StaticFiles(html=True)` sobre `hub/web`. |

### 4.3 uvicorn numa thread (`hub/server/runner.py`)

```python
cfg = uvicorn.Config(self.app, host=host, port=self.port, log_level="warning", lifespan="off",
                     ws="websockets-sansio", proxy_headers=False, timeout_graceful_shutdown=2, log_config=None)
self.server = uvicorn.Server(cfg)
self.thread = threading.Thread(target=self._serve, args=(self.server,), name="http", daemon=True)
```

- `ws="websockets-sansio"`: a opção `auto` usaria `websockets.legacy`, obsoleta.
- `proxy_headers=False`: "cliente local" nunca depende de `X-Forwarded-For`.
- `log_config=None`: **sem console** (`pythonw`, `.exe` com `console=False`) o `dictConfig` padrão do uvicorn quebra com
  `Unable to configure formatter 'default'` (`sys.stdout` é `None`). Bug real, reproduzido e coberto por teste
  (`test_runner_starts_without_console_streams`). Por isso o `__main__` também protege com `if sys.stdout:`.
- `lifespan="off"`; `timeout_graceful_shutdown=2` para o encerramento não esperar conexões WebSocket.
- `start()` espera `server.started` (até 5 s) e, se a porta estiver ocupada, para a thread e levanta
  `RuntimeError("Não consegui abrir a porta ...")`. `restart(host)` espera 0,3 s (a resposta da requisição que pediu o
  reinício precisa sair) e reabre; é como "acesso pelo celular" muda de `127.0.0.1` para `0.0.0.0` sem reiniciar o app.

## 5. Canais ao vivo por WebSocket

### 5.1 `/ws/live`: tudo, só o que mudou, até 30 vezes por segundo

`hub/server/live.py` inteiro:

```python
def diff(prev: dict, cur: dict) -> dict:
    changed = {k: v for k, v in cur.items() if prev.get(k) != v}
    changed.update({k: None for k in prev if k not in cur})     # None = dispositivo saiu
    return changed

async def _send_loop(ws, snapshot, interval: float) -> None:
    prev = snapshot()
    await ws.send_json({"t": "full", "devices": prev})
    while True:
        await anyio.sleep(interval)
        cur = snapshot()
        changed = diff(prev, cur)
        if changed:
            await ws.send_json({"t": "diff", "devices": changed})
        prev = cur

async def live_loop(ws, snapshot, interval: float = 1 / 30) -> None:
    await serve_until_disconnect(ws, lambda: _send_loop(ws, snapshot, interval))
```

- Mensagens: `{"t": "full", "devices": {id: {"health": {...}, "live": {...}}}}` e depois
  `{"t": "diff", "devices": {id: {...} | null}}`.
- **`serve_until_disconnect`** (`live.py:24-49`) roda o emissor num grupo de tarefas do anyio **e** escuta
  `ws.receive()` em paralelo. Sem isso, um canal parado (nada muda, nada é enviado) nunca perceberia que o cliente saiu, e o
  uvicorn esperaria essa tarefa para sempre ao reiniciar. Cancelamento cruzado via `tg.cancel_scope`.
- O cliente (`hub/web/js/api.js:52-66`): reconexão com atraso inicial de 1 s, dobrando até 5 s; `onopen` zera; código **4401**
  (não pareado) não insiste.
- Limitação conhecida e registrada no projeto: `snapshot()` roda no laço de eventos; um `live()` lento de plug-in atrasa o
  canal. Mantenha `live()` barato (ler estado em memória, não fazer I/O).

### 5.2 `/ws/painel`: um quadro por vez, sempre o mais novo

`hub/server/painel.py`. Intervalo 1/30 s, `KEEPALIVE = 1.0` (reenvia mesmo sem mudança, para o painel saber que o canal vive),
`SEND_TIMEOUT = 10.0` (`anyio.fail_after` no envio; cliente que não consome é desligado e o painel reconecta sozinho).

```python
async def _send_loop(ws, build, interval, keepalive, clock, sleep) -> None:
    n, prev, last_sent, mapa_sent = 0, None, 0.0, None
    while True:
        cur = build()
        mapa = cur.pop("mapa", None)
        now = clock()
        if prev is None or cur != prev or now - last_sent >= keepalive:
            n += 1
            prev, last_sent = cur, now
            await _send(ws, {"k": "estado", "n": n, **cur})
        if mapa and mapa["versao"] != mapa_sent:      # o mapa grande vai à parte, só quando a versão muda
            mapa_sent = mapa["versao"]
            await _send(ws, {"k": "mapa", **mapa})
        await sleep(interval)
```

- **Atraso nunca acumula:** o laço monta e envia **um** quadro e só monta o seguinte depois que o envio terminou. Num celular
  lento ele manda menos quadros (sempre o mais novo), nunca uma fila de quadros velhos.
- `clock` e `sleep` são parâmetros: os testes injetam relógio falso e não esperam tempo real.
- O cliente (`hub/web/painel/canal.js`) só recebe; reconecta com `min(5000, 500 * 2^tentativa)` ms; status:
  `conectando | conectado | reconectando | nao-pareado (4401) | bloqueado (4403)`; `parseMensagem` descarta lixo.
- No cliente, `createPanel().push(frame)` **descarta** quadros antigos e ignora números `n` menores ou iguais ao último.

### 5.3 Handshake de segurança dos WebSockets (`app.py:412-422`)

```python
async def ws_open(ws: WebSocket) -> bool:
    if not host_ok(ws) or not origin_ok(ws):
        await ws.close(code=4403)      # antes do accept: o handshake é recusado (HTTP 403)
        return False
    if not auth.is_viewer(host_of(ws), ws.cookies.get(COOKIE)):
        await ws.accept()               # aceita só para o cliente ver o código 4401
        await ws.close(code=4401)
        return False
    await ws.accept()
    return True
```

## 6. Plug-ins e registry

### 6.1 Contrato (`hub/core/plugin.py`)

Um `DevicePlugin` tem `kind`, `page` (qual JS monta a seção), `id`, `name` e os métodos: `start()`, `stop()`, `inputs()`,
`settings_spec()`, `settings_values()`, `actions_spec()`, `live()`, `health()`, `apply_settings(changes, confirmed)`,
`action(name, params)` e `describe()`. Dataclasses: `SettingSpec` (key, label, kind `int|bool|choice`, group, min/max/step,
unit, choices), `ActionSpec`, `Health(connected, rate_hz, error, detail)`, `PluginContext` (device_id, config, services,
test_mode, data_dir, log). A página JS correspondente exporta `mount(root, ctx) -> { update(entry), unmount() }`
(`hub/web/js/pages/*.js`).

### 6.2 Registro e descoberta (`hub/core/registry.py`)

- `PluginFactory(kind, label, match, create, device_id, auto_add)` por tipo; **a ordem das fábricas importa** (a primeira que
  reconhece o candidato assume; a genérica fica por último).
- `DeviceRegistry`: `add` (inicia e, se `start()` falhar, desregistra e tenta de novo na próxima varredura), `get`, `all` (ordem
  fixa `FIXED_ORDER = ("ffbeast", "rim", "shifter", "telemetry")`, o resto por nome), `remove`, `handle_candidates`, `snapshot`
  (um plug-in que lança vira `{"health": {error: "erro interno: ..."}, "live": {}}`: nunca derruba o canal), `stop_all`.
- Duas travas: `_lock` (RLock, dicionário) e `_scan_lock` (serializa varreduras; **não** é segurada enquanto o plug-in abre o
  hardware).
- `ServiceRegistry`: serviços compartilhados entre plug-ins (ex.: o aro registra uma saída de marcha que o câmbio consome).
- `Hub._scan_loop` varre a cada `SCAN_INTERVAL_S = 2.0`; `LogThrottle` limita a um log com rastreamento por minuto para cada
  erro distinto (um defeito persistente não inunda o log).

Exemplo de fábrica (`hub/plugins/rim/__init__.py`):

```python
def factory() -> PluginFactory:
    return PluginFactory(kind="rim", label="Volante",
        match=lambda c: c.kind == "joystick" and (c.vid, c.pid) in RIM_IDS,
        create=lambda ctx, name, c: RimPlugin(ctx, name),
        device_id=lambda c: "rim", auto_add=True)
```

## 7. Segurança

Tudo em `hub/server/app.py` (middleware `guard`, linhas 187-199) e `hub/server/auth.py`.

| Defesa | Como | Por quê |
|---|---|---|
| **Host** | `host_ok`: cabeçalho `Host` precisa estar em `{127.0.0.1:<porta>, localhost:<porta>, [::1]:<porta>, <ip da rede>:<porta>}`; senão 403 | Barra DNS rebinding. |
| **Origem** | Em métodos que não são GET/HEAD, se há `Origin`, tem de ser `http://<Host>`; senão 403 | Barra CSRF de outro site no navegador. |
| **Admin** | `X-Hub-Admin` com o token da execução **e** cliente local. `secrets.compare_digest` em **bytes** (com `str` não-ASCII levantaria `TypeError` e viraria 500). Token: `secrets.token_urlsafe(32)`, novo a cada execução | O celular nunca altera nada. |
| **Pareamento** | Código de uso único (`token_urlsafe(16)`, TTL 300 s) vira cookie `hub_viewer` (httponly, samesite lax, 1 ano); só o **hash SHA-256** do token fica no disco | Aparelhos remotos precisam ser autorizados no PC. |
| **Token só no fragmento** | A janela abre `http://127.0.0.1:<porta>/#admin=<token>`; o `#...` nunca vai ao servidor; `api.js` guarda em `sessionStorage` e apaga da URL com `history.replaceState` | Não cai em log nem no histórico. |
| **Limites** | `BodyLimit` 8 MB (Content-Length e contagem de partes); dashboards 64 KB; imagens 5 MB | Evita esgotar memória. |
| **Caminhos** | Nomes validados com `fullmatch` e listas fixas (`IMAGE_RE`, `MARCA_ARQUIVOS`, `PAIR_DESTINOS`) | Barra `..` e barra invertida no Windows. |
| **Proxy** | `proxy_headers=False` | "Local" nunca vem de `X-Forwarded-For`. |
| **Bind** | `127.0.0.1` por padrão; `0.0.0.0` só com "acesso pelo celular" ligado; se a porta da rede falhar, volta a `127.0.0.1` e avisa | Superfície mínima. |
| **Ponte do pywebview** | Sem `js_api`; só funções soltas registradas por nome com `window.expose(*controls)` (minimizar, maximizar, esconder, `is_frameless`) | O `js_api` expunha `_window.gui.os.system` pelo `getattr` encadeado. Achado de revisão. |

### 7.1 A regra S2 (escrita só por clique explícito no PC), em versão genérica

No projeto original, S2 é a regra: **qualquer escrita que afete o hardware só acontece por um clique explícito do usuário na
janela do PC**: nunca automática, nunca ao iniciar, nunca enquanto um controle deslizante se move, nunca por um
aparelho remoto. Como foi implementada, e como replicar:

- Servidor: toda rota que muda estado leva `dependencies=[Depends(require_admin)]`; `require_admin` exige o token **e** cliente
  local (`app.py:201-203`). A rota `POST /api/devices/{id}/settings` ainda exige `confirmed is True` (booleano estrito) quando o
  plug-in pede confirmação.
- Front: os controles só alteram um **rascunho**; só o botão "Aplicar" envia. Um teste estático
  (`test_ffbeast_page_never_sends_on_slider_input`) garante que o handler de `input` do slider nunca chama `post`.
- A **única exceção documentada** é a escrita de dashboards do painel (`/api/paineis*`), que não afeta hardware nem ajustes;
  fica registrada na spec, no docstring de `auth.py` e num comentário no ponto da rota (`app.py:473-475`).
- Regra de projeto complementar: cada exceção a uma regra de segurança é escrita onde ela vive, com a data e o motivo.

## 8. Janela (pywebview) e bandeja (pystray)

`hub/shell/window.py`, `tray.py`, `single_instance.py`:

- **Janela:** `webview.create_window("Hub Sim Racing", url, width=1200, height=780, min_size=(900, 600), frameless=True,
  easy_drag=False, hidden=start_hidden, background_color="#F5F5F3")` e `webview.start(gui="edgechromium")`. Sem borda, com barra
  de título própria em HTML (`.titlebar.pywebview-drag-region`, `index.html:13-20`); o `app.js` só a mostra se
  `pywebview.api.is_frameless()` responder.
- **Fechar esconde:** `window.events.closing` devolve `False` e chama `window.hide()`; só "Sair" da bandeja encerra
  (`state["quitting"] = True; window.destroy()`).
- **Reserva:** se `pywebview` ou `pythonnet` falharem, `_run_edge` abre `msedge --app=<url>` com perfil próprio
  (`--user-data-dir=<dados>/edge-perfil`, para a URL com o token não entrar no histórico do perfil de uso) ou o navegador padrão.
- **Bandeja:** `pystray.Icon` com "Abrir Hub", linha de status (●/○ por dispositivo, atualizada a cada 3 s por uma thread) e
  "Sair". O ícone muda se algum dispositivo está conectado. Título limitado a 127 caracteres (limite do `szTip` do Windows).
- **Instância única:** mutex nomeado (`CreateMutexW`, `Local\HubSimRacing`). A segunda cópia chama `POST /api/show` na primeira
  (sem proxy: `ProxyHandler({})`) e sai.
- **Modos de linha de comando:** `--modo-teste` (dispositivos simulados, sem teclas reais, pasta de dados própria, sem varrer USB,
  ignora `HUB_DATA_DIR`), `--sem-janela` (só o servidor), `--minimizado`, `--porta-web`, `--udp-porta`, `--dados`.
  `HUB_ADMIN_TOKEN` fixa o token (útil para abrir o hub de teste sempre na mesma URL).
- Encerramento (`__main__._shutdown`): cada passo em `try/except` com `logging.exception`; um passo que falha não impede os
  seguintes; por fim libera o mutex.

## 9. Testes

Rodar: `.venv\Scripts\python -m pytest -q` (inclui os testes de JS, que chamam o Node). `pytest.ini`:
`addopts = -m "not hardware"` e o marcador `hardware`.

| Camada | Arquivos | Como |
|---|---|---|
| Unidade Python | `tests/test_*.py` | Fakes e **gravações reais** de dispositivos em `tests/fixtures/` (somente leitura). |
| Servidor | `tests/test_server.py`, `test_server_painel.py` | `TestClient(create_app(FakeHub(...)), base_url="http://127.0.0.1:8770")`. WebSocket com `client.websocket_connect(WS)` e, para o servidor real, `websockets.sync.client.connect`. Helper `recv(ws, timeout)` que **falha em vez de travar** se não vier mensagem. |
| "Estáticos" do front | `tests/test_web_static.py` | Leem o texto dos `.js`/`.css`/`.html` e afirmam regras: sem `innerHTML` nos módulos de UI e 3D, sem recursos externos, hashes dos `vendor/`, `forceContextLoss()` presente, `layers.set(EDGE_LAYER)` presente, importmap só local, tokens existem, etc. |
| JS puro | `tests/js/*.test.mjs` | `node --test`. `tests/test_web_js.py` chama o Node por arquivo (com `skipif` se o Node não existe). |
| Hardware (leitura) | marcados `@pytest.mark.hardware` | Só com `-m hardware`, nunca em testes automáticos de escrita. |

Proteções em `tests/conftest.py` (todas `autouse`): bloqueiam a rede e a pasta real de instaladores
(`no_real_update_network`), a memória compartilhada de jogos, a porta real do OutGauge, a lista real de processos e a pasta
real de um jogo. **Um teste nunca toca o ambiente real do usuário.** `tmp_data` isola a pasta de dados.

### 9.1 Testar JS com DOM falso e Three.js real, sem navegador

Funções puras (`poses.js`) importam-se direto. Para módulos que usam `three` (`joints.js`), um gancho de resolução aponta o
nome `three` para a cópia local (`tests/js/joints.test.mjs:1-15`):

```js
import { register } from "node:module";
import { pathToFileURL } from "node:url";
const threeUrl = pathToFileURL(resolve(here, "../../hub/web/vendor/three/three.module.js")).href;
register("data:text/javascript," + encodeURIComponent(
  `export async function resolve(s, c, n) { if (s === "three") return { url: ${JSON.stringify(threeUrl)}, shortCircuit: true }; return n(s, c); }`));
const THREE = await import("three");
const { mountJoints, setJointAngle } = await import("../../hub/web/js/joints.js");
```

Para widgets de canvas e páginas, `tests/js/helpers/fake_dom.mjs` implementa um DOM mínimo (`FakeEl`, `FakeText`, `install()`),
com um contador `counters.writes` de escritas no DOM (prova que "widget parado não toca no DOM") e um `getContext("2d")` que
registra as chamadas. Estado do projeto na data da etapa de distribuição (13/10/2026): 899 testes Python e 67 do Node.

## 10. PyInstaller (`hub.spec`)

```python
a = Analysis(["run_hub.pyw"],
    datas=[("hub/web", "hub/web"), ("assets/marca", "assets/marca"), ("hub/fusion_addin", "hub/fusion_addin"),
           ("hub/plugins/ffbeast/data", "hub/plugins/ffbeast/data")],
    hiddenimports=["hub.plugins.ffbeast.plugin", "hub.plugins.rim.plugin", "hub.plugins.shifter_wifi.plugin",
                   "hub.plugins.generic_hid.plugin", "uvicorn.logging", "uvicorn.loops.auto", "uvicorn.protocols.http.auto",
                   "uvicorn.protocols.http.h11_impl", "uvicorn.loops.asyncio", "uvicorn.protocols.websockets.auto",
                   "uvicorn.protocols.websockets.websockets_sansio_impl", "websockets", "uvicorn.lifespan.off"])
exe = EXE(pyz, a.scripts, [], exclude_binaries=True, name="HubSimRacing", console=False, icon="assets/marca/logo.ico")
coll = COLLECT(exe, a.binaries, a.datas, name="HubSimRacing")
```

- **`onedir`** (`COLLECT`), não `onefile`: abre mais rápido e o instalador empacota a pasta.
- **`datas` copia `hub/web` inteira** (HTML, JS, vendor): é o que o `StaticFiles` serve; `hub.web_dir` resolve por
  `Path(__file__).resolve().parents[1] / "web"`, que funciona no `.exe` (arquivos em `_internal`).
- **`hiddenimports`:** os plug-ins são importados dentro de funções (o PyInstaller não os vê) e o uvicorn escolhe a
  implementação de WebSocket por texto (`ws="websockets-sansio"`), então os módulos `*_impl` precisam ser listados. Faltar um deles
  só aparece **no `.exe`**, não no pytest.
- **`console=False`:** sem `stdout`/`stderr`; ver `log_config=None` na seção 4.3.
- Feche o app antes de gerar: a pasta `dist` fica travada com ele aberto. Se o app do usuário estiver aberto, gere em
  `--distpath` e `--workpath` alternativos.

## 11. Inno Setup (`installer/hub.iss`)

Compilar: `ISCC.exe /DVersao=0.2.0 installer\hub.iss` (a versão vem da linha de comando; `#error` se faltar). Pontos de projeto:

- `AppId` **fixo** (um GUID): é o que faz uma versão nova instalar por cima da anterior e o desinstalador achar o programa.
  Nunca mude depois do primeiro lançamento.
- Instalação **por usuário, sem administrador**: `PrivilegesRequired=lowest`, `DefaultDirName={localappdata}\Programs\<App>`.
- `CloseApplications=yes` + `CloseApplicationsFilter=<exe>` fecham o app aberto (silenciosamente na atualização).
- Parâmetros próprios: `/RELANCAR` abre o app de novo no fim, mesmo em modo silencioso (seção `[Run]` com `Check: DeveRelancar`).
- **Os dados do usuário nunca são tocados** na instalação nem na atualização; a desinstalação pergunta "Apagar também meus
  ajustes...?" com **Não** como padrão (`MB_DEFBUTTON2`); no modo silencioso só com `/APAGARDADOS`.
- Pós-instalação: se "iniciar com o Windows" já estava ligado, reaponta a chave `Run` para o `.exe` instalado (nunca liga sozinho).
- Opcionais para ensaio: `/DOrigem=<pasta>` e `/O<pasta de saída>`.

## 12. Release assinado e atualização automática

### 12.1 O lado do hub (`hub/core/updater.py`, `update_manifest.py`, `ed25519.py`)

- O manifesto `latest.json` tem `versao, url, sha256, tamanho, notas, minima, assinatura`. A assinatura é **Ed25519** em base64
  sobre o JSON canônico (`json.dumps(corpo, sort_keys=True, separators=(",", ":"), ensure_ascii=False)` sem o campo `assinatura`).
- O hub **só verifica** (RFC 8032 em Python puro: não precisa de pacote novo no `.exe`); a chave pública está embutida em
  `hub/core/update_key.py` (não reproduzida aqui). A privada **nunca** entra no repositório (`.gitignore` cobre `*.pem` e
  `.hub-sim-racing/`).
- Conferências (`update_manifest._check_fields`): JSON sem campos repetidos e sem `NaN`/`Infinity`; manifesto até 64 KB; URL
  **HTTPS**, sem usuário, porta só 443, em **lista fixa de três hosts**, só ASCII visível; `sha256` com 64 hex minúsculos; `tamanho`
  de 1 byte a 300 MB; versões `X.Y.Z` estritas (sem pré-lançamento).
- `Updater`: primeira verificação 30 s depois de abrir e depois a cada 6 h (se a preferência estiver ligada); baixa com
  conferência de tamanho e SHA-256 e **confere de novo logo antes de rodar**; nunca rebaixa; instalar é só por clique de admin
  (`POST /api/update/install`) e recusado com 409 se `update_blocker()` (um plug-in com algo crítico aberto) disser que não.
  `fetch`, relógio, `spawn` e pastas são **injetados**, então os testes não usam rede nem executam instalador.
- Argumentos do instalador: `/VERYSILENT /SUPPRESSMSGBOXES /NORESTART /CLOSEAPPLICATIONS /RELANCAR`.

### 12.2 O lado do desenvolvedor (`tools/release.py`)

`py tools/release.py X.Y.Z --ensaio` e depois sem `--ensaio` (pergunta `Publicar X.Y.Z? (s/N)` antes de publicar).

1. Confere árvore limpa e branch de lançamento.
2. Muda `__version__`.
3. Roda as suítes Python e Node (para no primeiro erro).
4. PyInstaller.
5. Inno Setup (procura o `ISCC.exe`; nunca instala nada sozinho).
6. SHA-256 do instalador.
7. Monta e assina o `latest.json`, e **valida com a mesma função que o hub usa** (`um.parse_with_reason`): se o manifesto gerado
   não passar na conferência do hub, o release falha.
8. Commit e tag no repositório privado.
9. Cria o Release no repositório **público** (só instalador, site e `latest.json`; nenhum código) com `gh` e atualiza o site.

O `--ensaio` faz tudo até o passo 7 numa pasta temporária e **não publica nada** (nem commit, nem tag, nem push). A versão volta ao
valor original em qualquer falha ou desistência (`finally` restaura o `__init__.py`), exceto depois do commit da versão.
Sem a chave privada real embutida/disponível, o script recusa publicar (a chave de teste só serve para o ensaio).

## 13. Checklist para copiar o pipeline

1. `create_app(core)` como fábrica, `StaticFiles` por último, MIME fixos, docs desligadas.
2. Guard: Host, Origin, admin por token só local, pareamento por cookie com hash.
3. Canal ao vivo: `full` + `diff` (ou "um quadro por vez") e `serve_until_disconnect`.
4. Plug-ins com `fake.py`, `health()` e `live()` baratos; falha de plug-in nunca derruba o canal.
5. Testes: `conftest` que bloqueia o ambiente real; estáticos para regras de front; `node --test` para as contas.
6. `hub.spec` com `hiddenimports` explícitos; testar o `.exe` em modo de teste antes de publicar.
7. Instalador com `AppId` fixo; atualização assinada com lista fixa de hosts.
