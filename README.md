# verificar-upload-conteudo

Verifica, numa tarefa de **upload de conteúdo** da liveSEO
(`app.liveseo.com.br`), se **todos** os conteúdos produzidos já foram
publicados — e, com isso, comenta na tarefa se ela pode ser concluída ou aponta
o que falta subir.

Roda dentro do Claude Code, com a `earth-cli` autenticada como leitor de status.

**1 skill · sem sub-agente · custo de LLM $0**

---

## O que faz

| Skill | O que faz | Custo |
|---|---|---|
| `verificar-upload-conteudo` | Lê a lista de **Produção** de uma tarefa, checa a etapa atual de cada conteúdo (categoria ou blogpost) e devolve um quadro **publicado / a publicar / cancelado / não publicado**. Se tudo estiver publicado/cancelado, move a tarefa para "Para revisão" automaticamente; comentar (público) é oferecido com confirmação | $0 |

Dispara sozinha quando alguém cola um link de tarefa
(`…/projeto/CB###/task/<id>/<id>`) e pergunta se os conteúdos "foram
publicados", "estão no ar", "todos subiram", ou pede para "verificar o upload" /
"conferir a publicação das categorias/blogs".

## Como ela decide se está no ar

A pergunta parece simples — "já subiu tudo?" — mas o app tem três armadilhas, e a
skill existe justamente para não cair em nenhuma. Todas foram aprendidas na
prática (CB491, CB789):

```
lista de PRODUÇÃO (fonte da verdade)          bloco de UPLOAD
   dedup dos ids, tipo page/blog        ✗  (vazio / "Não encontrado" / stale)
            │
            ▼
   earth content list --param limit=500   →   new_step + current_status
   (1 chamada por tipo, cruza por id)          por conteúdo, status ATUAL
            │
            ▼
   new_step = PUBLISHED         =  ✅ no site (certeza)
   new_step = PUBLISH           =  ⏳ liberado pra subir → CONFIRMAR ao vivo
   current_status = REJECTED    =  🚫 cancelado, não é pendência
   new_step ≤ CLIENT_REVIEW     =  ❌ falta produzir/revisar/publicar
```

> ⚠ **Não use o MCP de conteúdo como leitor de status.** O
> `get_page_data_by_id` / `get_post_data_by_id` erra de forma consistente em boa
> parte das páginas — inclusive nas **publicadas** — e, quando responde, devolve
> o campo `step` **legado**, que fica desatualizado (ex.: blog 47807 do CB789
> aparece como `PLANNING`, mas o `new_step` real é `CLIENT_REVIEW`). O app usa
> `new_step`; a `earth-cli` e a skill também. O data warehouse (`earth-dw`) é
> acesso só de gestão — proibido aqui, quebraria a skill para outros operadores.

As três armadilhas, nomeadas:

- **O bloco de Upload mente.** A tarefa costuma ter um bloco "Upload" / "Controle
  de upload" além do de Produção, mas ele fica vazio ou desatualizado. A **lista
  de Produção** é a única fonte confiável do que precisa estar no ar.
- **`PUBLISH` ("Publicar") não é o mesmo que estar no site.** `PUBLISH` = conteúdo
  aprovado e **liberado pra subir** (handoff pro time técnico); `PUBLISHED`
  ("Publicado") = o técnico **subiu e confirmou**, aí sim está no ar. O upload é
  exatamente o passo que leva de um ao outro (no CB789, 14 dias entre `PUBLISH` e
  `PUBLISHED` no post 47691). Só `PUBLISHED` conta como no ar com certeza; **todo
  item em `PUBLISH` é confirmado ao vivo** — no ar conta, fora do ar é upload
  pendente. (Varia por squad: em algumas o item fica parado em `PUBLISH` mesmo já no
  ar; em outras `PUBLISH` é transitório e o que sobe vai pra `PUBLISHED`.)
- **Cancelado não é pendência.** Um item em `current_status: REJECTED` foi
  descartado na revisão e não precisa subir — não conta como "falta publicar" nem
  impede a conclusão.

## Estrutura

```
skills/verificar-upload-conteudo/
  SKILL.md                 # contrato: quando dispara, o fluxo de 4 passos,
                           # a tabela de classificação e os modelos de comentário
```

Uma skill só de procedimento — sem scripts, sem sub-agente. Toda a leitura e as
ações no app saem da `earth-cli` (fallback no MCP `liveseo-earth`).

## Como usar

Da interface do Claude Code, com a `earth-cli` autenticada, basta colar o link da
tarefa e perguntar:

> *"os conteúdos dessa tarefa já subiram? `…/projeto/CB789/task/…`"*

A skill então:

1. **Puxa a tarefa** e extrai só o bloco de **Produção**, deduplicando os ids e
   classificando cada um em `page` (categoria) ou `blog` (blogpost).
2. **Lê `new_step` + `current_status` de todos**, sem amostragem — em lote, uma
   chamada por tipo:

   ```bash
   earth --json --project 1079 content list blog --param limit=500
   earth --json --project 1079 content list page --param limit=500
   ```

3. **Confirma ao vivo** cada item em `PUBLISH` (além dos sem id do Earth ou com
   status duvidoso), comparando o título produzido com o `<h1>`/`document.title` da
   URL no ar.
4. **Relata** um quadro separando fato verificado de inferência:

   ```
   | Status                                            | Qtd |
   |---------------------------------------------------|-----|
   | ✅ Publicados (PUBLISHED — no site)                |  X  |
   | ⏳ A publicar (PUBLISH — confirmado ao vivo)       |  A  |
   | 🚫 Cancelados/rejeitados (não precisam subir)      |  C  |
   | ❌ Pendentes (≤ CLIENT_REVIEW)                     |  P  |
   ```

Verificar é livre. **Mover o card para "Para revisão" é automático:** quando tudo
estiver `PUBLISHED` (ou `PUBLISH` confirmado no ar) ou cancelado — zero pendências e
nenhum `PUBLISH` fora do ar — a skill **move sozinha, sem pedir confirmação**. Com
qualquer item pendente ou `PUBLISH` fora do ar, ela **não move**, e nunca move para
"Finalizada" (quem finaliza é o líder/revisor). **Comentar (público, visível ao
cliente) continua exigindo confirmação** — a skill mostra o rascunho antes de postar.

## Pré-requisito

- **`earth-cli` autenticada** — `earth --json auth status` tem que passar
  (exit 3 = não logado; `earth auth login` ou a env `EARTH_API_SESSION`). Sempre
  com `--json`.
- **Claude in Chrome** (opcional) — só como fallback do passo 3, para confirmar
  ao vivo um item sem id do Earth. Não é mais necessário para ler o status.

## Enum de referência (`new_step`)

Etapas na ordem da barra do app. Só `PUBLISHED` é "no ar" com certeza; `PUBLISH` é
confirmado ao vivo; o resto é pendência (rótulos das etapas 2-3 variam por projeto —
já visto `APPROVAL`, `SCHEDULE`, `BACKLOG`):

| # | Etapa (app)        | `new_step`            | Publicado? |
|---|--------------------|-----------------------|:----------:|
| 1 | Planejamento       | `PLANNING`            | ❌ |
| 2 | Aprovação de pauta | `SCHEDULE`/`APPROVAL` | ❌ |
| 3 | Backlog            | `BACKLOG`             | ❌ |
| 4 | Produção           | `PRODUCTION`          | ❌ |
| 5 | Revisão            | `REVIEW` / `REVIEWED` | ❌ |
| 6 | Revisão cliente    | `CLIENT_REVIEW`       | ❌ |
| 7 | **Publicar**       | `PUBLISH`             | ⏳ confirmar ao vivo |
| 8 | **Publicado**      | `PUBLISHED`           | ✅ no site |

`current_status`: `WAITING` é o normal de item no ar; `REJECTED` é
cancelado/rejeitado. O campo `status` (0/false/null) é inconsistente entre
endpoints — **não confie nele**; use `current_status` para detectar cancelado.

## Princípios

- **Leitor = `new_step` + `current_status` pela `earth-cli`.** O `step` do MCP é
  legado; o MCP de conteúdo erra; o DW é proibido.
- **Verifica todos, sem amostragem.** Deduplica ids; prefere `content list` em
  lote e cruza pelos ids da Produção.
- **A verdade é o status no Earth (e, em dúvida, o site ao vivo), não o bloco de
  Upload.**
- **Roteamento automático; comunicação com confirmação.** Verificar é livre; zero
  pendências → move para "Para revisão" sozinha. **Comentar (público) exige
  confirmação explícita.**
