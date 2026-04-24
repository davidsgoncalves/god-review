---
name: god-review
description: |
  Addon do GOD para code review de Pull Requests do GitHub. Lista PRs abertos que requerem atenção do usuário (review pedido + PRs onde ele comentou e tiveram nova atividade), gera pacote de comentários inline baseado em `GOD/learned-patterns.md` + `GOD/review-guide.md`, apresenta em bloco ASCII pro usuário revisar em batch, e publica via GitHub API após confirmação. Requer GOD instalado e `gh` CLI autenticado. Use quando o usuário mencionar: "god-review", "revisar meus PRs", "review dos PRs abertos", "PRs pendentes", "revisar PR", "checar PRs que pedem atenção". NÃO confundir com a sub-skill `review` do GOD, que compara plano vs execução de uma task.
tools: Read, Write, Edit, Bash, Grep, Glob, Agent
---

# god-review — Code review de PRs do GitHub

Addon do GOD. Lê os PRs abertos que pedem atenção do usuário no GitHub, gera comentários inline apoiado no conhecimento acumulado do GOD (`learned-patterns.md`) e num guia de estilo próprio do reviewer (`review-guide.md`), apresenta tudo em batch no terminal, e publica via `gh` após confirmação.

Esta skill NÃO é a sub-skill `review` do GOD — aquela compara plano vs execução de uma task. Esta aqui revisa Pull Requests remotos.

## Pré-requisitos (checar antes de qualquer coisa)

| Item | Como verificar | Se faltar |
|------|----------------|-----------|
| GOD instalado na versão atual | `GOD/VERSION` existe e lê `v5` (ou versão vigente) | Orientar o usuário a rodar `install` ou `upgrade` da skill GOD. NÃO prosseguir. |
| `gh` CLI autenticado | `gh auth status` retorna 0 | Pedir pro usuário rodar `gh auth login`. NÃO prosseguir. |
| `GOD/review-config.yml` | Arquivo existe | Rodar **primeiro uso** (ver abaixo) antes da invocação normal. |
| `GOD/review-guide.md` | Arquivo existe | Criar no primeiro uso. |

## Arquivos que a skill gerencia (dentro de `GOD/`)

Todos ficam em `GOD/` (ao lado de `patterns.md`, `learned-patterns.md`, etc.):

- **`review-config.yml`** — repos GitHub cobertos por este GOD, usuário autenticado. Criado no primeiro uso; editável manualmente.
- **`review-guide.md`** — guia crescente de tom, rigor por área, e preferências do reviewer. Começa com template básico e cresce com feedback capturado a cada review.

Esta skill **nunca** escreve em `GOD/knowledge.md`, `GOD/learned-patterns.md` ou `GOD/patterns.md`.

## Fluxo da skill

```
pré-checks → (primeiro uso, se necessário) → buscar PRs pendentes
  → tabela de seleção → revisar PR escolhido (gera → apresenta → confirma → publica)
  → capturar feedback → atualizar review-guide.md → próximo PR ou sair
```

---

## Primeiro uso

Rodar apenas se `GOD/review-config.yml` não existir.

### 1. Identificar o usuário do GitHub

```bash
gh api /user --jq '.login'
```

Guardar como `github_user` no config.

### 2. Descobrir candidatos a repos

Agregar sinais, nessa ordem de prioridade:

1. **`git remote -v`** no cwd (se for um repo git único).
2. **`GOD/patterns.md`** — seção "Branch inicial" pode citar projetos.
3. **`GOD/tasks/*/status.md`** — campo `prs:` tem URLs reveladoras (extrair `owner/repo` de cada URL).
4. **`gh repo list <org> --limit 200`** — só consultar se os sinais acima forem insuficientes e o usuário sinalizar a org.

Deduplicar e apresentar a lista ao usuário:

```
Encontrei estes repos como candidatos a cobertura deste GOD:

  1. vakinha/api          (via git remote)
  2. vakinha/web          (via tasks/PROJ-123/status.md)
  3. vakinha/mobile       (via tasks/PROJ-124/status.md)

Confirma todos? Remove algum? Adiciona mais algum (formato owner/repo)?
```

Aguardar resposta. Normalizar resposta em lista final.

### 3. Escrever `GOD/review-config.yml`

```yaml
github_user: davidgoncalves
repos:
  - vakinha/api
  - vakinha/web
  - vakinha/mobile
created_at: 2026-04-24
```

### 4. Criar `GOD/review-guide.md` a partir do template

Escrever o template abaixo (sem perguntar — o arquivo é pra crescer com uso).

```markdown
# Review Guide

Guia próprio deste projeto para code review. Cresce com os feedbacks do usuário ao longo das revisões.

## Tom e estilo

- Objetivo, direto, sem bajulação.
- Português (PT-BR) por padrão. Se o PR estiver em inglês, comentar em inglês.
- Perguntar, não acusar — "faz sentido X aqui?" > "isso está errado".

## Convenção de prefixos (Conventional Comments — versão enxuta)

Todo comentário inline começa com um prefixo. Escolher conforme o peso:

- `blocking:` — precisa ser resolvido antes do merge (bug, regressão, falha de segurança, contrato quebrado).
- `suggestion:` — proposta de mudança com motivação, autor decide.
- `nit:` — detalhe pequeno (nome, formatação), autor decide sem fricção.
- `question:` — dúvida real, não velada.
- `praise:` — elogio sincero a um trecho bem feito. Usar com parcimônia.

## Áreas de rigor extra

(vazio — preencher com "em arquivos sob src/billing/, subir nits a suggestion" e afins)

## Áreas de rigor reduzido

(vazio — ex.: "testes não precisam de nits de nome; código gerado não comentar")

## Tópicos que NÃO comentar

(vazio — ex.: "não comentar sobre ausência de testes em PRs marcados como WIP")

## Escalações aprendidas

(vazio — ex.: "qualquer uso de `any` em código TypeScript de domínio → escalar para `blocking`")
```

Reportar ao usuário: "`GOD/review-config.yml` e `GOD/review-guide.md` criados. Pode editar manualmente a qualquer momento."

---

## Invocação normal

### 1. Buscar PRs pendentes (A + B)

**Caso A — Reviews pedidos ao usuário e ainda não feitos:**

```bash
gh search prs --review-requested=@me --state=open \
  --json number,title,url,repository,updatedAt,author
```

**Caso B — PRs onde o usuário já comentou e houve atividade depois:**

Abordagem primária (mais barata): usar notifications do GitHub.

```bash
gh api /notifications --paginate \
  --jq '.[] | select(.subject.type == "PullRequest") | select(.reason == "comment" or .reason == "mention" or .reason == "review_requested")'
```

Filtrar por repos presentes em `review-config.yml`. O `updated_at` de cada notification é o sinal de "tem atividade nova que você ainda não viu".

**IMPORTANTE — filtrar por estado do PR:** notifications ficam no inbox mesmo depois que o PR é mergeado ou fechado. Sem filtro adicional, PRs já fechados aparecem na tabela e o usuário acaba revisando código morto. Antes de apresentar a tabela, para **cada** candidato do Caso B, buscar o estado do PR e descartar os que não forem `OPEN`:

```bash
gh pr view {n} --repo {owner/repo} --json state
```

Agrupar por repo e fazer em batch quando possível. Se o volume for grande (>30 candidatos), considerar `gh api /search/issues?q=is:pr+is:open+repo:{owner/repo}` e interseccionar com os números das notifications.

Abordagem secundária (se o usuário limpa notifications com frequência):
1. Listar PRs abertos onde ele deu review: `gh search prs --reviewed-by=@me --state=open --json number,repository,url,updatedAt`.
2. Para cada um, buscar timestamp do último review dele: `gh api /repos/{owner}/{repo}/pulls/{n}/reviews --jq '[.[] | select(.user.login == "{user}")] | max_by(.submitted_at) | .submitted_at'`.
3. Comparar com `updatedAt` do PR. Se PR foi atualizado depois e o updater não é ele, está pendente.

Merge dos resultados de A e B, deduplicar por `owner/repo#number`.

**Guard antes de revisar (defesa em profundidade):** mesmo com o filtro acima, reconsultar `state` no 3.1 (coleta de contexto) antes de gerar comentários. Se voltar `MERGED` / `CLOSED`, interromper e avisar o usuário — o PR pode ter sido fechado entre a tabela e a seleção (race comum quando o usuário revisa algo que acabou de ser aprovado por outra pessoa).

**Filtrar PRs do próprio usuário:** descartar PRs cujo `author.login == github_user`. Caso A (`--review-requested=@me`) naturalmente não traz seus próprios PRs, mas Caso B (notifications) traz — você pode ser mencionado ou comentar em PR seu. Incluir filtro `author.login != github_user` no merge final.

### 2. Apresentar tabela

Mostrar tabela limpa, ordenada por `updatedAt` decrescente:

```
Seus PRs pendentes:

  #  REPO             PR    TÍTULO                                   MOTIVO         ATUALIZADO
  1  vakinha/api      #418  Add phone field to user                  review-pedido  há 2h
  2  vakinha/web      #92   Refactor checkout form                   comentário     há 1d
  3  vakinha/api      #415  Migrate auth middleware                  review-pedido  há 3d

Qual revisar? (número, "todos" pra rodar em sequência, "sair")
```

### 3. Revisar 1 PR (loop pra cada PR selecionado)

#### 3.1 Coletar contexto do PR

```bash
gh pr view {n} --repo {owner/repo} --json number,title,body,headRefOid,commits,files
gh pr diff {n} --repo {owner/repo}
```

Guardar `headRefOid` — é o `commit_id` obrigatório pra comentários inline.

Ler também:
- `GOD/learned-patterns.md` (insumo de **conteúdo** — o que comentar).
- `GOD/review-guide.md` (insumo de **estilo** — como comentar, rigor por área).

#### 3.2 Gerar pacote de comentários

Para PRs grandes (>20 arquivos ou >500 linhas de diff), considerar rodar a análise via subagent `Explore` pra não estourar contexto.

Regras ao gerar:
- **Comentário sobre trecho de código específico → SEMPRE inline** (path + line + side). Nunca condensar observações de linha no body do review.
- Comentário sobre arquitetura, direção geral, ausência de testes no PR todo → vai no `body` do review (overall).
- Cada comentário deve citar, se aplicável, a regra do `learned-patterns.md` ou `review-guide.md` que o justifica (ajuda o autor a aprender o padrão).
- Não comentar coisas listadas em "Tópicos que NÃO comentar" no `review-guide.md`.
- Aplicar escalações aprendidas (ex.: `any` em TS de domínio → `blocking:`).
- Duplicar sensibilidade: se o mesmo padrão ruim aparece 5x no PR, comentar na primeira ocorrência e mencionar "também em linhas X, Y, Z" — não poluir com 5 comments iguais.

Estrutura interna de cada comentário gerado:

```
{
  path: "src/users/phone.ts",
  line: 42,
  side: "RIGHT",
  prefix: "blocking",
  body: "blocking: falta validação do formato do telefone antes de salvar...",
  justification: "learned-patterns.md → 'Validações de input em boundary'"
}
```

#### 3.3 Apresentar o pacote ao usuário (batch, ASCII bonito)

Sempre fora de code fences (terminal renderiza box-drawing chars nativamente). Separador topo:

```
═══════════════════════════════════════════════════════════════════════════
  PR #418 · vakinha/api · Add phone field to user
  Autor: @fulano · head: a1b2c3d · 8 arquivos, +142 −37
═══════════════════════════════════════════════════════════════════════════
```

Pra cada comentário inline, um bloco:

```
┌─ [1] blocking · src/users/phone.ts:42 ──────────────────────────────────
│
│ Código:
│    40 │ export function createUser(input: UserInput) {
│    41 │   const user = new User();
│    42 │   user.phone = input.phone;
│    43 │   return userRepo.save(user);
│    44 │ }
│
│ Comentário:
│   blocking: falta validação do formato do telefone antes de salvar.
│   Ver learned-patterns.md → "Validações de input em boundary".
│
└─────────────────────────────────────────────────────────────────────────
```

Comentário overall (body do review) num bloco próprio:

```
┌─ [overall] comentário do review ────────────────────────────────────────
│
│ O PR cobre bem o caso feliz, mas não tem testes pro cenário de telefone
│ inválido. Considere adicionar antes do merge.
│
└─────────────────────────────────────────────────────────────────────────
```

Resumo no final:

```
───────────────────────────────────────────────────────────────────────────
  5 comentários · 1 blocking, 2 suggestion, 2 nit · overall: COMMENT
  Próximo estado do review: REQUEST_CHANGES (há blocking)
───────────────────────────────────────────────────────────────────────────

O que fazer?
  • "ok" / "aprovar" — publica tudo como está
  • "remover 2, 4" — exclui comentários específicos
  • "editar 3" — entra em modo edição pro comentário 3
  • "escalar 5 blocking" — muda prefixo de um comentário
  • "cancelar" — descarta tudo, não publica
```

#### 3.4 Processar resposta do usuário

Loop até o usuário dizer `ok`/`aprovar` ou `cancelar`:
- `remover N, M` → tira da lista.
- `editar N` → pedir novo texto, substituir body preservando path/line/prefix.
- `escalar N {prefix}` → trocar prefix (regerar body mantendo conteúdo).
- `adicionar {path}:{line} {prefix} {texto}` → permite usuário inserir comentário que a skill não viu.
- Depois de qualquer edição, re-apresentar só os blocos afetados + novo resumo.

#### 3.5 Publicar via GitHub API

Determinar `event` do review:
- Há algum comentário `blocking:` → `REQUEST_CHANGES`.
- Tem comentários mas nenhum blocking → `COMMENT`.
- Zero comentários + usuário explicitamente disse "aprovar" → `APPROVE`.

Publicar tudo num único review (atomicidade):

```bash
gh api -X POST /repos/{owner}/{repo}/pulls/{n}/reviews \
  -f commit_id="{headRefOid}" \
  -f event="REQUEST_CHANGES" \
  -f body="{overall body ou vazio}" \
  --input - <<'EOF'
{
  "commit_id": "...",
  "event": "REQUEST_CHANGES",
  "body": "...",
  "comments": [
    {"path": "src/users/phone.ts", "line": 42, "side": "RIGHT", "body": "blocking: ..."},
    ...
  ]
}
EOF
```

Na prática, montar o JSON no filesystem (`/tmp/review-{n}.json`) e passar via `--input`. Capturar URL do review no retorno e exibir pro usuário.

#### 3.6 Capturar feedback pro `review-guide.md`

Logo após publicar, perguntar:

```
Algum ajuste no review-guide.md a partir deste review?

Ex.: "sempre escalar uso de `any` pra blocking", "não comentar ausência de
testes em PRs com label wip", "tom muito formal — mais direto da próxima".

(enter pra pular)
```

Se o usuário responder algo substantivo, adicionar na seção correta do `review-guide.md` (Tom / Rigor extra / Rigor reduzido / Não comentar / Escalações). Se não couber em nenhuma, criar seção "Notas" no final. Se o usuário só der enter, seguir.

### 4. Próximo PR ou sair

Se o usuário escolheu `todos` na tabela inicial, pegar o próximo. Senão, perguntar se quer revisar outro ou sair.

---

## Guard-rails

- **Nunca publicar sem confirmação.** Qualquer chamada a `gh api -X POST /reviews` exige um `ok`/`aprovar` explícito do usuário no turno imediatamente anterior.
- **Comentários sobre código = inline, sempre.** Se um comentário se refere a linhas específicas, não enfiar no body do review. A skill deve detectar isso ao gerar e alertar se o usuário tentar mover algo linha-específico pro overall.
- **Não escrever em `GOD/knowledge.md`, `GOD/learned-patterns.md`, `GOD/patterns.md`** — são territórios de outras skills.
- **Não criar branches, commits, ou alterações locais no projeto** — essa skill só lê diff remoto e escreve review remoto.
- **Respeitar `GOD/review-config.yml`** — nunca revisar PR de repo que não está nessa lista. Se o usuário passar URL de PR fora da lista, perguntar se quer adicionar o repo ao config antes.
- **Falha do `gh` é falha dura** — não tentar reconstruir o review em outro canal. Reportar o erro e parar.

---

## Modos de invocação (resumo)

| Intenção do usuário | Comportamento |
|---------------------|---------------|
| "revisar meus PRs" / sem argumento | Busca A+B, apresenta tabela, aguarda seleção |
| "revisar PR {url}" ou "{owner/repo}#{n}" | Pula tabela, vai direto pro review daquele PR (valida se repo está no config) |
| "só review pedidos" | Usa só caso A |
| "só com resposta nova" | Usa só caso B |
| "reset config" | Deleta `review-config.yml` e re-roda primeiro uso |

---

## Notas de implementação

- **Paginação**: `gh api --paginate` pra notifications e searches. PRs com muitos comentários podem precisar paginar `/reviews` e `/comments`.
- **Rate limit**: chamadas a `gh api` são baratas dentro do token do usuário, mas evitar loops desnecessários. Cachear em variáveis dentro do fluxo, não re-buscar.
- **Diff grande**: se `gh pr diff` retornar >500 linhas, considerar rodar a análise via subagent `Explore` com o diff como input, devolvendo só a lista de comentários candidatos.
- **PR em draft**: mostrar na tabela mas marcar como `draft` na coluna MOTIVO. Perguntar antes de revisar — draft costuma não pedir review formal.
