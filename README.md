# local-ultrareview

Skill local para Claude Code que roda **review de código em paralelo nos três modelos da família Claude 4.X** (Opus 4.7, Sonnet 4.6, Haiku 4.5) e agrega os achados num único relatório, distinguindo o que **convergiu** (alta confiança) do que apareceu em apenas um modelo (sinal mais fraco, mas frequentemente o mais interessante).

Alternativa caseira ao `/ultrareview` oficial — roda 100% local via subagentes do Claude Code, sem cloud fleet, sem custo extra além do plano.

## Como funciona

1. Você invoca `/local-ultrareview` (com ou sem argumento).
2. A skill coleta o diff alvo (branch atual, PR, ou path).
3. Dispara **três subagentes em paralelo**, cada um num modelo diferente, com perspectivas complementares:
   - **Opus** — correção e lógica (bugs sutis, race conditions, edge cases).
   - **Sonnet** — arquitetura e design (acoplamento, abstrações, convenções do projeto).
   - **Haiku** — defeitos de superfície (segurança óbvia, dead code, falta de testes).
4. Cruza os retornos: agrupa achados por arquivo+linha+conceito e classifica em **convergente** (2+ modelos), **único** (1 modelo) ou **divergente** (modelos discordam).
5. Escreve o relatório em `.local-ultrareview/report-<timestamp>.md` no repo alvo e imprime um resumo curto no chat.

## Instalação

A skill é um arquivo `SKILL.md` standalone. O Claude Code descobre skills em alguns caminhos. Escolha um:

### Opção 1 — Skill global do usuário

```bash
mkdir -p ~/.claude/skills/local-ultrareview
cp SKILL.md ~/.claude/skills/local-ultrareview/SKILL.md
```

Fica disponível em todos os projetos.

### Opção 2 — Skill do projeto

```bash
mkdir -p <repo>/.claude/skills/local-ultrareview
cp SKILL.md <repo>/.claude/skills/local-ultrareview/SKILL.md
```

Versionada junto do projeto.

### Opção 3 — Plugin (se você usa marketplace local)

Empacote como plugin com `.claude-plugin/plugin.json` e adicione a um marketplace.

## Uso

| Comando | O que faz |
|---|---|
| `/local-ultrareview` | Review do branch atual (vs default) + working tree |
| `/local-ultrareview 123` | Review do PR #123 do repo atual (via `gh`) |
| `/local-ultrareview src/foo/` | Review limitado ao path |
| `/local-ultrareview --all-staged` | Apenas `git diff --staged` |

## Pré-requisitos

- Claude Code com acesso aos 3 modelos (Opus, Sonnet, Haiku).
- `git` no PATH.
- `gh` CLI autenticado se for revisar PRs.

## Por que ensemble

Um modelo só tem pontos cegos consistentes. Modelos de tamanhos diferentes têm pontos cegos *diferentes*:

- Haiku é rápido e cobre superfície ampla, mas raciocina raso.
- Sonnet equilibra profundidade e velocidade, é forte em design.
- Opus raciocina mais fundo, é melhor para correção lógica adversarial.

Cruzar os três dá um sinal mais confiável que qualquer passada única — e os achados onde os três discordam costumam ser exatamente os pontos em que você quer parar e pensar.

## Saída

Cada execução salva um relatório em markdown em `.local-ultrareview/report-<timestamp>.md` (gitignorado por padrão), contendo:

- Resumo executivo (3–5 bullets).
- Veredito de cada modelo (`APPROVE | APPROVE_WITH_NITS | REQUEST_CHANGES | BLOCK`).
- **Achados convergentes** (alta confiança).
- **Achados únicos** (por modelo).
- **Achados divergentes** (com síntese do orquestrador).
- Reports brutos de cada subagente em `<details>` colapsáveis.

## Limitações conhecidas

- Diff > 1500 linhas: a skill pede confirmação antes de prosseguir (qualidade cai com noise alto).
- Não revisa código gerado (migrations, codegen) — abrir como issue se quiser endurecer isso.
- O orquestrador (modelo que executa a skill) **não** adiciona suas próprias opiniões além da agregação, pra evitar double-counting que enviesa o "convergente".

## Licença

MIT. Veja `LICENSE`.
