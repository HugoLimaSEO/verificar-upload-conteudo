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
  publicação. A skill só VERIFICA e relata; comentar e finalizar a tarefa são
  passos que ela oferece e só executa após confirmação explícita do usuário.
metadata:
  type: workflow
  source: >-
    Etapa atual de cada conteúdo é lida pela earth-cli (`earth content list` em
    lote, ou `earth content get` unitário), campos `new_step` + `current_status`.
    Tarefa/lista de produção, comentário e movimento de kanban também via
    earth-cli (fallback MCP liveseo-earth). NÃO usa o data warehouse (gestão) nem
    o get_page_data_by_id do MCP (devolve o `step` legado e erra em muitas páginas).
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

**Classificação de cada item:**

| Situação | Regra | Conta como |
|----------|-------|-----------|
| **Publicado** | `new_step` ∈ {`PUBLISH`, `PUBLISHED`} | ✅ no ar |
| **Cancelado/rejeitado** | `current_status` = `REJECTED` (ou marcado cancelado) | 🚫 não precisa subir |
| **Pendente** | qualquer outro `new_step`, sem ser cancelado | ❌ falta publicar |

Erra para o lado de sinalizar pendência: `new_step` desconhecido e **não**
cancelado = pendente.

**Por que `PUBLISH` também conta:** no fluxo da liveSEO, **blogpost no ar costuma
ficar em `PUBLISH`** ("Publicar") e **página no ar costuma ir para `PUBLISHED`**
("Publicado"). Aceitar só `PUBLISHED` marcaria blog publicado como pendente.

**Enum oficial (`new_step`)**, na ordem da barra de etapas do app:

| # | Etapa (app) | `new_step` | Publicado? |
|---|-------------|-----------|-----------|
| 1 | Planejamento | `PLANNING` | ❌ não |
| 2 | Aprovação de pauta | `SCHEDULE`* | ❌ não |
| 3 | Backlog | `BACKLOG`* | ❌ não |
| 4 | Produção | `PRODUCTION` | ❌ não |
| 5 | Revisão | `REVIEW` / `REVIEWED` | ❌ não |
| 6 | Revisão cliente | `CLIENT_REVIEW` | ❌ não |
| 7 | **Publicar** | `PUBLISH` | ✅ **sim** |
| 8 | **Publicado** | `PUBLISHED` | ✅ **sim** |

\* rótulos internos de "Aprovação de pauta"/"Backlog" não foram confirmados — como
a regra é accept-list, qualquer coisa fora de {`PUBLISH`,`PUBLISHED`} já cai em
"não publicado", então não depende desses nomes.

`current_status` observado: `WAITING` (normal, itens no ar ficam assim),
`REJECTED` (cancelado/rejeitado). O campo `status` (0/false/null) é **inconsistente
entre endpoints** — não confie nele; use `current_status` para detectar cancelado.

> **Não use `get_page_data_by_id` / `get_post_data_by_id` (MCP) como leitor de
> status.** Erram em muitas páginas e, quando respondem, trazem o `step` legado
> (errado). Use a `earth-cli`.

### 3. (Opcional) Confirmar ao vivo — só quando houver dúvida ou id ausente
Para item **sem id do Earth** (a Produção só linka a URL final) ou quando o
`new_step` parecer inconsistente com o contexto, confirme abrindo a URL no ar
(Claude in Chrome ou navegação) e comparando o **título produzido** com o
`document.title`/`<h1>` da página:

- Títulos **batem** e sem 404 → publicado. ✅
- Título **genérico/diferente** ou **404** → trate como **não publicado**. ⚠️

Cuidado: **alguns domínios de loja bloqueiam/atrasam** o navegador (timeout); se
não abrir, registre que a confirmação ao vivo não foi possível.

### 4. Ler os comentários (contexto) e relatar
`earth --json task comment list <subtaskId>` (ou o MCP `get_subtask_comments`)
costuma ter pistas: confirmações de "Upado" e relatos de problema. Cite no relatório.

Entregue um quadro claro, separando **fato verificado** de **inferência**:

```
## Verificação de upload — {CB} / tarefa {id}

Produção: N conteúdos (dedup). Etapa atual (new_step / current_status):

| Status | Qtd |
|--------|-----|
| ✅ Publicados | X |
| 🚫 Cancelados/rejeitados (não precisam subir) | C |
| ❌ Pendentes | P |

Pendências (liste cada não publicado com nome, etapa e link do Earth).
Cancelados (liste, para transparência).
```

Se houver **qualquer** item pendente (nem publicado nem cancelado), diga isso
explicitamente em vez de arredondar para "tudo certo".

## Comentar e finalizar (só com confirmação)

A skill **não** posta nada nem move o card por conta própria. Depois de relatar,
**mostre o rascunho do comentário** e só publique se o usuário confirmar.

### Comentário público
Poste um comentário **público** (visível a todos, incluindo o cliente). Publicar é
ação de comunicação — **exige confirmação explícita antes de postar**.

- earth-cli: `earth --json --yes task comment add` com `id_content` = **SUBTASK_ID**
  (o último número da URL da tarefa), `type_content=SUBTASK`, `type=MESSAGE`,
  `comment=<html>`.
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

### Rotear no kanban (só quando NÃO houver pendência)
Regra: **se nenhum item de produção estiver pendente** (todos publicados e/ou
cancelados/rejeitados), mova a tarefa para a coluna **"Para revisão"** (revisão
humana antes de finalizar) — só após confirmação do usuário. Resolva o id da coluna
**pelo nome "Para revisão"** (no CB491/CB789 é 215, mas **não fixe o id** — varia por
projeto):

- earth-cli: `earth --json --project {id} task move <task_tag_id> <subtask_id> <step_id>`.
- MCP (fallback): `get_kanban_columns` + `move_subtask_kanban_step`.

**Se houver qualquer pendência (item não publicado e não cancelado), NÃO mova** —
deixe na coluna atual e reporte o que falta. Não use "Finalizada": a skill entrega
para revisão, quem finaliza é o líder/revisor.

## Princípios

- **Leitor = `new_step` + `current_status` pela `earth-cli`** (`content list`/`get`).
  O `step` do MCP é legado; o MCP de conteúdo erra; o DW é proibido.
- **Verifique todos, sem amostragem.** Deduplique os ids; prefira `content list` em
  lote (1 chamada por tipo) e cruze pelos ids da Produção.
- **Accept-list em `PUBLISH` + `PUBLISHED`.** Blog no ar costuma ficar em
  `PUBLISH`; página no ar, em `PUBLISHED`. Qualquer etapa anterior = não publicado.
- **Cancelado/rejeitado (`current_status: REJECTED`) não é pendência** — não conta
  como falta e não impede a conclusão.
- **A verdade é o status no Earth (e, em dúvida, o site ao vivo), não o bloco de
  Upload.** A lista de produção diz o que precisa estar no ar.
- **Ações no app só com o dedo do usuário.** Verificar é livre; **comentar
  (público) e finalizar exigem confirmação explícita**.
