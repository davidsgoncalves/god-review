# god-review

Addon da skill **GOD** para code review de Pull Requests do GitHub.

> **⚠️ Requer a skill GOD instalada.** Essa skill não é standalone — ela lê `GOD/learned-patterns.md` pra gerar comentários e cria seus arquivos dentro da pasta `GOD/` do seu projeto. Sem GOD instalado e configurado (`GOD/VERSION` presente), `god-review` não roda. Instale o GOD primeiro e só depois adicione esse addon.

Lista os PRs abertos que pedem sua atenção, gera comentários inline apoiado no conhecimento acumulado do GOD, apresenta tudo em batch no terminal, e publica o review via `gh` após sua confirmação.

## Por que existe

A sub-skill `review` do GOD compara plano vs execução de uma task local. Essa skill é outro ângulo: revisar o que **os outros** escreveram em PRs que pedem sua atenção, sem precisar abrir o GitHub. Ela aproveita o `learned-patterns.md` do GOD — as regras que você foi destilando com o tempo — e aplica de volta nos PRs dos outros como insumo pros comentários.

## O que ela faz

```
PRs pendentes (review pedido + comentários com resposta nova)
    ↓
Tabela de seleção no terminal
    ↓
Para cada PR:
    • lê o diff e os arquivos
    • consulta learned-patterns.md (o quê comentar)
    • consulta review-guide.md (como comentar — tom, rigor por área)
    • gera pacote de comentários inline + overall
    • apresenta tudo em blocos ASCII
    • você revisa: remove / edita / escala / aprova
    • publica num review único via gh api
    • captura feedback pro review-guide.md crescer
```

## Pré-requisitos

- **GOD instalado** na versão atual no projeto onde você chama a skill. `GOD/VERSION` precisa existir.
- **`gh` CLI** autenticado com a conta que tem acesso aos PRs (`gh auth status`).
- **Primeiro uso** vai perguntar quais repos deste GOD estão no GitHub, criar `GOD/review-config.yml` e `GOD/review-guide.md`.

## Arquivos que ela gerencia

Dentro da pasta `GOD/` do projeto onde a skill é chamada:

| Arquivo | Propósito | Quem escreve |
|---------|-----------|--------------|
| `review-config.yml` | Lista de repos GitHub cobertos + user autenticado | Primeiro uso; usuário edita manualmente |
| `review-guide.md` | Guia de tom, rigor por área, escalações, coisas a não comentar | Primeiro uso cria o template; cresce com feedback após cada review |

**Não toca em:** `knowledge.md`, `learned-patterns.md`, `patterns.md`, `hooks.md`, nada dentro de `tasks/`.

## Como usar

### Primeira vez

Num projeto que já tem o GOD instalado:

```
> god-review
```

A skill detecta que `review-config.yml` não existe e roda o primeiro uso:
1. Identifica seu user do GitHub (`gh api /user`).
2. Agrega candidatos a repos: `git remote`, `patterns.md`, histórico de PRs em `tasks/*/status.md`.
3. Te apresenta a lista, você confirma / edita.
4. Cria os dois arquivos.

Depois entra no fluxo normal.

### Uso normal

```
> revisar meus PRs
```

Mostra tabela dos PRs pendentes (dois motivos possíveis: `review-pedido` ou `comentário` com resposta nova):

```
  #  REPO          PR    TÍTULO                        MOTIVO         ATUALIZADO
  1  vak/api       #418  Add phone field to user       review-pedido  há 2h
  2  vak/web       #92   Refactor checkout form        comentário     há 1d
```

Escolhe um número (ou `todos` pra rodar em sequência) e a skill gera o pacote de comentários.

### Revisando um PR

Cada comentário aparece num bloco ASCII com trecho de código, mensagem e prefixo:

```
┌─ [1] blocking · src/users/phone.ts:42 ──────────────────────────────
│
│ Código:
│    42 │   user.phone = input.phone;
│
│ Comentário:
│   blocking: falta validação do formato antes de salvar.
│   Ver learned-patterns.md → "Validações de input em boundary".
│
└──────────────────────────────────────────────────────────────────────
```

Ao final, um resumo + menu de ações:

```
5 comentários · 1 blocking, 2 suggestion, 2 nit · próximo estado: REQUEST_CHANGES

• "ok" / "aprovar" — publica tudo
• "remover 2, 4"    — tira comentários da lista
• "editar 3"        — abre o comentário pra reescrever
• "escalar 5 blocking" — muda o prefixo
• "cancelar"        — descarta tudo
```

Ao confirmar, publica via `gh api POST /reviews` (tudo num review só, atômico). Depois pergunta se você quer registrar algum aprendizado no `review-guide.md`.

## Convenção de prefixos

Padrão Conventional Comments enxuto — todo comentário inline começa com um prefixo:

| Prefixo | Quando usar | Efeito no review state |
|---------|-------------|------------------------|
| `blocking:` | Precisa resolver antes do merge | `REQUEST_CHANGES` |
| `suggestion:` | Proposta com motivação, autor decide | `COMMENT` |
| `nit:` | Detalhe pequeno | `COMMENT` |
| `question:` | Dúvida real | `COMMENT` |
| `praise:` | Elogio sincero | `COMMENT` |

Se nenhum comentário tem `blocking:` e você disser "aprovar" explicitamente, vira `APPROVE`. Qualquer `blocking:` no pacote → `REQUEST_CHANGES`.

## Como `review-guide.md` evolui

Depois de cada review publicado, a skill pergunta:

```
Algum ajuste no review-guide.md a partir deste review?
```

Feedback comum que vira regra:

- **Rigor extra por área** — "em `src/billing/`, qualquer nit sobe pra suggestion".
- **Rigor reduzido** — "testes não precisam de nits de nome".
- **Não comentar** — "não comentar ausência de testes em PRs com label `wip`".
- **Escalações** — "uso de `any` em TS de domínio → sempre `blocking:`".
- **Tom** — "mais direto da próxima, menos hedging".

A skill adiciona o texto na seção correta. Você pode editar o arquivo manualmente a qualquer momento.

## Relação com as outras skills

- **GOD** — pré-requisito. Esta skill não faz nada sem `GOD/` instalado.
- **sub-skill `review` do GOD** — assunto diferente. Aquela compara plano vs execução local; essa aqui revisa PRs remotos.
- **`learn` do GOD** — complementar. `learn` destila uma task finalizada em regras no `learned-patterns.md`; `god-review` consome essas regras pra comentar nos PRs dos outros. O loop fecha: você aprende implementando, e reusa revisando.

## Guard-rails

- **Nunca publica sem confirmação explícita** do usuário.
- **Comentário sobre linha de código vai inline**, sempre. Nada de esconder observação linha-específica no body do review.
- **Não revisa PR de repo fora de `review-config.yml`** — pergunta se você quer adicionar o repo antes.
- **Não escreve em arquivos de outras skills** (`knowledge.md`, `learned-patterns.md`, `patterns.md`).
- **Não cria branches, commits ou alterações locais** — essa skill só lê diff remoto e escreve review remoto.

## Modos de invocação

| Você diz | Comportamento |
|----------|---------------|
| `revisar meus PRs` / sem argumento | Busca A+B, mostra tabela |
| `revisar PR {url}` ou `{owner/repo}#{n}` | Pula tabela, vai direto |
| `só review pedidos` | Usa só o caso A (review-requested) |
| `só com resposta nova` | Usa só o caso B (PRs com atividade nova onde comentei) |
| `reset config` | Apaga `review-config.yml` e roda primeiro uso de novo |

## Status

Primeira versão. A skill assume estrutura do GOD v5. Se rodar em instalação v4 ou inferior, vai pedir `upgrade` antes.
