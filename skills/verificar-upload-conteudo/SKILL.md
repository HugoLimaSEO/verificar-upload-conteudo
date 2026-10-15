---
name: verificar-upload-conteudo
description: >-
  Verifica, numa tarefa de "upload de conteúdo" da liveSEO (app.liveseo.com.br),
  se TODOS os conteúdos produzidos já foram publicados — partindo da lista de
  "Produção" (também chamada "Controle de produção") da tarefa e checando a etapa
  atual de cada conteúdo, seja categoria (page) ou blogpost (blog). A LEITURA DA
  ETAPA É FEITA PELA earth-cli (`earth content list` / `earth content get`),
  lendo os campos `new_step` e `current_status` — a CLI bate no endpoint certo do
  app e traz o status atual; o MCP de conteúdo NÃO é usado porque devolve o campo
  `step` legado (errado) e erra em boa parte das páginas. Devolve um quadro
  publicado / cancelado / não publicado. Ao final, oferece publicar um COMENTÁRIO
  PÚBLICO na tarefa dizendo que ela pode ser concluída, ou apontando qual conteúdo
  ainda não foi publicado. Use SEMPRE que o usuário colar um link de tarefa do app
  (…/projeto/CB###/task/<id>/<id>) e perguntar se os conteúdos "foram publicados",
  "foram upados", "estão no ar", "todos subiram", ou pedir para "verificar o
  upload", "conferir a publicação das categorias/blogs", "checar se falta subir
  algum conteúdo". Dispare também quando descreverem uma tarefa de
  produção/revisão/upload de categorias ou blogposts e quiserem saber o status de
  publicação. A skill VERIFICA e relata; quando a confirmação ao vivo mostra que um
  conteúdo em "Publicar" já está no ar, ela MARCA esse conteúdo como "Publicado" no
  módulo de conteúdo (new_step=PUBLISHED). Quando TODOS os conteúdos estão
  publicados e/ou cancelados (zero pendências), ela move a tarefa automaticamente
  para "Para revisão" e registra na tarefa uma NOTA INTERNA dizendo que a validação
  foi feita e quais conteúdos estão publicados. O comentário PÚBLICO (visível ao
  cliente) continua exigindo confirmação explícita do usuário.
metadata:
  type: workflow
  source: >-
    Etapa atual de cada conteúdo é lida pela earth-cli (`earth content list` em
    lote, ou `earth content get` unitário), campos `new_step` + `current_status`.
    Marcar como Publicado usa a mesma rota do app (PATCH
    /content/{post|page}/{projectId}/{id}/save-changes, new_step=PUBLISHED) com a
    credencial da earth-cli. Tarefa/lista de produção, comentário/nota interna e
    movimento de kanban também via earth-cli (fallback MCP liveseo-earth). NÃO usa
    o data warehouse (gestão) nem o get_page_data_by_id do MCP (devolve o `step`
    legado e erra em muitas páginas).
---

# verificar-upload-conteudo

Uma tarefa de **upload de conteúdo** na liveSEO tem, na descrição da subtask, um
bloco de **Produção** (às vezes "Controle de produção") com a lista dos conteúdos
escritos. O objetivo é responder com confiança: *"todos os conteúdos produzidos já
foram publicados?"* e, com isso, comentar na tarefa se ela pode ser concluída ou o
que falta subir.

Decisões de desenho — **aprendidas na prática (CB491, CB789), não mude sem motivo**:

- **A etapa atual vem dos campos `new_step` + `current_status`, lidos pela
  `earth-cli`** — não do MCP. A CLI usa a mesma sessão da API do app e devolve o
  status atual e correto. Por que não o MCP de conteúdo:
  - O MCP `get_page_data_by_id` / `get_post_data_by_id` **erra de forma
    consistente em boa parte das páginas** (timeout/serialização), inclusive em
    páginas **publicadas** — erro de MCP não diz nada sobre o status.
  - Mesmo quando responde, o MCP devolve o campo **`step` legado**, que fica
    desatualizado (ex.: blog 47807 do CB789 aparece como `step: PLANNING`, mas o
    `new_step` real é `CLIENT_REVIEW`). O app usa `new_step`; a CLI e a skill também.
- **NÃO use o data warehouse (`earth-dw`)** — é acesso só de gestão; quebraria a
  skill para outros operadores.
- **A lista de PRODUÇÃO é a fonte da verdade sobre o que precisa estar no ar.** A
  tarefa costuma ter também um bloco **Upload** (ou "Controle de upload"), mas ele
  é **pouco confiável** (fica vazio, com "Não encontrado" ou desatualizado).
  **Não use o Upload como critério** — no máximo como contexto.
- **Conteúdo cancelado/rejeitado NÃO é pendência.** Um item em
  `current_status: REJECTED` (ou marcado como cancelado) foi descartado na revisão
  e **não precisa subir** — não conte como "falta publicar" e não impeça a
  conclusão por causa dele. (Aprendido no CB789: blog 47807 estava
  `new_step: CLIENT_REVIEW` mas `current_status: REJECTED` — cancelado, não pendente.)

## Pré-requisito
A leitura usa a **`earth-cli` autenticada**. Confirme com `earth --json auth status`
(exit 3 = não autenticado; peça `earth auth login` ou a env `EARTH_API_SESSION`).
Sempre use `--json`. **Não é mais necessário Claude in Chrome logado** para ler o
status — ele fica só como fallback opcional de confirmação ao vivo (passo 3).

## Fluxo

### 1. Puxar a tarefa e extrair a lista de PRODUÇÃO
Use a `earth-cli` (`earth --json task subtask ...`) ou o MCP `get_subtask_by_url`
para obter a subtask. O campo `description` traz HTML com os blocos e um link do
Earth para cada conteúdo.

Extraia **apenas o bloco de Produção**. Os títulos variam — aceite qualquer um que
contenha "produç" (**"Produção"**, **"Produções"**, **"Controle de produção"**).
Pare de coletar ao chegar no bloco seguinte (geralmente após um `<hr>`), que se
chama **"Upload"** ou "Controle de upload".

Para cada item capture **id e tipo**, pelo padrão do link:

- `…/projeto/CB###/page/{id}` → **categoria/página** → tipo `page`.
- `…/projeto/CB###/blog/{id}` → **blogpost** → tipo `blog`.

Uma tarefa pode misturar os dois tipos — classifique item a item. **Deduplique os
ids**. Cada id único vira uma linha do relatório. Alguns itens da lista podem não
ter id do Earth (só apontam direto para a URL no ar) — trate à parte (passo 3).

Você também precisa do **projectId numérico** (não o código CB). Resolva com
`earth --json project get CB###` (campo `id`) ou o MCP `get_projects_list`
(ex.: CB491 → 662; CB789 → 1079).

### 2. Ler `new_step` + `current_status` de TODOS os conteúdos via earth-cli (verificação principal)
**Verifique todos, sem amostragem.**

**Preferido — em lote (1 chamada por tipo):** `earth content list` traz a lista
inteira do projeto com `new_step` e `current_status`. Puxe uma vez por tipo
presente na Produção e cruze pelos ids:

```bash
earth --json --project 1079 content list blog --param limit=500
earth --json --project 1079 content list page --param limit=500   # se houver páginas
```

Indexe o resultado por `id` e, para cada id da Produção, leia `new_step` e
`current_status`. (A listagem aceita filtros: `--param new_step=...`,
`--param current_status=...`, `--param canceled=...`.)

**Unitário (fallback / id ausente na lista):**

```bash
earth --json --project 1079 content get blog 47807
earth --json --project 1079 content get page 33537
```

Campos que importam: `new_step`, `current_status` (e `name`, `url`).

**Classificação de cada item (4 estados):**

| Situação | Regra | Conta como |
|----------|-------|-----------|
| **Publicado (no ar)** | `new_step` = `PUBLISHED` | ✅ está no site |
| **A publicar (upload não confirmado)** | `new_step` = `PUBLISH` | ⏳ liberado pra subir — **CONFIRMAR ao vivo** (passo 3) |
| **Cancelado/rejeitado** | `current_status` = `REJECTED` | 🚫 não precisa subir |
| **Pendente** | `new_step` anterior a PUBLISH (PLANNING…CLIENT_REVIEW), sem ser cancelado | ❌ falta produzir/revisar/publicar |

Erra para o lado de sinalizar pendência: `new_step` desconhecido e **não**
cancelado = pendente.

**A diferença Publicar × Publicado (confirmado no histórico, CB789):**
`PUBLISH` ("Publicar") = conteúdo **aprovado e liberado para ir ao ar** — é o
handoff pro time técnico, entra na fila de upload; **não garante que já está no
site**. `PUBLISHED` ("Publicado") = o técnico **subiu e confirmou** — aí sim está
no ar. O upload é exatamente o passo que leva de `PUBLISH` → `PUBLISHED` (no CB789,
14 dias entre um e outro no post 47691).

Por isso: **só `PUBLISHED` conta como "no ar" com certeza.** `PUBLISH` é o único
estado ambíguo — em algumas squads o item fica parado em `PUBLISH` mesmo já no ar
(CB491), em outras `PUBLISH` é só transitório e o que está no ar vai pra
`PUBLISHED` (CB789). **Todo item em `PUBLISH` deve ser confirmado ao vivo** (passo
3): no ar → conta como publicado; fora do ar/404 → é upload pendente.

**Enum oficial (`new_step`)**, na ordem da barra de etapas do app:

| # | Etapa (app) | `new_step` | Publicado? |
|---|-------------|-----------|-----------|
| 1 | Planejamento | `PLANNING` | ❌ não |
| 2 | Aprovação de pauta | `SCHEDULE`* | ❌ não |
| 3 | Backlog | `BACKLOG`* | ❌ não |
| 4 | Produção | `PRODUCTION` | ❌ não |
| 5 | Revisão | `REVIEW` / `REVIEWED` | ❌ não |
| 6 | Revisão cliente | `CLIENT_REVIEW` | ❌ não |
| 7 | **Publicar** | `PUBLISH` | ⏳ liberado — **confirmar ao vivo** |
| 8 | **Publicado** | `PUBLISHED` | ✅ **no ar** |

\* rótulos internos de "Aprovação de pauta"/"Backlog" variam por projeto (já visto
`APPROVAL`, `SCHEDULE`, `BACKLOG`) — como só `PUBLISHED` conta como no ar e
`PUBLISH` sempre é confirmado ao vivo, o resto (qualquer etapa ≤ CLIENT_REVIEW) já
cai em "pendente" independentemente do rótulo exato.

`current_status` observado: `WAITING` (normal, itens no ar ficam assim),
`REJECTED` (cancelado/rejeitado). O campo `status` (0/false/null) é **inconsistente
entre endpoints** — não confie nele; use `current_status` para detectar cancelado.

> **Não use `get_page_data_by_id` / `get_post_data_by_id` (MCP) como leitor de
> status.** Erram em muitas páginas e, quando respondem, trazem o `step` legado
> (errado). Use a `earth-cli`.

### 3. Confirmar ao vivo — itens em `PUBLISH`, sem id, ou em dúvida
Confirme abrindo a URL no ar (Claude in Chrome ou navegação) e comparando o
**título produzido** com o `document.title`/`<h1>` da página. Faça isso para:

- **Todo item em `new_step: PUBLISH`** — é "liberado pra publicar", não garante que
  já subiu; a confirmação ao vivo decide se conta como publicado ou upload pendente.
- Item **sem id do Earth** (a Produção só linka a URL final).
- Qualquer `new_step` que pareça inconsistente com o contexto.

Resultado:
- Títulos **batem** e sem 404 → publicado. ✅
- Título **genérico/diferente** ou **404** → **upload pendente**. ⚠️

Cuidado: **alguns domínios de loja bloqueiam/atrasam** o navegador (timeout); se
não abrir, registre que a confirmação ao vivo não foi possível e mantenha o item
como **a confirmar** (não arredonde para publicado).

Itens em `PUBLISHED` **não** precisam de confirmação ao vivo — já estão no site.

### 3b. Marcar como Publicado no módulo de conteúdo (automático)
Todo item em `PUBLISH` que a confirmação ao vivo mostrou **no ar** deve ser
**marcado como "Publicado"** no Earth (`new_step` → `PUBLISHED`) — senão o módulo
de conteúdo continua em "Publicar" mesmo com a página no site. **Faça sem pedir
confirmação** (é o registro do que já foi verificado). **Só marque o que foi
confirmado no ar**: fora do ar, 404, título divergente ou "a confirmar" (site não
abriu) **não** são marcados.

A earth-cli não tem comando para mudar a etapa; o script abaixo usa a **mesma rota
do app** (`PATCH /content/{post|page}/{projectId}/{id}/save-changes` com
`new_step: PUBLISHED`) e a credencial da earth-cli. Ele só mexe em item que está em
`PUBLISH` (ignora cancelado, já publicado e etapas anteriores — não pula etapas) e
preenche o que o app exige para "Publicado" quando estiver vazio: **data de
publicação** (= hoje) e **responsável pela publicação** (= o operador logado). Sem
mês de referência ele não grava e avisa (`bloqueado_sem_mes_de_referencia`).

Salve no scratchpad como `marcar_publicado.py` e rode **uma vez por tipo**, só com
os ids confirmados no ar (rode com `--dry-run` antes se quiser ver o corpo):

```bash
python marcar_publicado.py --project 1079 --type blog --ids 52918,52952
python marcar_publicado.py --project 1079 --type page --ids 34762
```

```python
"""Marca conteúdos como Publicado (new_step=PUBLISHED) no módulo de conteúdo do Earth.

Uso: python marcar_publicado.py --project 1079 --type blog --ids 47807,47808 [--dry-run]
Só mexe em item que está em PUBLISH ("Publicar"); ignora cancelado, já publicado e etapas anteriores.
Usa a credencial da earth-cli (mesma rota do app: PATCH /content/{post|page}/{projectId}/{id}/save-changes).
"""
import argparse, datetime, json, os, subprocess, sys

try:
    from earth_cli import manifest
    from earth_cli.utils.http import EarthClient
    from earth_cli.utils.tenant import get_base_url
except ImportError:  # roda de novo com o python do venv da earth-cli (pipx)
    for p in ("~/pipx/venvs/earth-cli/Scripts/python.exe", "~/.local/pipx/venvs/earth-cli/Scripts/python.exe",
              "~/.local/pipx/venvs/earth-cli/bin/python", "~/.local/share/pipx/venvs/earth-cli/bin/python"):
        py = os.path.expanduser(p)
        if os.path.exists(py) and os.path.abspath(py) != os.path.abspath(sys.executable):
            sys.exit(subprocess.run([py, *sys.argv]).returncode)
    sys.exit("earth-cli não encontrada — instale/autentique a earth-cli (pipx) antes.")

ap = argparse.ArgumentParser()
ap.add_argument("--project", required=True, help="projectId numérico (ex.: CB789 -> 1079)")
ap.add_argument("--type", required=True, choices=["blog", "post", "page"])
ap.add_argument("--ids", required=True, help="ids separados por vírgula")
ap.add_argument("--dry-run", action="store_true")
a = ap.parse_args()
sys.stdout.reconfigure(encoding="utf-8")

company, server = "liveseo", "prod"
try:
    import tomllib
    with open(os.path.expanduser("~/.earth/config.toml"), "rb") as f:
        cfg = tomllib.load(f).get("default", {})
    company, server = cfg.get("company", company), cfg.get("server", server)
except Exception:
    pass

c = EarthClient(get_base_url(company, server), company, server, manifest.load())
c._session()
eu = c.raw_get("/auth/me").json()["data"]["id"]
H = {"project": str(a.project)}
tipo_get = "page" if a.type == "page" else "blog"
tipo_save = "page" if a.type == "page" else "post"
hoje = datetime.date.today().isoformat()


def ler(cid):
    r = c._client.get(f"/content/{tipo_get}/{cid}", headers=H)
    return r.json() if r.status_code == 200 else {"_http": r.status_code, "_msg": r.text[:200]}


saida = []
for cid in [int(x) for x in a.ids.split(",") if x.strip()]:
    d = ler(cid)
    if "_http" in d:
        saida.append({"id": cid, "resultado": "erro_leitura", **d})
        continue
    base = {"id": cid, "nome": d.get("name"), "etapa_antes": d.get("new_step")}
    if d.get("current_status") == "REJECTED":
        saida.append({**base, "resultado": "ignorado_cancelado"})
        continue
    if d.get("new_step") == "PUBLISHED":
        saida.append({**base, "resultado": "ja_publicado"})
        continue
    if d.get("new_step") != "PUBLISH":
        saida.append({**base, "resultado": "ignorado_fora_de_publicar"})
        continue
    if not d.get("reference_date"):
        saida.append({**base, "resultado": "bloqueado_sem_mes_de_referencia"})
        continue
    corpo = {"new_step": "PUBLISHED"}
    if not d.get("publishDate"):
        corpo["publishDate"] = f"{hoje}T00:00:00.000Z"
    if not d.get("publisher_id"):
        corpo["publisher_id"] = eu
    corpo["changedAt"] = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    if a.dry_run:
        saida.append({**base, "resultado": "dry_run", "corpo": corpo})
        continue
    r = c._client.patch(f"/content/{tipo_save}/{a.project}/{cid}/save-changes", json=corpo, headers=H)
    if r.status_code >= 300:
        saida.append({**base, "resultado": "erro_gravacao", "_http": r.status_code, "_msg": r.text[:200]})
        continue
    depois = ler(cid).get("new_step")
    saida.append({**base, "resultado": "marcado_publicado" if depois == "PUBLISHED" else "nao_confirmado",
                  "etapa_depois": depois, "corpo": corpo})

print(json.dumps(saida, ensure_ascii=False, indent=1))
```

O script relê cada item depois de gravar e só reporta `marcado_publicado` se o
`new_step` voltou `PUBLISHED`. `erro_gravacao` com HTTP 403 = o operador não tem
permissão de publicar nesse projeto — **o conteúdo continua no ar** (conta como
publicado para a tarefa), mas registre no relatório e na nota interna que a marcação
no Earth ficou pendente. Fallback sem earth-cli: o mesmo PATCH pela sessão logada do
app (Claude in Chrome, `fetch` com `credentials: "include"` em
`https://app.liveseo.com.br/v2/content/{post|page}/{projectId}/{id}/save-changes`).

### 4. Ler os comentários (contexto) e relatar
`earth --json task comment list <subtaskId>` (ou o MCP `get_subtask_comments`)
costuma ter pistas: confirmações de "Upado" e relatos de problema. Cite no relatório.

Entregue um quadro claro, separando **fato verificado** de **inferência**:

```
## Verificação de upload — {CB} / tarefa {id}

Produção: N conteúdos (dedup). Etapa atual (new_step / current_status):

| Status | Qtd |
|--------|-----|
| ✅ Publicados (PUBLISHED — no site) | X |
| ⏳ A publicar (PUBLISH — confirmado ao vivo: no ar / fora do ar) | A |
| 🚫 Cancelados/rejeitados (não precisam subir) | C |
| ❌ Pendentes (≤ CLIENT_REVIEW) | P |

A publicar (liste cada PUBLISH, o resultado da confirmação ao vivo e se foi
marcado como Publicado no Earth — passo 3b).
Pendências (liste cada um com nome, etapa e link do Earth).
Cancelados (liste, para transparência).
```

Trate o bucket **A publicar (`PUBLISH`)** sempre com o resultado da confirmação ao
vivo: os que estão no ar somam aos publicados de fato; os fora do ar viram
pendência de upload. Se houver **qualquer** item pendente (ou PUBLISH fora do ar),
diga isso explicitamente em vez de arredondar para "tudo certo".

## Comentar e finalizar

**Roteamento no kanban é automático.** Quando **todo item de produção estiver
`PUBLISHED` (ou `PUBLISH` confirmado no ar) ou cancelado/rejeitado** — zero
pendências e nenhum `PUBLISH` fora do ar — **mova a tarefa para "Para revisão"
sozinha, sem pedir confirmação** (autorização durável do Hugo) e **registre a nota
interna da validação** (abaixo), também sem pedir confirmação. Havendo qualquer
pendência, **não mova** — deixe na coluna atual e reporte o que falta.

Ordem quando está tudo ok: **3b marcar Publicado → mover para "Para revisão" →
nota interna**.

**O comentário público continua exigindo confirmação.** Postar algo visível ao
cliente é comunicação: **mostre o rascunho e só publique se o usuário confirmar.**

### Comentário público
Poste um comentário **público** (visível a todos, incluindo o cliente). Publicar é
ação de comunicação — **exige confirmação explícita antes de postar**.

- earth-cli: `earth --json --project CB### --yes comment add --body <arquivo.json>`
  com `{"id_content": <SUBTASK_ID>, "type_content": "SUBTASK", "comment": "<html>",
  "type": "MESSAGE", "checked": 0}` — SUBTASK_ID = o último número da URL da tarefa.
- MCP (fallback): `post_subtask_comment`.

O campo `comment` aceita HTML. Modelos:

- **Todos publicados (ou publicados + cancelados, sem pendência):**
  > ✅ Verificação de upload concluída: dos {N} conteúdos de produção, {X} já estão
  > publicados (no ar){, e {C} foi/foram cancelado(s)/rejeitado(s) e não precisa(m)
  > subir}. Esta atividade pode ser concluída.

- **Falta publicar:** liste cada pendente com **nome, etapa atual e a URL do
  conteúdo no Earth** (`https://app.liveseo.com.br/projeto/{CB}/{page|blog}/{id}`):
  > ⚠️ {P} de {N} conteúdos ainda não publicados: <br>
  > • {Nome} — em Revisão: https://app.liveseo.com.br/projeto/{CB}/page/{id} <br>
  > • {Nome} — em Produção: https://app.liveseo.com.br/projeto/{CB}/blog/{id} <br>
  > Os demais já estão no ar. Concluir só após publicar os itens acima.

  Monte a URL pelo **tipo** (page/blog) e **id** de cada item; use o `name`/título
  lido no passo 2 como texto.

### Rotear no kanban (automático quando tudo estiver no ar ou cancelado)
Regra: mova para **"Para revisão"** **automaticamente** quando **todo item de
produção estiver `PUBLISHED` (ou `PUBLISH` confirmado no ar) ou cancelado/rejeitado**
— ou seja, zero pendências e nenhum `PUBLISH` fora do ar. **Não peça confirmação
para esse movimento** (autorização durável do Hugo); apenas informe no relatório que
moveu. Resolva o id da coluna **pelo nome "Para revisão"** (no CB491/CB789 é 215, mas
**não fixe o id** — varia por projeto):

- earth-cli: `earth --json --project {id} task move <task_tag_id> <subtask_id> <step_id>`.
- MCP (fallback): `get_kanban_columns` + `move_subtask_kanban_step`.

**Se houver qualquer pendência — item ≤ CLIENT_REVIEW, ou `PUBLISH` que a
confirmação ao vivo mostrou fora do ar — NÃO mova.** Deixe na coluna atual e reporte
o que falta subir. Não use "Finalizada": a skill entrega para revisão, quem finaliza
é o líder/revisor.

### Nota interna da validação (automática, junto com o movimento)
Logo depois de mover para "Para revisão", registre na tarefa uma **nota interna**
(`type: INTERNAL_NOTE` — só o time vê, o cliente não) dizendo que a validação foi
feita e **quais conteúdos estão publicados**. É o rastro para quem for revisar: não
precisa refazer a checagem. Não exige confirmação (não é comunicação com o cliente).

- earth-cli: `earth --json --project CB### --yes comment add --body <arquivo.json>`
  com `{"id_content": <SUBTASK_ID>, "type_content": "SUBTASK", "comment": "<html>",
  "type": "INTERNAL_NOTE", "checked": 0}`.
- MCP (fallback): `post_subtask_internal_note`.

HTML aceito: `p, strong, em, ul/ol/li, a` (`<table>` é escapado pela API — use
lista). Modelo:

```html
<p><strong>[Validação de upload]</strong> Validação feita em {DD/MM/AAAA}: os {N}
conteúdos da Produção foram conferidos (etapa no Earth + confirmação ao vivo dos que
estavam em "Publicar"). Tarefa movida para Para revisão.</p>
<p><strong>Publicados ({X}):</strong></p>
<ul>
  <li><a href="{URL no ar}">{Nome}</a> — <a href="https://app.liveseo.com.br/projeto/{CB}/{page|blog}/{id}">Earth</a>{ — marcado como Publicado nesta validação}</li>
</ul>
<p><strong>Cancelados ({C}):</strong> {Nome} (<a href="...">Earth</a>), ...</p>
```

- Liste **todos** os publicados, com o link no ar e o link do Earth; marque com
  "marcado como Publicado nesta validação" os que o passo 3b mudou de etapa.
- Omita o bloco de cancelados se não houver.
- Se alguma marcação do passo 3b falhou (ex.: 403), diga na nota: "{Nome} está no
  ar, mas não foi possível marcar como Publicado no Earth ({motivo})".

O movimento e a nota interna (e a marcação Publicado do passo 3b) são os passos
automáticos. **Publicar comentário público continua sendo ação manual, com
confirmação.**

## Princípios

- **Leitor = `new_step` + `current_status` pela `earth-cli`** (`content list`/`get`).
  O `step` do MCP é legado; o MCP de conteúdo erra; o DW é proibido.
- **Verifique todos, sem amostragem.** Deduplique os ids; prefira `content list` em
  lote (1 chamada por tipo) e cruze pelos ids da Produção.
- **No ar com certeza = `PUBLISHED`.** `PUBLISH` ("Publicar") é "liberado pra subir",
  **não** garante que está no site — **sempre confirme ao vivo** os itens em PUBLISH.
  Qualquer etapa ≤ CLIENT_REVIEW = pendente.
- **Cancelado/rejeitado (`current_status: REJECTED`) não é pendência** — não conta
  como falta e não impede a conclusão.
- **A verdade é o status no Earth (e, em dúvida, o site ao vivo), não o bloco de
  Upload.** A lista de produção diz o que precisa estar no ar.
- **Confirmado no ar → marcado como Publicado no Earth.** Item em `PUBLISH` que a
  confirmação ao vivo mostrou no site vira `PUBLISHED` no módulo de conteúdo
  (passo 3b); o que não foi confirmado no ar não é marcado.
- **Roteamento automático quando tudo no ar/cancelado; comunicação com confirmação.**
  Verificar é livre; zero pendências → **move para "Para revisão" sozinha** e
  **registra a nota interna** com o que está publicado (sem confirmação).
  **Comentar (público, visível ao cliente) exige confirmação explícita.**
