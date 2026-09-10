---
name: local-ultrareview
description: Multi-model code review that runs Opus 4.7, Sonnet 4.6 and Haiku 4.5 in parallel via subagents and aggregates findings into a single report. Use when the user wants a thorough review of a local branch, a path, or a GitHub PR with multi-model perspectives. Triggers — `/local-ultrareview`, "ultrareview local", "multi-model review", "ensemble review", "revisa com sonnet e haiku".
license: MIT
---

# local-ultrareview

Runs an **ensemble code review** locally: the same diff is reviewed in parallel by three subagents on Opus, Sonnet and Haiku, then the orchestrator (you) aggregates the findings into one report — flagging which findings converge (multiple models agreed → high confidence) vs. which are unique to a single model (still useful, but verify).

**Why ensemble.** A single model has consistent blind spots. Different model sizes have *different* blind spots: Haiku catches surface-level issues fast, Sonnet is good at architectural smells, Opus reasons deeper about correctness. Running them in parallel and cross-checking gives a more reliable signal than any single pass.

**Scope.** This skill is for the *orchestration*: it does not contain the review prompt logic — that lives in the subagent instructions below. Tune those prompts as you learn what each model is best at.

---

## When to invoke

- User typed `/local-ultrareview` (with or without arguments).
- User asked for a "multi-model review", "review with sonnet and haiku", "ensemble review", "review caseiro", "ultrareview local".
- User explicitly asked you to cross-check a diff against multiple models.

**Do NOT invoke** for: single-file syntax checks, formatting questions, or anything the user can answer themselves in < 1 minute. The minimum cost is 3 subagent invocations — don't burn that on trivia.

---

## Inputs (argument parsing)

The skill accepts one optional argument:

| Argument form | Meaning | How to collect the diff |
|---|---|---|
| *(none)* | Current branch vs default branch + working tree | `git diff $(git merge-base HEAD "$(git rev-parse --abbrev-ref origin/HEAD 2>/dev/null \| sed 's@^origin/@@' \|\| echo main)")..HEAD` then append `git diff HEAD` for uncommitted |
| `<integer>` (e.g. `123`) | GitHub PR number | `gh pr diff <num>` + `gh pr view <num> --json title,body,headRefName,baseRefName` |
| `<path>` (file or directory) | Just that path's pending changes | `git diff -- <path>` + `git diff HEAD -- <path>` |
| `--all-staged` | Only staged changes | `git diff --staged` |

If the working directory is not a git repository, abort with a clear message — there is nothing to diff. Offer to run `git init` if the user wants.

---

## Orchestration flow

### Step 1 — Determine scope and collect context

1. Run the appropriate `git`/`gh` commands above to capture the diff.
2. If the diff is **larger than ~1500 lines**, ask the user before proceeding (or offer to scope to a sub-path). Subagent context still fits, but signal-to-noise drops.
3. Capture lightweight context:
   - `git log --oneline -20` (recent commits — helps subagents understand intent)
   - Top-level `CLAUDE.md` if present (project conventions)
   - `README.md` first 80 lines (project purpose)
   - List of modified files with line counts

Save all of this to a working bundle in memory — you will pass it verbatim to each subagent so they all see *exactly* the same input.

### Step 2 — Dispatch three subagents in parallel

**Critical:** make a single message with three `Agent` tool calls so they run concurrently. Sequential dispatch wastes wall-clock and defeats the point.

For each subagent, use:

- `subagent_type: "general-purpose"`
- `model:` — one of `"opus"`, `"sonnet"`, `"haiku"`
- `description:` — `"<model> ensemble review"` (e.g. `"sonnet ensemble review"`)
- `prompt:` — the bundle from Step 1 + the role-specific prompt below

Each subagent gets a **distinct perspective** to play to the model's strengths. This is intentional — three identical prompts would just give you three slightly different copies of the same review.

#### Subagent A — Opus (Correctness & Logic)

> You are reviewing a code change for **correctness, logic, and subtle bugs**. Focus on: race conditions, off-by-ones, null/error handling gaps, incorrect invariants, broken contracts between callers and callees, regressions in edge cases. Trace through the diff thinking adversarially: what input would break this? Do not flag style or naming issues — other reviewers cover those. Respect [output schema](#subagent-output-schema). Return at most 15 findings, ranked by severity.

#### Subagent B — Sonnet (Architecture & Design)

> You are reviewing a code change for **architectural fit and design quality**. Focus on: layering violations, leaky abstractions, coupling that will hurt later, naming that misleads, comments that lie, premature abstractions, missed abstractions. Are the changes consistent with the surrounding code's conventions (use CLAUDE.md and README context)? Do not flag micro-bugs that don't change the design — Opus covers those. Respect [output schema](#subagent-output-schema). Return at most 15 findings, ranked by severity.

#### Subagent C — Haiku (Surface Defects & Hygiene)

> You are reviewing a code change for **surface-level defects and hygiene**. Focus on: obvious bugs, security smells (hardcoded secrets, SQL string concat, unescaped HTML, shell injection), missing tests for new branches, dead code, unused imports, debug prints / `console.log`, broken formatting. Be exhaustive on the surface — skim wide. Respect [output schema](#subagent-output-schema). Return at most 20 findings.

#### Subagent output schema

Every subagent must reply in **exactly** this markdown structure. Tell them so explicitly in the prompt:

```markdown
## SUMMARY
<2–4 sentences: what the change does, overall impression, biggest risk>

## VERDICT
<one of: APPROVE | APPROVE_WITH_NITS | REQUEST_CHANGES | BLOCK>

## FINDINGS

### F1 [severity: critical|high|medium|low|info] [category: bug|security|perf|design|style|test|docs]
**File:** path/to/file.ext:LINE (or :LINE-LINE for ranges)
**Issue:** <one sentence>
**Detail:** <2–5 sentence body explaining the problem and why it matters>
**Suggestion:** <concrete fix or "n/a" if exploratory>

### F2 ...
```

If a subagent fails to follow the schema, parse what you can and note the deviation in the final report — do not re-dispatch (wasteful).

### Step 2.5 — Checagem cruzada entre PRs abertas (determinística, sem subagent)

**Review de PR individual não pega colisão entre PRs.** Nenhuma das três lentes vê o que está fora
do diff que recebeu. Em uma rodada real, com 17 PRs abertas e treze nascidas na mesma manhã, duas
colisões passaram por um ensemble completo e só apareceram quando alguém comparou as PRs entre si.

Rode esta etapa **sempre que houver mais de uma PR aberta tocando o mesmo arquivo**. É determinística
— `gh` e `grep`, sem modelo — então é barata e não consome cota.

```bash
# 1. arquivos tocados por esta PR
gh pr diff <N> --name-only

# 2. quais outras PRs abertas tocam os mesmos arquivos
for p in $(gh pr list --state open --json number --jq '.[].number'); do
  [ "$p" = "<N>" ] && continue
  comm -12 <(gh pr diff <N> --name-only | sort) <(gh pr diff $p --name-only | sort) \
    | sed "s/^/PR $p compartilha: /"
done
```

Para cada par com arquivo em comum, verifique **três classes de colisão**:

| Classe | Como detectar | Por que git não avisa |
|---|---|---|
| **Símbolo duplicado** | Extrair `public function`, `class`, `const` adicionados nos dois diffs e interseccionar | Se os blocos forem colados em posições diferentes do arquivo, o merge textual passa — e o PHP dá `Cannot redeclare` |
| **Literal de fixture** | Uma PR adiciona um valor à fixture; outra usa o mesmo literal numa asserção de unicidade/ausência | **Arquivos diferentes.** Zero conflito textual, e as duas passam isoladas |
| **Instrução de merge obsoleta** | Ler a seção de "ordem de merge" do corpo de cada PR e conferir se ela cita todas as PRs concorrentes | Texto em prosa; nada valida |

```bash
# símbolos adicionados por PR, para interseccionar
gh pr diff <N> | awk '/^\+\+\+ b\//{f=$3} /^\+ *(public|private|protected) function/ {print f, $0}'
```

**A terceira classe é a mais perigosa** porque é ativa: uma PR pode carregar no corpo uma instrução
que, seguida à risca por quem for mergear, quebra a suíte. Foi o caso de uma PR cuja frase "sem
colisão de nome, manter os dois blocos" tinha sido escrita sobre outra PR.

Reporte os achados desta etapa numa seção própria do relatório — **eles não pertencem a nenhuma
lente**, porque nenhuma lente podia tê-los visto.

⚠️ **Sem CI no repositório, esta etapa é a única barreira.** Se o projeto não tiver workflow que
rode a suíte em PR, diga isso explicitamente no relatório: significa que colisão só é descoberta
depois do merge, em produção.

### Step 3 — Aggregate

Cross-reference findings across the three subagent outputs. A finding "matches" another if **any two of these** are true:

1. Same file (path equality, case-insensitive).
2. Line range overlaps OR is within ±5 lines.
3. Issue/Detail text covers the same concept (semantic match — you, the orchestrator, judge this).

Classify each cluster:

- **CONVERGENT** — 2 or 3 models flagged it. High confidence. Surface prominently.
- **UNIQUE** — only one model flagged it. Lower confidence but often the most interesting (catches what others missed). Keep but mark.
- **DIVERGENT** — one model flags as critical, another explicitly says it's fine (e.g. "the error handling here is correct" vs. "missing error handling"). Surface for human judgment.

#### Calibração — o que o consenso significa, e o que não significa

Regra derivada de evidência externa e de uma rodada real em 05/08/2026 (ver "Lições" abaixo):

> **Consenso é sinal de alta precisão. Discordância não é sinal de nada.**

Na prática:
- **CONVERGENT → tratar como bloqueante**, sem gastar revisor humano confirmando.
- **UNIQUE → fila de triagem barata.** Não é ruído (frequentemente é o achado mais interessante),
  mas não merece a atenção do revisor escasso antes de passar por um filtro rápido.
- **DIVERGENT → não tente desempatar por votação.** Discordância entre modelos não carrega
  informação sobre quem está certo. Desempate só por **medição** (query, comando, teste), nunca
  por rodar mais um modelo.

⚠️ **O consenso tem um modo de falha próprio: os modelos convergem no mesmo erro plausível**
("popularity trap" — há estudo mostrando que ensemble simples pode *piorar* F1 porque o segundo
modelo adiciona falso positivo em vez de detecção nova). Consenso eleva a confiança de que o
achado é **real**, não de que a **explicação causal** dele está certa.

**Consequência para o design da rodada:** ensemble só agrega valor quando os agentes têm acesso a
**evidência diferente** — banco, `gh`, logs, execução de teste — e não quando todos opinam sobre o
mesmo texto. Ao despachar, prefira dar a cada lente uma **fonte de verificação distinta** a dar a
todos o mesmo diff.

Severity for a cluster = max of contributing severities.

### Step 4 — Emit report

Write the final report to **`.local-ultrareview/report-<UTC-timestamp>.md`** at the repo root. Create `.local-ultrareview/` if needed and add it to `.gitignore` if not already excluded.

Report structure:

```markdown
# local-ultrareview — <scope summary>

**Run:** <timestamp UTC> · **Scope:** <branch/PR/path> · **Diff:** <N files, +M/-K lines>
**Models:** Opus 4.7, Sonnet 4.6, Haiku 4.5

## Executive summary

<3–5 bullets — the most important things the user needs to know>

## Verdicts

| Model | Verdict |
|---|---|
| Opus | <verdict> |
| Sonnet | <verdict> |
| Haiku | <verdict> |

## Convergent findings (high confidence)

For each: title, severity, file:line, which models flagged it, condensed description, suggested fix.

## Unique findings

Grouped by model. For each: same fields. Flag with a note that only one model raised it.

## Divergent findings

For each: the disagreement, each model's stance, your synthesis ("I lean toward X because…").

## Raw per-model reports

Collapsed sections (`<details>`) with each subagent's full output verbatim — preserved for audit.
```

After writing the file, print a **terse summary in chat**: file count, verdicts, top 3 convergent findings, path to the full report. Do not paste the entire report — the user opens the file.

---

## Operational notes

- **Always parallel dispatch.** One message, three `Agent` calls. Anything else is a bug.
- **Never re-dispatch on minor schema misses.** Parse leniently. Re-dispatch only if a subagent failed outright (empty result, crash).
- **Cost is not a concern** for this user (high-usage plan). Do not optimize prompts to shorten subagent inputs — fidelity > token count.
- **Token budget per subagent.** Pass full diff up to ~1500 lines. Beyond that, ask before proceeding.
- **Don't suggest commits.** This skill produces a report. The user decides what to act on.
- **The orchestrator is whoever is reading this skill** — typically Opus or Sonnet. The orchestrator does *not* review the code itself; it only dispatches, aggregates, and writes the report. (Reviewing in the orchestrator on top of three subagent reviews is double-counting and biases the convergent set.)

---

## Lições de uma rodada real — 05/08/2026

Auditoria de uma resposta técnica que ia para a liderança de engenharia. **Nove agentes no total;
sete afirmações caíram**, todas com a mesma assinatura: **medição certa, inferência causal errada.**

O que aprendemos, e que vale aplicar a toda rodada:

1. **O que derrubou as afirmações foi medir, não votar.** Dois modelos concordaram na tese central
   e os dois estavam errados. O que a derrubou foi um terceiro ir buscar a distribuição real nos
   dados — mais de 99% dos registros contradiziam a tese em uma query.
2. **"O código existe" ≠ "o caminho funciona na operação".** O mecanismo auditado existia e estava
   correto; tinha sido mergeado dias antes e fora exercitado poucas vezes na história do produto.
   **Sempre que um achado depender de um caminho de código, meça a frequência real de uso dele.**
3. **Negativa confiante de agente errou 3 vezes** ("não existe card X", "a pasta Y não existe").
   Verificar antes de agir sobre ausência — especialmente antes de **criar** algo que já existe.
4. **Entrega perdida por protocolo.** Na primeira rodada, 2 de 5 pareceres nunca chegaram porque os
   agentes emitiram texto final em vez de `SendMessage`. Deixe a instrução de entrega explícita e
   no topo do prompt; na segunda rodada, com isso corrigido, foram 5 de 5.
5. **Dê lentes com fontes diferentes.** As lentes que produziram achado decisivo foram as que
   tinham acesso a algo que as outras não tinham: dados reais de produção, `gh`, o histórico da
   discussão. As que só liam o mesmo diff convergiram entre si — inclusive no erro.

6. **Antes de tratar duas revisões como contraditórias, compare o escopo delas.** Em uma rodada
   posterior, uma revisão aprovou um PR sem ressalva e outra achou nele um problema grave — não era
   contradição: a segunda tinha um PR dependente no escopo e a primeira não. Revisões que cobrem
   conjuntos diferentes se **complementam**; só há contradição real quando as duas viram exatamente
   o mesmo material.

## Failure modes & fixes

| Symptom | Fix |
|---|---|
| Subagent returns prose, no `## FINDINGS` section | Parse what you can; mark as schema-deviant in raw section. |
| One subagent times out | Continue with 2; downgrade "convergent" threshold to "≥1 of 2". Note in report header. |
| Diff is empty | Abort early; tell the user there is nothing to review. |
| Repo is not a git repo | Abort; offer `git init`. |
| `gh pr diff` fails (not authenticated) | Tell the user to run `gh auth login`; do not silently fall back to a different scope. |

---

## Example invocations

```
/local-ultrareview
→ Review current branch (HEAD vs origin/HEAD's default) + working tree.

/local-ultrareview 1234
→ Review PR #1234 of the current repo (via gh).

/local-ultrareview src/payments/
→ Review pending changes touching src/payments/ only.

/local-ultrareview --all-staged
→ Review what's `git diff --staged`.
```

---

## When NOT to use this skill

- Reviewing a single line / one-character fix. Just eyeball it.
- Asking "is this approach right?" before code exists. That's a design conversation, not a review.
- Looking for security audit in isolation — use `/security-review` instead; this skill is general-purpose.
- Reviewing generated code (migrations, codegen output) — too noisy; the signal is in the inputs, not the output.
