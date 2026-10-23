# Proposta: Web UI para o `mcp`

> Status: proposta para discussão · Base: `main` @ `71d884b`
> Protótipo navegável: [`prototype.html`](./prototype.html) · Telas em [`overview.png`](./overview.png), [`servers.png`](./servers.png), [`edit.png`](./edit.png), [`playground.png`](./playground.png), [`activity.png`](./activity.png), [`access.png`](./access.png), [`settings.png`](./settings.png)

## 1. Resumo

O `mcp` já tem tudo o que uma interface precisa para mostrar — servidores, catálogo de tools, decisões de ACL, auditoria de cada chamada, instâncias rodando — mas **nada disso é acessível fora do terminal**. A proposta é um subcomando `mcp ui` que sobe uma Web UI local, servida pelo mesmo binário, sem daemon novo, sem SDK e sem segundo artefato para instalar.

Três decisões estruturantes:

1. **Mesmo binário, dois modos.** `mcp ui` (local, lê config/banco/instâncias deste host) e `mcp serve --http … --management` (time, lê e escreve num proxy remoto, atrás de `serverAuth`/`adminRoles`). A UI é a mesma; a fonte de dados muda.
2. **Leitura por padrão, escrita com plano.** Nenhum campo é editado "ao vivo": toda alteração produz um *diff* contra o arquivo real, passa por validação e só então é aplicada atomicamente e publicada com `reload`. O arquivo continua sendo a fonte da verdade — a UI nunca guarda estado de config.
3. **A UI não inventa autorização.** Em modo remoto ela reusa a ACL existente (o catálogo que você vê é o que sua identidade pode chamar) e exige `adminRoles` para escrever. Em modo local ela é, por definição, tão poderosa quanto o seu shell — e a proposta diz isso na cara, não esconde.

**Decidido:** HTML renderizado no servidor com **htmx embutido** (sem Node, sem etapa de build) e o modo local **editável** desde a primeira versão — a escrita passa pelo protocolo `plan → apply → reload` da §6.3.

**Escopo mínimo que entrega valor:** Overview com saúde dos backends, catálogo de tools com schemas, playground com formulário gerado do `inputSchema`, trilha de auditoria filtrável e edição de servidores (add/remove/campos) com diff antes de gravar. Nenhuma linha de API nova é necessária para isso no modo local.

![Overview](./overview.png)

---

## 2. Por que agora

O posicionamento do projeto é explícito: um proxy central que segura credenciais, aplica ACL e registra tudo, para times que colocam agentes de IA chamando ferramentas ([`docs/why-mcp-over-cli-for-teams.md`](../why-mcp-over-cli-for-teams.md)). Esse posicionamento cria três perguntas que a CLI responde mal:

| Pergunta | Hoje | Com UI |
|---|---|---|
| "Quais backends estão de pé e quais tools eles publicam?" | `mcp --list`, `mcp <server> --list`, um comando por servidor | Uma tela, com estado de conexão, cache e classificação |
| "Por que a chamada do fulano foi negada?" | `mcp logs --errors` + ler o JSON, ou `mcp acl check` adivinhando o caso | Trilha com filtro, decisão de ACL e regra que casou, lado a lado |
| "Que argumentos esse tool aceita?" | `mcp sentry --info` e escrever o JSON à mão | Formulário derivado do schema, com dry-run do JSON-RPC |

Há também um custo de adoção: experimentar o projeto hoje exige saber a sintaxe `mcp <server> <tool> '{...}'`. Um playground reduz isso a escolher um item numa lista.

E há um argumento de produto: as métricas que o projeto usa para se vender ("um backend = um processo, 5 clientes, 3 GB economizados") **não são visíveis em lugar nenhum**. `/health` devolve contadores soltos; ninguém consegue mostrar "9 backends, 5 clientes, 4 processos" numa reunião sem `curl | jq`.

---

## 3. O que já existe (inventário)

Tudo abaixo foi lido do código. É a matéria-prima da UI — e também o limite dela.

| Superfície | Estado real | Onde |
|---|---|---|
| `servers.json` | JSON com **comentários aceitos na leitura**; `${VAR}` substituído **textualmente** antes do parse | `src/config.rs:339-392`, `src/config.rs:421-426` |
| Escrita de config | Só `add`/`remove`/`update`; regravam o arquivo inteiro, sem comentários, com chaves reordenadas, **sem escrita atômica** | `src/manager.rs:304-324` |
| Auth de cliente | `auth.json` (schema v1, por issuer/URL); OAuth acontece no 401, não há comando `mcp auth` | `src/auth/store.rs:20-30`, `src/auth/store.rs:132-196` |
| ACL | `serverAuth.acl`, RBAC ou legacy; **só o proxy HTTP aplica** | `docs/reference/config-file.md:469-597` |
| Auditoria | ChronDB, chave `audit:{millis}-{uuid}`, 16 campos, **sem retenção, sem paginação**; follow = polling de 1s | `src/audit.rs:176-201`, `src/audit.rs:468-488`, `src/cli/logs.rs:53-72` |
| Cache de tools | `cache:tools:{alias}` + hash SHA-256 da config do backend; sem TTL | `src/cache.rs:8-32`, `src/config.rs:434-446` |
| Classificação | `tool-classification.json` (read/write/ambiguous, com confiança e fonte) | `src/classifier_cache.rs:32-39` |
| Instâncias | `instances/<pid>.json` + `flock` como prova de vida; `mcp reload` manda SIGHUP | `src/instances.rs:38-52`, `src/instances.rs:212-267` |
| HTTP do proxy | `/health`, `/mcp`, `/mcp/sse`, `POST /reload` (só com `adminRoles`, senão 404) | `src/serve/http.rs:512-516` |
| OAuth AS | `/.well-known/oauth-protected-resource`, `/.well-known/oauth-authorization-server`, `/register`, `/authorize`, `/token` | `src/server_auth/oauth_as/mod.rs:61-74` |
| Métricas | OTLP apenas (`mcp.proxy.requests`, `…request.duration`, gauges). **Não existe `/metrics` nem Prometheus** | `src/telemetry.rs:78-127` |
| Registry | `registry.modelcontextprotocol.io/v0.1/servers`, `limit=20`, sem cache | `src/registry.rs:4`, `src/registry.rs:56-63` |
| CORS | **não existe** — nenhum `CorsLayer`; UI em outra origem é bloqueada pelo navegador | `src/serve/http.rs:512-547` |
| Notificações | descartadas (`202` com corpo `null`); nenhum `notifications/*/list_changed` | `src/serve/http.rs:845-848` |
| Sessão MCP | stateless; o único estado é o mapa SSE legado, **não vinculado à identidade** | `src/serve/http.rs:33`, `src/serve/http.rs:912-947` |
| `/health` | anônimo por design; `tools` é o catálogo inteiro, não filtrado por ACL | `src/serve/http.rs:663-680` |
| Erro 401 | sem `WWW-Authenticate` — cliente não descobre o AS pelo padrão MCP | `src/serve/http.rs:64-102` |

### O que falta para uma UI — e o tamanho de cada buraco

| # | Lacuna | Consequência | Correção proposta |
|---|---|---|---|
| 1 | Nenhuma API de gestão (só `/health` e `/reload`) | UI não consegue listar servidores, tools, tokens ou auditoria por HTTP | `/api/v1` (seção 6) |
| 2 | Nenhum caminho de leitura/escrita que preserve o arquivo | Salvar pela UI apagaria comentários e reordenaria chaves | Módulo `config::source` preservando bytes, comentários e `${VAR}` |
| 3 | Auditoria carrega tudo e filtra em memória | Tela de atividade fica lenta com semanas de log | Paginação por cursor + prefixo temporal |
| 4 | Follow é polling de 1s | Feed "ao vivo" com latência e carga desnecessária | `broadcast` in-process + SSE |
| 5 | Sem paginação, retenção ou rotação | Banco cresce para sempre; UI não pode prometer histórico | Retenção configurável (política, não só UI) |
| 6 | Sem estado de credencial (expiração, escopo) | O caso "401 no backend X" é o mais comum e o mais opaco | `/api/v1/auth/upstream` + re-auth guiado |
| 7 | Sem API de instâncias, cache ou classificação | Overview não mostra o que está de pé | `/api/v1/instances`, `/cache` |
| 8 | Sem timeout por servidor | UI não expõe o que não existe | Fora de escopo (documentar) |
| 9 | Sem CORS e sem API same-origin | Uma SPA em `localhost:3000` não fala com `127.0.0.1:8080` | Servir a UI pela mesma origem (decisão de arquitetura, não gambiarra de CORS) |
| 10 | Reload de `oauthAs` deixa emissor e validador dessincronizados | Token recém-emitido é rejeitado; `requires_restart` não avisa | Tratar `oauthAs` como restart-only e bloquear o apply |

---

## 4. Princípios

Herdados dos "design principles" do projeto (`docs/reference/architecture.md:225-231`), traduzidos para interface:

1. **Um binário.** A UI é servida por `mcp ui`/`mcp serve`, com assets embutidos. Nada de `docker compose up` para ver a própria config.
2. **Local-first.** Sem conta, sem telemetria, sem CDN, sem fonte remota. Funciona offline.
3. **O arquivo é a fonte da verdade.** A UI lê, mostra o diff, escreve — e esquece. Nada de estado de config no servidor.
4. **Leitura é livre, escrita é um evento.** Escrever exige identidade admin, produz uma entrada de auditoria e um backup.
5. **Segredo não entra no DOM.** Nem token, nem header, nem argumento marcado como sensível. Nem "revelar por 5 segundos".
6. **A UI nunca mente sobre autoridade.** Se a ACL não vale para o seu shell, a tela diz isso; se o reload só sinaliza (sem ACK), a tela mostra "sinal enviado", não "aplicado".

---

## 5. Arquitetura

```
┌ Modo local: mcp ui ──────────────────────────────────────────────┐
│                                                                  │
│  browser ──HTTP──▶ axum (loopback, token de sessão)              │
│                     ├── /api/v1/*      lê config/DB/instâncias   │
│                     ├── /assets/*      SPA embutida (rust-embed) │
│                     └── /api/v1/events  SSE (audit + estado)     │
│                                                                  │
│  in-process: ConfigSource ─▶ McpClient ─▶ backends               │
│              AuditStore  ─▶ ChronDB                              │
│              InstanceRegistry ─▶ <config_dir>/instances          │
└──────────────────────────────────────────────────────────────────┘

┌ Modo time: mcp serve --http :7332 --management ──────────────────┐
│                                                                  │
│  browser ──HTTPS──▶ proxy (auth: bearer / forwarded / oauth_as)  │
│                       ├── /mcp, /mcp/sse      (MCP, como hoje)   │
│                       ├── /health             (como hoje)        │
│                       ├── /reload             (adminRoles)       │
│                       └── /api/v1/*           (novo, adminRoles  │
│                                                para escrita;     │
│                                                catálogo filtrado │
│                                                por identidade)   │
└──────────────────────────────────────────────────────────────────┘
```

**Decisão: dentro do binário, não num serviço separado.**

| Opção | Prós | Contras | Veredito |
|---|---|---|---|
| **A. `mcp ui` no mesmo binário** (recomendada) | Sem versão descolada; reusa config/auth/ACL/cache/instances; instala junto; cabe no Docker que já existe | Adiciona peso de build (frontend) ao repo Rust | ✅ |
| B. Endpoints novos só no `serve`, SPA separada | Separação de repositórios | Duas versões, dois deploys, ACL reimplementada ou mal reusada | ❌ |
| C. App desktop (Tauri) | Aparência nativa | Terceira toolchain, sem acesso prático a servidor remoto | ❌ |

O modo remoto não é um segundo produto: é o **mesmo** bundle falando com uma API que, no modo local, é servida pelo mesmo processo. Isso mantém a superfície de código pequena — um router, duas implementações de *backend* de dados (local e remoto).

Servir os assets pela mesma origem não é preferência estética: **não existe CORS no servidor** (`src/serve/http.rs:512-547`), então uma SPA rodando em outra porta seria bloqueada pelo navegador no primeiro `fetch`. A alternativa — adicionar `CorsLayer` com `allow_origin` configurável — abriria a API de administração para qualquer origem que o operador colar ali. Same-origin elimina a classe inteira de problema.

### Onde a UI lê o quê

| Dado | Local (`mcp ui`) | Remoto (`serve --management`) |
|---|---|---|
| Config | arquivo direto (com `${VAR}` literal) | API lê o arquivo do host do proxy |
| Tools/catálogo | cache `cache:tools:*` + descoberta sob demanda | cache em memória do proxy |
| Auditoria | ChronDB local | ChronDB do proxy |
| Instâncias | diretório `instances/` | o próprio processo + `/api/v1/instances` |
| Credenciais upstream | `auth.json` (metadados; nunca o segredo) | `auth.json` do proxy |
| Chamar tool | `McpClient` in-process | `POST /mcp` |

---

## 6. API `/api/v1`

Convenções: JSON, `snake_case`, erros como `{"error": {"code", "message", "hint"?}}`; toda escrita exige `If-Match` com o ETag do recurso; toda escrita gera auditoria com `source: "ui"`.

### 6.1 Leitura

| Método | Rota | Retorno | Estado |
|---|---|---|---|
| GET | `/health` | contadores atuais | **existe** (`src/serve/http.rs:663-680`) |
| GET | `/api/v1/meta` | versão, modo (local/remoto), caminhos, read-only?, capabilities | novo |
| GET | `/api/v1/servers` | lista com tipo, estado, tools, último uso, política de idle, saúde da credencial | novo |
| GET | `/api/v1/servers/{name}` | detalhe + config redigida + hash | novo |
| GET | `/api/v1/servers/{name}/tools` | tools do cache, com classificação (`read`/`write`/`ambiguous`, fonte, confiança) | novo |
| GET | `/api/v1/tools` | catálogo namespaced agregado, paginado e filtrável | novo |
| GET | `/api/v1/audit` | `?server=&tool=&identity=&method=&errors=&since=&cursor=&limit=` | novo |
| GET | `/api/v1/events` | SSE tipado: `audit`, `state` (backends/clientes), `reload` | novo |
| GET | `/api/v1/acl` | política + matriz efetiva (role × servidor) | novo |
| GET | `/api/v1/acl/check?subject=&server=&tool=` | veredito + cadeia de grants avaliada | novo (reusa `cli/acl.rs`) |
| GET | `/api/v1/auth/upstream` | por servidor: tipo, expira em, escopo — **sem segredo** | novo |
| GET | `/api/v1/auth/clients` | registros OAuth (issuer, client_id, criado em) | novo |
| GET | `/api/v1/instances` | pid, transporte, addr, versão, uptime, filtro `--only/--exclude` | novo |
| GET | `/api/v1/cache` | por backend: `cached_at`, nº de tools, hash ok/stale | novo |

### 6.2 Escrita (exige admin + `If-Match`)

| Método | Rota | Semântica |
|---|---|---|
| POST | `/api/v1/config/validate` | parse + schema; devolve erros por caminho JSON |
| POST | `/api/v1/config/plan` | recebe patch, devolve diff unificado + impacto (tools que somem, backends que exigem restart) |
| PUT | `/api/v1/config` | aplica patch atômico (tmp + fsync + rename, 0600, backup `.bak.<ts>`) |
| POST | `/api/v1/reload` | publica (SIGHUP local / atualização em processo); alias do `POST /reload` que já existe |
| POST | `/api/v1/servers/{name}/connect` | força descoberta agora (equivalente a `--health`) |
| POST | `/api/v1/servers/{name}/reauth` | inicia OAuth e devolve a URL do browser |
| POST | `/api/v1/cache/purge` | limpa cache de tools/classificação de um backend ou tudo |
| PUT | `/api/v1/acl` | substitui `serverAuth.acl` (mesmo caminho de plan/apply) |
| POST | `/api/v1/auth/tokens` · DELETE `/api/v1/auth/tokens/{id}` | cria/revoga bearer token (segredo só na resposta de criação) |
| POST | `/api/v1/instances/{pid}/reload` | SIGHUP seletivo (reusa `instances::signal_reload`) |

### 6.3 O protocolo de escrita (o pedaço que não pode dar errado)

O arquivo tem comentários, `${VAR}` literais e ordem significativa para humanos — e o writer atual destrói os três (`src/manager.rs:304-324`). A UI **não pode** reusar esse caminho. Proposta:

```
1. read  → bytes crus + mtime + sha256            (nunca load_config: ele expande ${VAR})
2. parse → Value preservando comentários (json5-like já suportado na leitura)
3. patch → edição cirúrgica no Value (só o nó tocado; desconhecidos preservados)
4. diff  → unificado, contra os bytes originais
5. valid → parse tipado com as mesmas regras de config.rs + avisos (nomes reservados,
           tool_acl conflitante, provider sem subconfig, adminRoles vazio)
6. write → tmp no mesmo diretório → fsync → rename → backup → 0600
7. apply → reload seletivo; resposta por instância
```

Critérios de aceite (testáveis, sem browser):

- round-trip de um arquivo com comentários e `${VAR}` não altera um byte;
- aplicar um patch que só muda `idle_timeout` de um servidor toca apenas aquela linha;
- arquivo inválido nunca é escrito (validação antes do rename);
- escrita concorrente: se o `sha256` mudou desde a leitura, `409` com o diff novo;
- toda escrita aparece em `mcp logs` com `source: "ui"` e o diff resumido.

### 6.4 Restrições que a UI precisa respeitar (e mostrar)

Estas não são bugs a corrigir na UI: são fatos do servidor. A interface deve refleti-los, não escondê-los.

| Fato | Efeito na UI |
|---|---|
| ACL só vale no proxy HTTP; `mcp <server> <tool>` e o modo local ignoram | Selo visível: "ACL não se aplica a esta sessão" no modo local |
| `adminRoles` compara **exatamente**, sem `*`, e anônimo nunca passa | Se o token não tem roles, mostrar "seu token não tem role admin" em vez de 403 genérico |
| Token bearer na forma legada (`{"tok": "subject"}`) nunca administra | A tela de tokens deve exigir a forma `{subject, roles}` ao criar |
| `oauthAs` alterado + reload = emissor e validador dessincronizados | `plan` marca a mudança como `requires_restart` (o servidor não marca) |
| `MCP_SERVERS_CONFIG` (config inline) não tem arquivo canônico | Entrar em **read-only** e explicar por quê |
| Argumentos só existem no audit quando `log_arguments=true` (e **nunca** no proxy) | Coluna "argumentos" vazia com explicação, sem parecer bug |
| `tool_name` é namespaced no proxy (`sentry__x`) e cru na CLI (`x`); timestamp é UTC no proxy e local na CLI | Normalizar na apresentação e guardar os dois campos |
| `acl check --subject` lê roles só do mapa `subjects`, não dos tokens | O simulador deve oferecer "identidade do token" além de "sujeito do ACL" |
| Sem push de notificações; SSE legado não é vinculado à identidade | UI usa SSE **próprio** para eventos; para MCP, `POST /mcp` stateless |
| Corpo limitado a ~2 MB (default do axum), sem rate limit | Resultados grandes precisam ser truncados com "abrir bruto", não estourar |

---

## 7. Experiência

Sete telas. Cada uma responde a uma pergunta, não a uma entidade do banco.

| Tela | Pergunta que responde | Dados |
|---|---|---|
| **Overview** | "Está tudo de pé? O que está consumindo o quê?" | `/health`, `/servers`, `/instances`, últimas negações |
| **Servers** | "Como este backend está configurado e o que ele publica?" | config redigida, tools, classificação, credencial |
| **Editar servidor** | "O que exatamente vai mudar no arquivo?" | formulário, diff, impacto, backup |
| **Playground** | "Como eu chamo este tool?" | `inputSchema`, dry-run, resultado, execuções recentes |
| **Activity** | "O que aconteceu, para quem, e por quê foi negado?" | audit + decisão de ACL + classificação |
| **Access** | "Quem pode o quê, e por quê?" | ACL, matriz, simulador, tokens, clientes OAuth |
| **Settings** | "Onde ficam as coisas e o que está ligado?" | caminhos, auditoria, telemetria, instâncias |

### Overview

Stat cards (backends conectados/total, tools publicadas, clientes ativos, chamadas 24h), tabela de backends com estado e política de idle, throughput, últimas negações, revisão de protocolo negociada. O banner de topo é para o único caso que exige ação: backend com 401, config alterada no disco, instância desatualizada.

O número que o projeto vende precisa aparecer: **"9 backends · 5 clientes · 4 processos"** com um link explicando a conta. É o argumento de economia de RAM virando um gráfico.

### Servers

Lista à esquerda, detalhe à direita. Abas: catálogo (tools + classificação), config efetiva (redigida), ACL efetiva para aquele backend, histórico. O hash da config e o "modificado há X" existem para explicar por que o cache foi invalidado.

![Servers](./servers.png)

### Playground

Formulário gerado do `inputSchema` (string → input/textarea, enum → select, `required` marcado), alternância Form/JSON, dry-run mostrando exatamente o JSON-RPC que a CLI enviaria (`mcp … --dry-run`), resultado com tipo de conteúdo (texto, imagem, JSON), e o veredito de ACL **para a identidade atual**.

Não é um chat. Não chama LLM. É um cliente MCP com formulário.

![Playground](./playground.png)

### Activity

Tabela densa, filtros por servidor/identidade/tempo/erro/negado, SSE "ao vivo", linha expandível com argumentos (quando `log_arguments`), decisão de ACL, regra que casou, classificação e confiança. Export JSONL. A coluna "ACL" mostra `allow`/`deny`, e o detalhe mostra **por que** — que é o que falta hoje no `mcp logs`.

![Activity](./activity.png)

### Access

Roles com seus grants, matriz efetiva, sujeitos, simulador (mesma avaliação de `mcp acl check`), tokens bearer (segredo só na criação), clientes OAuth registrados por DCR. O simulador é a peça central: hoje "por que negou?" exige rodar a CLI com três flags e interpretar saída de tabela.

![Access](./access.png)

### Editar servidor

O coração da decisão "editável": o formulário nunca grava direto. Ele produz um patch, o patch vira um diff contra os bytes atuais, o painel de impacto diz quantas instâncias recebem SIGHUP, o que sai do cache e se exige restart. Aplicar só é possível com o arquivo ainda no mesmo hash.

![Editar servidor](./edit.png)

### Settings

Caminhos (config, auth, auth_server, db, instances), estado da auditoria, telemetria OTLP, instâncias com `reload` individual, versão. Serve também como página de "por que a UI não me deixa editar?" (modo inline, arquivo somente-leitura, sessão sem admin).

![Settings](./settings.png)

### Detalhes de interação

- **⌘K** abre a paleta: pula para servidor, tool ou tela (protótipo tem funcionando).
- **⌘↵** roda a tool no playground.
- Tabela com densidade confortável/compacta; foco visível; tudo navegável por teclado.
- Estados vazios úteis: "nenhum backend configurado → adicionar do registry", "auditoria desligada → como ligar", "sem permissão → pedir admin".
- Erros sempre com causa e próximo passo: `401` → botão re-autenticar; `timeout` → link para a trilha com a duração.

---

## 8. Fluxos críticos

### 8.1 Adicionar um servidor do registry

```
Registry (busca) → prévia da config gerada (npx/uvx/docker) → campos de env
  → plano: diff "adiciona 1 servidor" → validar → aplicar → reload
  → se env faltando: badge "configurado, credencial pendente"
```
A UI deve gerar exatamente o mesmo bloco que `mcp add` gera (`src/manager.rs:343-413`), inclusive `${VAR}` — e nunca pedir para colar o segredo no arquivo.

### 8.2 Diagnosticar um 401

```
Overview (banner) → Servers/{x} → "credencial expirada" → Re-autenticar
  → OAuth no browser (fluxo já existe) → token salvo em auth.json
  → reconectar backend → tools voltam ao catálogo
```
Hoje esse caminho é: ver o erro em `mcp logs`, descobrir que é auth, rodar `mcp <server> --health`, torcer.

### 8.3 Investigar uma negação

```
Activity (filtro deny) → linha → painel: regra avaliada, classificação, confiança
  → "testar como…" → abre Access com sujeito/tool preenchidos → veredito
  → se for política errada: editar grant → plano → aplicar
```

### 8.4 Alterar config com segurança

Ver §6.3. A UI nunca escreve direto: `plan` (diff + impacto) → confirmar → `apply` → `reload` por instância → confirmação com resultado (`added/changed/removed/requires_restart`).

---

## 9. Segurança

Um servidor HTTP local é uma superfície real: qualquer processo do usuário alcança `127.0.0.1`, e qualquer página aberta no navegador tenta. O modelo:

| Ameaça | Mitigação |
|---|---|
| DNS rebinding (site malicioso falando com `localhost`) | `Host` só aceita `127.0.0.1`, `localhost`, `[::1]`; `Sec-Fetch-Site: cross-site` rejeitado; **sem CORS** |
| CSRF | Cookie `HttpOnly; SameSite=Strict; Path=/` + header `X-MCP-UI: 1` obrigatório em métodos não-GET + `Origin` conferido |
| Sessão de outro usuário na máquina | Bind `127.0.0.1`; token de 32 bytes por sessão, injetado na URL de abertura e trocado por cookie; TTL; botão "encerrar sessão" |
| XSS via saída de tool | Nada de `innerHTML` com dado de backend; JSON renderizado como texto com syntax highlight; markdown sanitizado; CSP `default-src 'self'; script-src 'self'` |
| Vazamento de segredo | Config sempre redigida; `auth.json` só metadados; token bearer só na resposta de criação; `log_arguments=false` respeitado; nunca logar corpo de credencial |
| Escrita indevida | Admin obrigatório; `If-Match`; backup; auditoria de toda escrita com diff resumido |
| UI como bypass de ACL | Modo local declara "ACL não se aplica ao shell"; modo remoto **filtra o catálogo por identidade** (mesma ACL) e exige `adminRoles` para escrita |
| Dependência de CDN / supply chain | Zero recurso externo; assets embutidos; CSP sem `unsafe-inline` |
| Sessão remota em HTTP claro | `--insecure` continua necessário fora de loopback (regra atual), e a UI avisa |
| `/health` anônimo revela topologia | A UI não deve depender dele para nada sensível; propor `WWW-Authenticate` e um `/health` mínimo quando há auth |
| Provider `forwarded` confia no header sem checar IP | Aviso na tela de Settings quando `providers: ["forwarded"]` está ativo |
| SSE legado tem `session_id` não vinculado à identidade | A UI não usa esse transporte; se um dia usar, é bug de segurança |
| `auth.json` é gravado sem atomicidade nem `0600` | Se a UI gerenciar tokens, ela grava com o mesmo `tmp+rename+chmod` do config |

**Fora de escopo de segurança (dito explicitamente):** a UI local não protege contra outro processo do mesmo usuário — quem lê `~/.config/mcp` já tem os tokens. Ela protege contra o navegador.

---

## 10. Frontend: htmx embutido, sem build

Decisão tomada: **HTML no servidor + htmx**, servido pelo próprio binário. Nada de Node, npm, bundler ou `dist/` para manter sincronizado. O repositório continua sendo um projeto Rust do começo ao fim.

| Peça | Escolha | Por quê |
|---|---|---|
| Templates | `minijinja` + `templates/*.html` embutidos com `include_str!` | Jinja familiar, auto-escape por padrão (`{{ }}`), sem etapa de compilação |
| Interatividade | htmx 2 (vendorizado, 51 KB) | `hx-post`/`hx-target`/`hx-swap` cobrem formulário inline, diff parcial e polling |
| Estilo | um `app.css` escrito à mão (~600 linhas) | O protótipo é o contrato visual; sem Tailwind, sem dependência |
| Estado no cliente | nenhum | A página é derivada do arquivo; recarregar é sempre seguro |

O que o htmx faz aqui — e o que **não** faz:

- `hx-post="/servers/{name}/plan"` com `hx-target="#diff"` mostra o diff sem sair da página;
- `hx-get="/activity/rows?since=…"` com `hx-trigger="every 3s"` mantém a trilha viva;
- formulários continuam funcionando sem JS (POST normal → redirect), então a UI degrada para HTML puro;
- nenhum estado de aplicação vive no navegador: não há SPA, não há store, não há hidratação.

Consequência boa: a UI inteira é testável com `GET`/`POST` e assert no HTML — sem browser no CI.

```
src/ui/
  mod.rs          # router axum, sessão, bootstrap
  session.rs      # token, cookie, Host/Origin
  pages.rs        # renderização minijinja
  handlers/       # overview, servers, playground, activity, access, settings
  templates/*.html
  assets/         # htmx.min.js (vendorizado), app.css
src/config_source.rs   # leitura/escrita preservando comentários e ${VAR}
```

`mcp ui --addr 127.0.0.1:7332 --open [--read-only]` — `--read-only` existe para quem quiser servir a UI sem habilitar escrita, e é o padrão quando a config vem inline (`MCP_SERVERS_CONFIG`).

## 11. Fases

| Fase | Entrega | Critério de aceite | Esforço |
|---|---|---|---|
| **0 — Base editável** | `mcp ui` com Overview, Servers (listar/editar/adicionar/remover), Playground, Activity, Settings; `config_source` com diff + escrita atômica; htmx embutido | `cargo install --path . && mcp ui` abre; editar `idle_timeout` mostra diff e grava sem perder comentários; playground roda uma tool real | 2–3 sem |
| **1 — Operação** | re-auth guiado, purge de cache, add do registry com busca, reload por instância, SSE de auditoria | Fluxo 8.2 e 8.4 completos; toda escrita auditada | 2 sem |
| **2 — Time** | `/api/v1` no `serve`, auth por identidade, catálogo filtrado por ACL, Access (ACL/tokens/clientes) | `serve --management` atrás de `adminRoles`; catálogo difere por identidade | 3–4 sem |
| **3 — Polimento** | retenção de auditoria, gráficos derivados do audit, coleções, links compartilháveis, `enabled` por servidor | Histórico de 30d paginado; retenção configurável | 2–3 sem |

Fase 0 é deliberadamente **sem API nova**: no modo local, o processo já tem tudo em mãos. É o menor caminho para validar a hipótese com uso real, antes de investir em superfície pública.

## 12. Não-objetivos

- **Cliente de chat / agente.** A UI não conversa com LLM, não orquestra agentes, não substitui Claude Code.
- **Gerenciador de usuários.** Identidade continua delegada ao reverse proxy (`oauth2-proxy`, Cloudflare Access).
- **Deploy/orquestração.** Não sobe Kubernetes, não gerencia containers.
- **Editor de config genérico.** Só os campos que o schema conhece; chaves desconhecidas são preservadas, não editáveis.
- **Substituto do `serve`.** O proxy continua sendo o produto; a UI é uma lente sobre ele.
- **Multi-tenant.** Uma instância, uma config, um conjunto de credenciais.

---

## 13. Riscos

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Scope creep para "dashboard de gateway" | Alta | Alto | Não-objetivos + fases com gate |
| Corromper `servers.json` | Baixa | **Crítico** | Escrita atômica, backup, validação antes do rename, testes de round-trip e de concorrência |
| Vazamento de segredo na tela | Média | Crítico | Redação no servidor (não no cliente), teste automatizado que varre o HTML/JSON por padrões de token |
| UI virar dependência (ninguém usa a CLI) | Baixa | Médio | Paridade: toda ação da UI tem comando equivalente, e a tela mostra qual |
| Concorrência ChronDB (UI + serve + CLI) | Média | Médio | Abrir o DB em modo leitura para a UI; escritas só via ações explícitas; validar comportamento com dois processos |
| Node no CI / `dist` desatualizado | Média | Baixo | Job de verificação; fallback htmx |
| Acessibilidade e densidade | Baixa | Baixo | Dark-first + contraste AA; teclado desde a fase 0 |
| Manutenção de frontend por mantenedor Rust | Média | Médio | Stack mínima (sem framework de UI pesado), poucos arquivos, sem estado global |
| UI se tranca fora ao editar `adminRoles` | Média | Alto | `plan` recusa remover a última role da identidade que está aplicando, sem confirmação explícita |
| Split-brain de `oauthAs` após reload | Média | Alto | Bloquear no `plan`: mudança em `oauthAs` = "exige restart", com o comando pronto |
| Duplicar regras de ACL na UI (e divergir do servidor) | Alta | Alto | A UI **chama** o avaliador existente (`acl::eval`), nunca reimplementa |

---

## 14. Decisões (tomadas e em aberto)

Tomadas:

1. **`mcp ui` no binário principal.** Sem crate separado — a UI usa `config`, `audit`, `instances` e `client` diretamente.
2. **htmx embutido, sem build.** O custo de um formulário schema-driven é aceitável; o de um pipeline Node em um projeto Rust não.
3. **Modo local editável por padrão.** `--read-only` existe como opt-in e como padrão quando a config é inline.

Em aberto:

4. **Sessão:** cookie em memória (some ao reiniciar) ou persistida com "confiar neste navegador"?
5. **A UI entra na imagem Docker `latest`?** (~50 KB extras; recomendaria sim, com `MCP_UI_ADDR` explícito.)
6. **Idioma da interface:** inglês (consistente com docs e CLI) — confirmar.
7. **Retenção de auditoria:** pré-requisito da fase 3 ou feature independente?

## 15. Apêndice — PRs

1. **docs:** esta proposta + o protótipo (nenhum código de produção).
2. **`config_source`:** leitura com comentários, `${VAR}` literal, patch cirúrgico por span, escrita atômica, backup, `409` por hash. Só Rust, testável, sem UI. **Maior risco, primeiro.**
3. **`mcp ui` fase 0:** router, sessão, templates, htmx vendorizado, telas e edição em cima do PR 2.
4. **fase 1:** SSE de auditoria, re-auth, purge de cache, registry.
5. **fase 2:** `/api/v1` no `serve` + `--management`.

O protótipo em [`prototype.html`](./prototype.html) é autocontido (abre no navegador, sem build) e serve como contrato visual: cores, densidade, componentes e estados estão ali para serem copiados, não reinterpretados. A implementação difere só na mecânica (formulários e partials em vez de JS de cliente).
