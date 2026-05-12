# Agent Skills Pattern — Roteamento Regex-First

> **Escopo deste documento.** Esta é uma especialização do pattern Agent Skills voltada para **máxima previsibilidade e mínima latência**: o roteamento de skills é feito por **regex/keyword matching determinístico**, sem LLM e sem embeddings na decisão. O LLM continua sendo o "executor" da skill (raciocínio, geração de resposta, tool use), mas **não decide qual skill ativar**.
>
> Este documento é **self-contained**: pode ser lido sem o doc geral. Onde houver paridade total com o pattern híbrido, é apontado para fins de cross-reference.
>
> **Quando usar este pattern.** Domínios fechados, vocabulário previsível, baixa tolerância a latência, ambiente regulado onde determinismo total importa mais que flexibilidade conversacional. Exemplos clássicos: jornadas bancárias estruturadas, atendimento operacional, fluxos de cobrança, suporte L1.
>
> **Quando NÃO usar.** Conversas abertas, domínios criativos, agentes coder, casos em que o usuário descreve a intenção em linguagem muito variada. Aí o híbrido (regex + LLM fallback) ou full-LLM faz mais sentido.

---

## Sumário

1. [Por que regex-first (e o que isso muda)](#1-por-que-regex-first-e-o-que-isso-muda)
2. [Conceitos centrais](#2-conceitos-centrais)
   - [2.1 Determinismo total como princípio](#21-determinismo-total-como-princípio)
   - [2.2 Quando o LLM ainda entra](#22-quando-o-llm-ainda-entra)
   - [2.3 Multi-stage matching](#23-multi-stage-matching)
   - [2.4 ADR — escolha da engine de regex (re2 vs re vs hyperscan)](#24-adr--escolha-da-engine-de-regex-re2-vs-re-vs-hyperscan)
3. [Anatomia de uma Skill (com triggers regex)](#3-anatomia-de-uma-skill-com-triggers-regex)
4. [Padrão AKU adaptado para regex-first](#4-padrão-aku-adaptado-para-regex-first)
5. [Estrutura de diretórios do template](#5-estrutura-de-diretórios-do-template)
6. [Implementação em LangGraph](#6-implementação-em-langgraph)
7. [Exemplos completos de skills](#7-exemplos-completos-de-skills)
8. [Engenharia de regex para NLU](#8-engenharia-de-regex-para-nlu)
9. [Tool gating, governança e segurança](#9-tool-gating-governança-e-segurança)
10. [Testes, evals e observabilidade](#10-testes-evals-e-observabilidade)
11. [Boas práticas e anti-patterns](#11-boas-práticas-e-anti-patterns)
12. [Roadmap de evolução do template](#12-roadmap-de-evolução-do-template)
13. [Stack de mercado e quando subir de nível](#13-stack-de-mercado-e-quando-subir-de-nível)

---

## 1. Por que regex-first (e o que isso muda)

### 1.1 O problema que motiva esta variante

No pattern híbrido (doc principal), o LLM é o roteador de skills em última instância. Isso paga **um round-trip ao LLM** por turno onde a skill ainda não está ativa. Mesmo com short-circuit via keyword matching, há cenários em que mesmo essa latência adicional incomoda:

- **Banking ops em horário de pico** com SLA agressivo de TTFT.
- **Voice agents** onde 200ms de latência adicional já é perceptível.
- **Fluxos de cobrança e suporte L1** com vocabulário muito previsível e milhões de turnos/dia, onde até 80ms × volume vira problema de custo de infra.
- **Compliance regulatório** que exige reprodutibilidade bit-a-bit do roteamento.

### 1.2 O pattern regex-first em uma frase

> **A decisão de qual skill ativar é uma função pura `f(input) → skill_name`, executada por regex compilado em tempo de boot, em <1ms, sem chamadas a modelo nenhum.**

O LLM continua sendo essencial para **executar** o passo a passo da skill — mas a **seleção** da skill é feita antes de o LLM ver o input.

### 1.3 Comparação direta

| Aspecto | LLM-router (puro) | Híbrido (keyword + LLM) | **Regex-first (este doc)** |
|---|---|---|---|
| Decisão de skill | LLM | Keyword → LLM fallback | **Regex puro** |
| Latência adicional | +1.5–2s | +5–80ms (com fallback ~10%) | **<1ms** |
| Determinismo | Baixo | Alto | **100%** |
| Reprodutibilidade | Estatística | Alta | **Bit-a-bit** |
| Tolerância a linguagem livre | Alta | Média | Baixa |
| Custo de manutenção | Baixo (LLM aprende) | Médio | **Alto (cada padrão é manual)** |
| Auditabilidade | Trace LLM | Trace híbrido | **Match log determinístico** |

### 1.4 O que muda na arquitetura

Comparando com o pattern híbrido, três peças mudam:

1. **Frontmatter da skill ganha um bloco `triggers` rico**, com regex priorizados, padrões de exclusão e captura de entidades inline.
2. **Router determinístico** (`RegexRouter`) substitui o caminho LLM como rota padrão; o LLM-fallback é **opcional** (e em muitos domínios pode ser desligado completamente).
3. **CI ganha gates fortes**: coverage de regex, false-positive rate, fuzz testing.

O restante (tool gating, governança, SKILL.md como contrato, audit log, subagentes) **fica idêntico**.

---

## 2. Conceitos centrais

### 2.1 Determinismo total como princípio

A regra de ouro do pattern regex-first é:

> **Mesma frase do usuário, mesma skill ativada — sempre, em qualquer instância, em qualquer momento.**

Isso não é um detalhe técnico — é a justificativa do pattern existir. Você troca flexibilidade por:

- **Auditoria perfeita.** O log diz exatamente que regex casou e por quê.
- **Reprodução de bugs.** Um issue de produção é reproduzível com `RegexRouter.route(input)` numa sessão Python local.
- **A/B testing sem viés de modelo.** Você muda regex, mede impacto, pronto.
- **SLA de latência previsível.** P99 do roteamento é cravado em milissegundos.

### 2.2 Quando o LLM ainda entra

Mesmo em pattern regex-first puro, o LLM continua sendo central — só não decide qual skill. Ele continua responsável por:

- **Executar o workflow da skill** (raciocinar entre passos, decidir qual tool chamar dentro do escopo gated).
- **Gerar a resposta natural ao usuário.**
- **Pedir clarificação** quando precisa.
- **Lidar com situações imprevistas** dentro da skill ativa.

O que sai do LLM é apenas o **routing layer**.

### 2.3 Multi-stage matching

Regex puro não significa "uma única expressão monolítica". O pattern usa **múltiplos estágios**, em ordem de custo crescente:

```
Estágio 1 — Normalização (~50µs)
    lowercase, strip acentos, normalizar números/moeda, contrações

Estágio 2 — Fast path: exact match em hash set (~10µs)
    frases canônicas ("contestar pix", "segunda via cartão")

Estágio 3 — Aho-Corasick: multi-keyword scan (~100µs para 10k keywords)
    matching simultâneo de centenas/milhares de keywords

Estágio 4 — Regex compilado por skill, em ordem de priority (~1ms total)
    padrões com captura de entidades, lookahead, etc.

Estágio 5 — (opcional) LLM fallback se nada casou e domínio não é fechado
```

Em domínios fechados (banco operacional), os estágios 1–4 cobrem ~95% dos turnos. Estágio 5 vira opcional.

### 2.4 ADR — escolha da engine de regex (re2 vs re vs hyperscan)

> **Contexto.** A escolha da engine de regex em Python tem implicações fortes em segurança (catastrophic backtracking), latência (P99) e expressividade (features suportadas). Este ADR registra a decisão padrão do template.

**Opções em jogo:**

| Engine | Tipo | Backtracking? | Lookahead/Lookbehind? | Backreferences? | Linear time? | Multi-pattern? |
|---|---|---|---|---|---|---|
| `re` (stdlib) | NFA com backtracking | Sim (perigoso) | Sim | Sim | **Não** | Não |
| `regex` (PyPI) | NFA estendida | Sim | Sim (full) | Sim | Não | Não |
| `re2` (`google-re2`) | DFA/NFA, sem backtracking | **Não** | Limitado (apenas non-capturing) | **Não** | **Sim** | Não |
| `hyperscan` | DFA compilado SIMD | Não | Limitado | Não | **Sim** | **Sim (massivo)** |

**Decisão padrão do template: `re2` (via `google-re2`)**, com escape para `hyperscan` quando o catálogo passa de ~1000 patterns ou o volume passa de ~10k req/s por instância.

**Razões.**

1. **Eliminação de catastrophic backtracking.** Em `re`/`regex`, padrões como `(a+)+b` contra `aaaaaaaaaaaa!` podem rodar por minutos e travar o agente. Em ambiente bancário, isso é vetor de DoS. `re2` é **garantidamente linear** no tamanho do input.

2. **Latência previsível.** Sem backtracking, o P99 do roteamento é função apenas do tamanho do input e do número de patterns — não do conteúdo. Você consegue cravar SLA.

3. **Compatibilidade suficiente.** Os recursos que `re2` não suporta (backreferences `\1`, lookahead/lookbehind com captura) **raramente são necessários para NLU** — quando aparecem, geralmente são sinal de regex mal escrito.

4. **Não trava o GIL excessivamente.** `re2` é C++, libera GIL durante matching, escala melhor com threads.

**Quando subir para `hyperscan`.**

- Catálogo > 1000 patterns simultâneos.
- Necessidade de matching multi-pattern em uma única passada (Aho-Corasick generalizado).
- Hot path com >10k req/s por instância.

**Quando voltar para `re` (stdlib).**

- Skills experimentais em sandbox local.
- Patterns que **realmente** precisam de backreferences ou lookahead com captura, e você não consegue refatorar.
- Em todos os casos: **com timeout obrigatório** (`signal.alarm` ou execução em thread separada com kill).

**Implicações no código.**

```python
# src/agent/skills/regex_engine.py
try:
    import re2 as _re_engine
    ENGINE = "re2"
except ImportError:
    import re as _re_engine
    ENGINE = "re"
    import warnings
    warnings.warn(
        "google-re2 não disponível, usando 're' stdlib. "
        "Em produção, instale google-re2 para evitar catastrophic backtracking.",
        RuntimeWarning,
    )

def compile_pattern(pattern: str):
    return _re_engine.compile(pattern)
```

---

## 3. Anatomia de uma Skill (com triggers regex)

A skill continua sendo um diretório com um `SKILL.md` no centro, frontmatter YAML + markdown. **A diferença está no bloco `triggers`**, que aqui é first-class citizen.

### 3.1 Frontmatter estendido

```yaml
---
name: pix-disputa-med
description: >
  Conduz contestação de Pix via MED (Mecanismo Especial de Devolução).
  Usado quando o usuário relata Pix indevido, fraude ou erro operacional.

# ---- ROUTING (regex-first) ----
priority: 80                        # 0-100; maior vence em empate
triggers:
  # Estágio 2: exact match (mais rápido, mais explícito)
  exact:
    - "contestar pix"
    - "abrir med"
    - "devolver pix recebido"
    - "estornar pix"

  # Estágio 3: keywords individuais (Aho-Corasick)
  keywords:
    - "med"
    - "mecanismo especial devolução"
    - "pix indevido"
    - "golpe pix"
    - "fraude pix"
    - "pix errado"
    - "pix por engano"

  # Estágio 4: regex compilados (re2-safe), em ordem de prioridade
  regex:
    - pattern: '\bcontestar\s+(?:um\s+|o\s+)?pix\b'
      priority: 95
      captures: []
    - pattern: '\bpix\s+(?:de\s+)?(?P<valor>r\$?\s*[\d.,]+)\s+(?:que|errad)'
      priority: 90
      captures: ['valor']
    - pattern: '\b(?:caí|cair)\s+(?:em\s+|num\s+|n[oa]\s+)?golpe\b'
      priority: 85
      captures: []

  # Padrões que DESQUALIFICAM esta skill (mesmo se um dos acima casar)
  exclude:
    - pattern: '\bpix\s+agendad'      # agendamento, não disputa
    - pattern: '\bnão\s+(?:cheguei|consegui)\s+(?:a\s+)?contestar'  # negação

allowed-tools:
  - consulta_transacao_pix
  - abre_chamado_med
  - notifica_destinatario
  - escala_humano

domain: pagamentos.pix
owner: squad-pix-disputas
governance:
  requires_human_approval: false
  escalation_threshold_brl: 50000
  audit_log: true
version: 3.0.0
---

# Skill: Disputa de Pix via MED
...
```

### 3.2 Campos do bloco `triggers` — referência

| Campo | Estágio | Propósito | Performance |
|---|---|---|---|
| `exact` | 2 | Frases canônicas, match exato após normalização | ~10µs |
| `keywords` | 3 | Substring matching, escaneado em paralelo via Aho-Corasick | ~100µs para 10k keywords |
| `regex[].pattern` | 4 | Regex compilado (re2-safe), pode capturar entidades | ~1ms para 100 patterns |
| `regex[].priority` | 4 | Override de prioridade por padrão (sobrescreve priority da skill) | — |
| `regex[].captures` | 4 | Named groups que viram entidades no estado do agente | — |
| `exclude[].pattern` | 4 | Se casar, **desqualifica** a skill, mesmo que outros padrões tenham casado | — |

### 3.3 Como o `priority` é resolvido

A regra de tiebreak, em ordem:

1. **`exclude` casou?** → skill descartada, independente do resto.
2. **Estágio mais alto que casou.** Exact > Keyword > Regex.
3. Dentro do mesmo estágio: **maior `priority`** vence.
4. Empate no priority: **maior cobertura** (% de caracteres do input cobertos pelos matches).
5. Empate na cobertura: **mais recente em `version`** (skill mais nova vence — útil para A/B).
6. Empate total: **erro de configuração**, lint do CI falha.

Resultado: **não há ambiguidade em runtime**. Toda decisão é determinística e explicável.

---

## 4. Padrão AKU adaptado para regex-first

Os 7 elementos da Atomic Knowledge Unit se mantêm. O que muda é o **elemento 1 (Intent)**, que agora é expresso formalmente em `triggers` (não em prosa na `description`):

| # | Elemento | Onde no SKILL.md (regex-first) |
|---|---|---|
| 1 | **Intent** | `triggers.exact` + `triggers.keywords` + `triggers.regex` (formal); `description` (humano) |
| 2 | **Conhecimento procedural** | Seções "Workflow" e "Anti-patterns" |
| 3 | **Tool bindings** | `allowed-tools` + seção "Tool bindings" |
| 4 | **Metadados organizacionais** | `owner`, `domain` |
| 5 | **Governança** | `governance` + seção "Escalação" |
| 6 | **Continuação** | `next_skills`, `requires_skills` |
| 7 | **Validadores** | `validators/` + `test_triggers.yaml` (dataset de eval) |

> **Consequência prática.** Em regex-first, a `description` deixa de ser o "gatilho do LLM" e passa a ser **documentação humana**. Compliance e produto leem `description`; o roteamento usa `triggers`. Isso libera você para escrever descrições mais ricas, sem se preocupar em "agradar" o LLM-router.

---

## 5. Estrutura de diretórios do template

```
my-agent/
├── pyproject.toml
├── README.md
├── .env.example
│
├── src/
│   └── agent/
│       ├── __init__.py
│       ├── graph.py
│       ├── state.py
│       ├── nodes.py
│       ├── prompts/
│       │   └── system_base.md
│       ├── tools/
│       │   ├── registry.py
│       │   └── pix.py
│       ├── skills/
│       │   ├── __init__.py
│       │   ├── loader.py            # parser de SKILL.md
│       │   ├── registry.py          # SkillRegistry
│       │   ├── regex_engine.py      # wrapper re2/re
│       │   ├── normalizer.py        # normalização do input
│       │   ├── regex_router.py      # ⭐ peça central deste pattern
│       │   ├── ahocorasick.py       # wrapper pyahocorasick
│       │   ├── middleware.py        # SkillsMiddleware
│       │   └── observability.py     # logs estruturados de match
│       └── config.py
│
├── skills/
│   ├── pix-envio/
│   │   ├── SKILL.md
│   │   └── test_triggers.yaml       # dataset de eval por skill
│   ├── pix-disputa-med/
│   │   ├── SKILL.md
│   │   ├── REFERENCE.md
│   │   └── test_triggers.yaml
│   ├── cartao-segunda-via/
│   │   ├── SKILL.md
│   │   └── test_triggers.yaml
│   └── _shared/
│       ├── kyc-validacao/
│       └── escala-humano/
│
├── tests/
│   ├── test_normalizer.py
│   ├── test_regex_router.py
│   ├── test_skill_triggers.py       # roda todos test_triggers.yaml
│   ├── test_no_catastrophic_regex.py # fuzz contra patterns lentos
│   └── evals/
│       └── routing_accuracy.py
│
└── scripts/
    ├── new_skill.py
    ├── lint_skills.py               # valida triggers, priority, re2-safety
    ├── benchmark_router.py          # P50/P95/P99 do roteamento
    └── coverage_report.py           # % de inputs cobertos por triggers
```

---

## 6. Implementação em LangGraph

### 6.1 Estado

```python
# src/agent/state.py
from typing import Annotated, Optional
from typing_extensions import TypedDict
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages


class AgentState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]

    # Skill ativa na sessão (pinning)
    active_skill: Optional[str]

    # Histórico de skills ativadas
    skill_history: list[str]

    # Tools liberadas (None = todas; lista = gated)
    allowed_tool_names: Optional[list[str]]

    # Entidades extraídas pelos triggers (named captures)
    entities: dict

    # Auditoria de routing
    routing_method: Optional[str]       # "exact" | "keyword" | "regex" | "pinned" | "llm_fallback"
    routing_match: Optional[dict]       # detalhe do match (qual padrão, qual skill, qual capture)

    governance: dict
```

### 6.2 Normalização do input

A normalização é determinística e idempotente. Roda **antes** de qualquer matching.

```python
# src/agent/skills/normalizer.py
import unicodedata
import re

# Pré-compilados no import (zero custo por chamada)
_WS = re.compile(r"\s+")
_PUNCT = re.compile(r"[^\w\s\-\$,.]")
_NUMERIC_RUN = re.compile(r"(\d[\d.,]*)")
_CONTRACTIONS = {
    "pra": "para",
    "pro": "para o",
    "tava": "estava",
    "ta": "está",
    "vc": "você",
    "tb": "também",
    "msg": "mensagem",
    "qq": "qualquer",
}


def normalize(text: str) -> str:
    """Normalização determinística do input do usuário.

    Garante que 'Contestar PIX' e 'contestar pix' viram a mesma string
    antes do matching. Idempotente: normalize(normalize(x)) == normalize(x).
    """
    # 1) lowercase
    t = text.lower()
    # 2) remove acentos (NFD + filtra combining marks)
    t = unicodedata.normalize("NFD", t)
    t = "".join(c for c in t if unicodedata.category(c) != "Mn")
    # 3) expansão de contrações comuns em PT-BR
    words = []
    for w in t.split():
        words.append(_CONTRACTIONS.get(w, w))
    t = " ".join(words)
    # 4) remove pontuação irrelevante, preserva $ , . - (úteis em valores)
    t = _PUNCT.sub(" ", t)
    # 5) colapsa whitespace
    t = _WS.sub(" ", t).strip()
    return t
```

**Princípio:** normalização deve ser idêntica em **boot-time** (quando compila triggers) e **runtime** (quando roteia). Isso é garantido aplicando a mesma função em ambos.

### 6.3 Compilação eager (boot-time)

```python
# src/agent/skills/regex_router.py — parte 1: build
from dataclasses import dataclass, field
from typing import Optional
import ahocorasick

from .regex_engine import compile_pattern
from .normalizer import normalize


@dataclass
class CompiledRegex:
    skill_name: str
    pattern_str: str
    compiled: object
    priority: int
    captures: list[str] = field(default_factory=list)


@dataclass
class CompiledExclude:
    skill_name: str
    compiled: object


class RegexRouter:
    """Roteador determinístico multi-estágio.

    Compila TUDO no __init__. Em runtime, route() é só matching.
    """

    def __init__(self, registry):
        self.registry = registry

        # Estágio 2: exact match — dict { frase_normalizada -> (skill, priority) }
        self._exact: dict[str, tuple[str, int]] = {}

        # Estágio 3: Aho-Corasick — keyword → lista de (skill, priority)
        self._ac = ahocorasick.Automaton()
        self._kw_index: dict[str, list[tuple[str, int]]] = {}

        # Estágio 4: regex compilados, agrupados por skill
        self._regex_by_skill: dict[str, list[CompiledRegex]] = {}
        self._excludes_by_skill: dict[str, list[CompiledExclude]] = {}

        self._build()

    def _build(self):
        for skill in self.registry.all():
            triggers = skill.triggers or {}
            base_priority = skill.priority or 50

            # exact
            for phrase in triggers.get("exact", []):
                key = normalize(phrase)
                if key in self._exact:
                    existing_skill, existing_prio = self._exact[key]
                    if existing_skill != skill.name:
                        raise ValueError(
                            f"Conflito de 'exact' trigger: '{phrase}' está em "
                            f"'{existing_skill}' e '{skill.name}'. Resolva manualmente."
                        )
                self._exact[key] = (skill.name, base_priority)

            # keywords (Aho-Corasick)
            for kw in triggers.get("keywords", []):
                key = normalize(kw)
                self._kw_index.setdefault(key, []).append((skill.name, base_priority))
                self._ac.add_word(key, (key, skill.name, base_priority))

            # regex
            for entry in triggers.get("regex", []):
                pattern_str = entry["pattern"]
                # validação: re2 compila? se não, é erro de config
                compiled = compile_pattern(pattern_str)
                self._regex_by_skill.setdefault(skill.name, []).append(
                    CompiledRegex(
                        skill_name=skill.name,
                        pattern_str=pattern_str,
                        compiled=compiled,
                        priority=entry.get("priority", base_priority),
                        captures=entry.get("captures", []),
                    )
                )

            # excludes
            for entry in triggers.get("exclude", []):
                compiled = compile_pattern(entry["pattern"])
                self._excludes_by_skill.setdefault(skill.name, []).append(
                    CompiledExclude(skill_name=skill.name, compiled=compiled)
                )

        self._ac.make_automaton()
```

### 6.4 Matching em runtime

```python
# src/agent/skills/regex_router.py — parte 2: route

@dataclass
class RouteResult:
    skill_name: str
    method: str                      # "exact" | "keyword" | "regex"
    matched_pattern: str
    priority: int
    coverage: float
    entities: dict                   # named captures

    def to_audit(self) -> dict:
        return {
            "skill": self.skill_name,
            "method": self.method,
            "pattern": self.matched_pattern,
            "priority": self.priority,
            "coverage": round(self.coverage, 3),
            "entities": self.entities,
        }


class RegexRouter:
    # ... __init__ acima ...

    def route(self, user_input: str) -> Optional[RouteResult]:
        normalized = normalize(user_input)
        if not normalized:
            return None

        candidates: list[RouteResult] = []

        # Estágio 2: exact match
        if normalized in self._exact:
            skill_name, prio = self._exact[normalized]
            if not self._is_excluded(skill_name, user_input):
                return RouteResult(
                    skill_name=skill_name,
                    method="exact",
                    matched_pattern=normalized,
                    priority=prio + 1000,  # exact sempre vence keyword/regex
                    coverage=1.0,
                    entities={},
                )

        # Estágio 3: keywords via Aho-Corasick
        kw_hits: dict[str, list[tuple[str, int]]] = {}  # skill -> [(kw, prio)]
        total_covered = 0
        for end_idx, (kw, skill_name, prio) in self._ac.iter(normalized):
            kw_hits.setdefault(skill_name, []).append((kw, prio))
            total_covered += len(kw)

        for skill_name, hits in kw_hits.items():
            if self._is_excluded(skill_name, user_input):
                continue
            # score: prioridade + bonus por keyword multi-palavra
            best_prio = max(prio + len(kw.split()) for kw, prio in hits)
            coverage = sum(len(kw) for kw, _ in hits) / max(len(normalized), 1)
            candidates.append(RouteResult(
                skill_name=skill_name,
                method="keyword",
                matched_pattern=",".join(kw for kw, _ in hits),
                priority=best_prio,
                coverage=coverage,
                entities={},
            ))

        # Estágio 4: regex
        for skill_name, patterns in self._regex_by_skill.items():
            if self._is_excluded(skill_name, user_input):
                continue
            for p in patterns:
                m = p.compiled.search(normalized)
                if m:
                    entities = {}
                    for cap in p.captures:
                        try:
                            entities[cap] = m.group(cap)
                        except (IndexError, KeyError):
                            pass  # group opcional não casou
                    coverage = (m.end() - m.start()) / max(len(normalized), 1)
                    candidates.append(RouteResult(
                        skill_name=skill_name,
                        method="regex",
                        matched_pattern=p.pattern_str,
                        priority=p.priority,
                        coverage=coverage,
                        entities=entities,
                    ))
                    break  # primeiro regex que casa naquela skill

        if not candidates:
            return None

        # Tiebreak determinístico:
        #   1) maior priority
        #   2) maior coverage
        #   3) maior version (resolvido fora, no registry)
        candidates.sort(
            key=lambda c: (c.priority, c.coverage),
            reverse=True,
        )
        return candidates[0]

    def _is_excluded(self, skill_name: str, raw_input: str) -> bool:
        for exc in self._excludes_by_skill.get(skill_name, []):
            if exc.compiled.search(raw_input.lower()):
                return True
        return False
```

### 6.5 Middleware

```python
# src/agent/skills/middleware.py
from langchain.agents.middleware import AgentMiddleware
from .observability import log_routing


class SkillsMiddleware(AgentMiddleware):
    """Aplica routing regex-first + tool gating a cada turno.

    Não chama LLM. A única exceção é se llm_fallback_enabled=True e
    nenhum trigger casa — aí o catálogo entra no system prompt e o LLM
    pode usar uma tool load_skill (mesmo comportamento do pattern híbrido).
    """

    def __init__(self, registry, all_tools, router, llm_fallback=False):
        self.registry = registry
        self.all_tools = all_tools
        self.router = router
        self.llm_fallback = llm_fallback

    def before_model(self, state, request):
        last_user = self._last_user_message(state)

        # ---- Camada 0: Pinning ----
        if state.get("active_skill"):
            if not self._should_unpin(state, last_user):
                return self._apply_skill(
                    state["active_skill"], request, method="pinned",
                )
            # vai despinnar
            state["active_skill"] = None
            state["allowed_tool_names"] = None

        # ---- Camada 1: RegexRouter ----
        result = self.router.route(last_user)
        if result:
            log_routing(state, result)
            state["active_skill"] = result.skill_name
            state["skill_history"] = state.get("skill_history", []) + [result.skill_name]
            state["routing_method"] = result.method
            state["routing_match"] = result.to_audit()
            state["entities"] = {**state.get("entities", {}), **result.entities}
            return self._apply_skill(result.skill_name, request, method=result.method)

        # ---- Camada 2 (opcional): LLM fallback ----
        if self.llm_fallback:
            request.system_prompt = request.system_prompt.replace(
                "{skill_catalog}", self.registry.catalog()
            )
            request.tools = self.all_tools
            state["routing_method"] = "llm_fallback"
            return request

        # ---- Sem fallback: prossegue sem skill ativa ----
        request.tools = self._default_tools()  # tools de "menu" ou onboarding
        state["routing_method"] = "none"
        return request

    def _apply_skill(self, name, request, method):
        skill = self.registry.get(name)
        request.system_prompt = (
            request.system_prompt
            + f"\n\n--- SKILL ATIVA: {skill.name} (v{skill.version}) ---\n{skill.body}"
        )
        request.tools = [t for t in self.all_tools if t.name in (skill.allowed_tools or [])]
        return request

    def _should_unpin(self, state, user_input: str) -> bool:
        UNPIN_TRIGGERS = ["esquece isso", "voltar ao menu", "outra coisa", "cancelar"]
        text = normalize(user_input)
        if any(t in text for t in UNPIN_TRIGGERS):
            return True
        # se outro regex casa skill diferente da pinned, troca
        other = self.router.route(user_input)
        if other and other.skill_name != state["active_skill"]:
            return True
        return False
```

### 6.6 Fluxo end-to-end

```
[user] "preciso contestar um pix de R$ 500 que mandei errado"
   │
   ▼
[normalize] → "preciso contestar um pix de r$ 500 que mandei errad"
   │
   ▼
[RegexRouter]
   • Estágio 2 (exact): "contestar pix" não bate sozinho (input mais longo)
   • Estágio 3 (AC): casa keywords ["pix", "errad", ...]
       candidato: pix-disputa-med, prio=80+coverage
   • Estágio 4 (regex):
       padrão '\bcontestar\s+(?:um\s+|o\s+)?pix\b' casa, prio=95
       padrão '\bpix\s+(?:de\s+)?(?P<valor>r\$?\s*[\d.,]+).*errad' casa, prio=90
       captures: valor="r$ 500"
   • Tiebreak: prio 95 vence
   │
   ▼
[middleware]
   • active_skill = "pix-disputa-med"
   • entities = {"valor": "r$ 500"}
   • allowed_tool_names = [consulta_transacao_pix, abre_chamado_med, ...]
   • system_prompt += corpo da SKILL.md
   │
   ▼
[LLM] — 1 call apenas
   • Lê workflow, vê entidade valor pré-extraída, decide próximo passo
   • Pergunta o end_to_end_id ou chama consulta_transacao_pix direto
```

**Total de LLM calls para roteamento: 0.** Latência adicionada por roteamento: <1ms.

---

## 7. Exemplos completos de skills

### 7.1 `pix-envio` — skill simples

```markdown
---
name: pix-envio
description: Realiza envio de Pix imediato para uma chave.
priority: 70
triggers:
  exact:
    - "enviar pix"
    - "fazer pix"
    - "pagar com pix"
    - "transferir por pix"
  keywords:
    - "mandar pix"
    - "fazer um pix"
    - "transferir pix"
  regex:
    - pattern: '\b(?:enviar|mandar|fazer|realizar)\s+(?:um\s+)?pix\b'
      priority: 80
    - pattern: '\bpix\s+(?:de\s+)?(?P<valor>r\$?\s*[\d.,]+)\s+(?:para|pra)\s+(?P<destino>.+)'
      priority: 85
      captures: ['valor', 'destino']
  exclude:
    - pattern: '\bcontestar\b'
    - pattern: '\bagendar?\b'
allowed-tools:
  - consulta_chave_pix
  - valida_saldo
  - executa_pix
  - notifica_usuario
domain: pagamentos.pix
owner: squad-pix-emissao
version: 2.0.0
---

# Skill: Envio de Pix

## Workflow
1. Se `entities.valor` e `entities.destino` já vieram do regex, pular para passo 3.
2. Solicitar chave Pix do destinatário.
3. Chamar `consulta_chave_pix(chave)` e CONFIRMAR o titular.
4. Solicitar/confirmar valor.
5. Chamar `valida_saldo(valor)`.
6. Chamar `executa_pix(chave, valor)`.
7. Notificar.
```

### 7.2 `pix-disputa-med` — skill com regex sofisticado

```markdown
---
name: pix-disputa-med
description: Conduz contestação de Pix via MED (Mecanismo Especial de Devolução).
priority: 85
triggers:
  exact:
    - "contestar pix"
    - "abrir med"
    - "estornar pix"
    - "devolver pix recebido"
  keywords:
    - "med"
    - "pix indevido"
    - "golpe pix"
    - "fraude pix"
    - "pix errado"
    - "caí em golpe"
    - "fui enganado"
  regex:
    # Verbos de contestação + pix
    - pattern: '\b(?:contestar|estornar|devolver|cancelar)\s+(?:um\s+|o\s+|esse\s+)?pix\b'
      priority: 95
    # Pix com valor + erro (entity extraction)
    - pattern: '\bpix\s+(?:de\s+)?(?P<valor>r\$?\s*[\d.,]+).*\b(?:errad|indevid|engan)'
      priority: 92
      captures: ['valor']
    # Relato de fraude/golpe
    - pattern: '\b(?:cai|cair|fui)\s+(?:em\s+|n[oa]\s+)?(?:golpe|fraude|enganad)'
      priority: 88
    # MED como sigla
    - pattern: '\bmed\b(?!ic)'  # "med" mas não "medico"
      priority: 80
  exclude:
    - pattern: '\bpix\s+agendad'
    - pattern: '\bnão\s+(?:cheguei|consegui)\s+(?:a\s+)?(?:contestar|abrir)'
    - pattern: '\bcomo\s+(?:funciona|que\s+e)\s+(?:o\s+)?med\b'  # explicativo, não acionável
allowed-tools:
  - consulta_transacao_pix
  - abre_chamado_med
  - consulta_status_med
  - notifica_destinatario
  - escala_humano
domain: pagamentos.pix
owner: squad-pix-disputas
governance:
  escalation_threshold_brl: 50000
  audit_log: true
version: 3.0.0
---

# Skill: Disputa de Pix via MED
...workflow padrão...
```

### 7.3 `cartao-segunda-via` — captura múltipla

```markdown
---
name: cartao-segunda-via
description: Solicita segunda via de cartão (físico ou virtual).
priority: 75
triggers:
  exact:
    - "segunda via cartao"
    - "segunda via do cartao"
    - "novo cartao"
  keywords:
    - "perdi cartao"
    - "cartao perdido"
    - "cartao roubado"
    - "cartao bloqueado"
  regex:
    - pattern: '\bsegunda\s+via\s+(?:do\s+)?(?P<tipo>cartao|cartao\s+virtual|cartao\s+fisico)\b'
      priority: 90
      captures: ['tipo']
    - pattern: '\b(?:perdi|roubaram|extraviou)\s+(?:meu\s+|o\s+)?cartao\b'
      priority: 85
  exclude:
    - pattern: '\bnao\s+(?:perdi|recebi)\b'
allowed-tools:
  - identifica_cartao_cliente
  - bloqueia_cartao
  - solicita_segunda_via
  - confirma_endereco_entrega
domain: cartoes
owner: squad-cards-self-service
version: 1.4.0
---
```

### 7.4 Dataset de eval da skill (`test_triggers.yaml`)

```yaml
# skills/pix-disputa-med/test_triggers.yaml
skill: pix-disputa-med

positive_examples:
  - input: "quero contestar um pix"
    expected_method: regex
    expected_entities: {}

  - input: "abrir MED"
    expected_method: exact
    expected_entities: {}

  - input: "caí num golpe e mandei pix de R$ 1.500 sem querer"
    expected_method: regex
    expected_entities:
      valor: "r$ 1.500"

  - input: "pix indevido na minha conta"
    expected_method: keyword

negative_examples:
  - input: "quero enviar um pix"
    expected_skill: pix-envio

  - input: "agendar pix para amanhã"
    expected_skill: pix-agendamento

  - input: "como funciona o MED?"  # explicativo, exclude
    expected_skill: null
    reason: "exclude pattern fires"

  - input: "preciso falar com o médico"  # falso positivo de "med"
    expected_skill: null
```

---

## 8. Engenharia de regex para NLU

### 8.1 Princípios de design

**Camadas, não monólitos.** Resista à tentação de escrever um regex gigante que cobre todos os casos. Quebre em vários, com prioridades. Mais fácil de testar, debugar, evoluir.

**Captura de entidades é parte do roteamento, não trabalho do LLM.** Se você sabe que "pix de R$ 500" tem um valor, capture com named group e jogue no `entities`. O LLM começa o trabalho com a entidade já em mãos — economiza tokens e turnos.

**Word boundaries em português.** `\b` em regex Unicode geralmente funciona, mas cuidado com acentos quando você **não** normaliza. Como a normalização do template tira acentos, `\b` opera sobre ASCII limpo — comportamento previsível.

**Use non-capturing groups `(?:...)` sempre que não precisar do match.** Performance e clareza.

### 8.2 Padrões idiomáticos de PT-BR

| Padrão de fala | Regex idiomático |
|---|---|
| "um/uma/o/a opcional" | `(?:um\s+|uma\s+|o\s+|a\s+)?` |
| "que/de opcional" | `(?:que\s+|de\s+)?` |
| Verbos com flexão básica | `(?:cancelar?|cancelei|cancelou)` |
| Valor monetário | `r\$?\s*[\d.,]+` |
| CPF (após normalização) | `\b\d{3}\.?\d{3}\.?\d{3}\.?\d{2}\b` |
| Datas curtas | `\b\d{1,2}/\d{1,2}(?:/\d{2,4})?\b` |
| "pra/para" | `(?:pra|para)` (a normalização já expande, mas seguro) |
| Negação | `\bn[ãa]o\s+(?:vou|quero|consegui|consigo)` |

### 8.3 Entity extraction inline

Named groups (`(?P<nome>...)`) que viram entradas em `state.entities` são uma das partes mais valiosas do pattern. Padrões úteis:

```python
# Valor monetário
r'(?P<valor>r\$?\s*\d{1,3}(?:[.,]\d{3})*(?:[.,]\d{2})?)'

# CPF (após normalizer remover pontos/hífens, fica só dígitos)
r'(?P<cpf>\d{11})'

# E2E ID de Pix (32 chars hex)
r'(?P<e2e>[a-f0-9]{32})'

# Número de protocolo
r'protocolo\s+(?P<protocolo>\d{6,10})'

# Tipo de cartão
r'cartao\s+(?P<tipo>fisico|virtual|adicional)'
```

> **Boa prática.** Toda entidade extraída tem um schema declarado em `entities_schema.py`. Compliance pode revisar quais entidades são extraídas de cada skill. PII (CPF, telefone) é flagado e tratado com mascaramento no log.

### 8.4 Priority e disambiguation

O sistema de priority é a peça mais sensível. Heurísticas para definir:

| Categoria de pattern | Priority sugerida |
|---|---|
| Exact match canônico | 100 (automático: +1000 no router) |
| Regex com verbo de ação específico ("contestar pix") | 90–95 |
| Regex com entity extraction completa | 85–92 |
| Regex de relato indireto ("caí em golpe") | 80–88 |
| Keyword multi-palavra | 70–80 (base) |
| Keyword única | 50–70 |
| Sigla isolada ("med", "ted") | 75–85 (com cuidado de exclude) |

**Regra de ouro:** se duas skills competem por inputs parecidos, **declare exclude em uma delas** ao invés de só ajustar priority. Excludes são mais explícitos e auditáveis.

### 8.5 Performance — fazer e não fazer

**Fazer:**
- Compilar tudo no boot (`re2.compile` é caro, matching é barato).
- Aho-Corasick para >50 keywords.
- Normalizar input uma vez por turno, passar string normalizada adiante.
- Cache de resultado de roteamento por input (LRU pequeno, ~1000 entradas) — turnos idênticos têm match cached.

**Não fazer:**
- `re.compile` dentro de `route()`. Sempre eager.
- Regex com `.*` no início. Use anchors (`\b`, `^`) ou prefixo literal.
- Backreferences (`\1`) — re2 não suporta e raramente é necessário em NLU.
- Lookbehind variável — não funciona em re2.

---

## 9. Tool gating, governança e segurança

Idêntico ao pattern híbrido. As três camadas continuam:

1. `allowed-tools` no SKILL.md (contrato declarativo).
2. Filtragem em `_apply_skill` do middleware (enforcement runtime).
3. Validators determinísticos em `validators/` da skill (pré-condições antes de side-effect).

**Diferença específica deste pattern:** como a ativação é determinística e auditável bit-a-bit, o **audit log fica mais rico**:

```python
# Log de cada roteamento (estruturado, formato consumível por SIEM)
{
    "event": "skill_activated",
    "session_id": "...",
    "user_id": "...",
    "skill": "pix-disputa-med",
    "skill_version": "3.0.0",
    "routing_method": "regex",
    "matched_pattern": "\\bcontestar\\s+(?:um\\s+)?pix\\b",
    "priority": 95,
    "coverage": 0.42,
    "entities": {"valor": "r$ 1.500"},
    "input_hash": "sha256:abc123...",   # input mascarado, hash para reprodução
    "router_version": "1.2.0",
    "engine": "re2",
    "latency_ms": 0.4
}
```

Esse log é o que dá ao compliance a propriedade que regex-first tem e LLM-first não: **prova matemática de que mesmo input → mesma skill**.

### 9.1 ReDoS — Regex Denial of Service

Embora `re2` elimine catastrophic backtracking, ainda existem ataques de payload-baseados:

- **Inputs gigantes.** Defina `MAX_INPUT_LENGTH = 2048` no normalizer. Acima disso, truncar.
- **Inputs unicode-bombed.** Após NFD + remoção de combining marks, comprimento volta ao normal, mas valida antes.
- **Excludes pesados.** Se um exclude é caro e roda para muitas skills, ele vira hot path. Mantenha excludes simples e anchorados.

### 9.2 Versionamento e rollback

Cada skill tem `version` em SemVer. Mudanças em triggers são **change** (alteram comportamento de roteamento). Hash de cada `RegexRouter` é calculado no boot:

```python
def router_signature(router) -> str:
    """Hash determinístico do estado do router. Mesmo signature => mesmo
    comportamento de roteamento. Útil para A/B e rollback."""
    h = hashlib.sha256()
    for skill in sorted(router.registry.all(), key=lambda s: s.name):
        h.update(skill.name.encode())
        h.update(skill.version.encode())
        h.update(json.dumps(skill.triggers, sort_keys=True).encode())
    return h.hexdigest()[:16]
```

Esse hash entra no log de cada turno. Quando alguém pergunta "por que esse cliente foi roteado pra X em 12 de novembro?", você reconstrói o estado do router daquela data e reproduz.

---

## 10. Testes, evals e observabilidade

### 10.1 Quatro níveis de teste

| Nível | O que testa | Onde |
|---|---|---|
| **Unit do normalizer** | `normalize()` é determinística e idempotente | `tests/test_normalizer.py` |
| **Unit por skill** | `test_triggers.yaml` (positives + negatives) | `tests/test_skill_triggers.py` |
| **Cross-skill** | Skill correta vence em casos de competição | `tests/test_disambiguation.py` |
| **Fuzz / ReDoS** | Nenhum pattern explode com input adversarial | `tests/test_no_catastrophic_regex.py` |

### 10.2 Teste de positivos/negativos por skill

```python
# tests/test_skill_triggers.py
import yaml, pytest, glob
from agent.skills.regex_router import RegexRouter
from agent.skills.registry import SkillRegistry


@pytest.fixture(scope="module")
def router():
    reg = SkillRegistry()
    reg.load_dir(Path("skills"))
    return RegexRouter(reg)


def collect_test_cases():
    cases = []
    for ttf in glob.glob("skills/**/test_triggers.yaml", recursive=True):
        data = yaml.safe_load(open(ttf))
        skill_name = data["skill"]
        for ex in data.get("positive_examples", []):
            cases.append((skill_name, "positive", ex))
        for ex in data.get("negative_examples", []):
            cases.append((skill_name, "negative", ex))
    return cases


@pytest.mark.parametrize("skill_name,kind,example", collect_test_cases())
def test_routing(router, skill_name, kind, example):
    result = router.route(example["input"])

    if kind == "positive":
        assert result is not None, f"input {example['input']!r} não casou nenhuma skill"
        assert result.skill_name == skill_name
        if "expected_method" in example:
            assert result.method == example["expected_method"]
        if "expected_entities" in example:
            for k, v in example["expected_entities"].items():
                assert result.entities.get(k) == v

    else:  # negative
        expected = example.get("expected_skill")
        if expected is None:
            assert result is None or result.skill_name != skill_name
        else:
            assert result is not None and result.skill_name == expected
```

### 10.3 Fuzz contra ReDoS

```python
# tests/test_no_catastrophic_regex.py
import time
from hypothesis import given, strategies as st


@given(st.text(min_size=1, max_size=2048))
def test_router_under_50ms(router, payload):
    start = time.perf_counter()
    router.route(payload)
    elapsed = (time.perf_counter() - start) * 1000
    assert elapsed < 50, (
        f"Router took {elapsed:.1f}ms on payload of len={len(payload)}. "
        f"Suspect catastrophic backtracking or O(n²) pattern."
    )
```

### 10.4 Métricas em produção

```
skill_router_latency_ms{p=99}              # SLO: <5ms
skill_router_matches_total{skill, method}  # contagem por método
skill_router_no_match_total                # turnos sem match (input fora do catálogo)
skill_router_excluded_total{skill}         # quantas vezes a skill foi descartada por exclude
skill_router_signature                     # hash do router atual (gauge único)
skill_router_entity_extracted_total{skill, entity}
```

A métrica `no_match_total` é a mais importante para evolução: ela aponta os gaps de cobertura. Logs dos inputs que não casaram viram backlog de novos triggers.

---

## 11. Boas práticas e anti-patterns

### 11.1 Boas práticas

✅ **Triggers como contrato.** Trate `triggers:` no SKILL.md com o mesmo cuidado de schema de API. Mudanças passam por PR review do owner.

✅ **`description` para humanos, `triggers` para máquina.** Não tente "agradar o LLM" na description, ela não roteia mais.

✅ **Excludes explícitos sobre ajustar priority.** "Esta skill **não** atende esse caso" é mais legível e auditável que "essa outra skill tem priority maior".

✅ **Captura de entidades sempre que possível.** Se você consegue extrair valor, número de cartão, data, faça. Reduz turnos.

✅ **Versionamento de triggers.** Subir version a cada mudança de `triggers`. Histórico Git + métrica `router_signature` dão rastreabilidade total.

✅ **Logs com hash de input, não input em claro.** Privacy by design.

### 11.2 Anti-patterns

❌ **Regex gigante com 50 alternações.** Quebre em vários. Maintainability > "elegância".

❌ **Catastrophic backtracking patterns.** `(.*)*`, `(a+)+b`, etc. Use re2 e o CI bloqueia.

❌ **Triggers genéricos demais.** Keyword `"pix"` sozinha casa toda skill de Pix. Use frases (`"contestar pix"`, `"enviar pix"`).

❌ **Confiar em capitalization/acentos.** Sempre normalize antes.

❌ **Skill sem `test_triggers.yaml`.** CI deve bloquear.

❌ **Esquecer `exclude`.** Toda skill tem casos próximos de outras. Documentar excludes evita 80% dos bugs de roteamento.

❌ **Mudar triggers sem mudar version.** Quebra reprodutibilidade.

❌ **Ativar LLM fallback "por garantia".** Se você está usando regex-first, decida: ou domínio é fechado (sem fallback), ou é aberto (vai pro híbrido). Fallback "morno" gera bugs intermitentes.

---

## 12. Roadmap de evolução do template

**Fase 1 — Fundação**
- Loader com bloco `triggers`, RegexRouter com 4 estágios, middleware, CI lint.

**Fase 2 — Cobertura e qualidade**
- `scripts/coverage_report.py`: porcentagem de inputs reais (sample de logs) cobertos por triggers.
- Dataset de fuzz por skill, gerado por usuários reais (anonimizado).
- Sugestões automatizadas de novos triggers a partir de `no_match` logs.

**Fase 3 — Performance**
- Migrar Aho-Corasick para `hyperscan` se P99 > 5ms.
- LRU cache de resultados de roteamento por hash de input normalizado.
- Pré-warming do router em workers (carregar registry no fork).

**Fase 4 — Governança avançada**
- Pipeline de aprovação: mudança em `triggers` requer aprovação do owner + revisão de impacto via diff de `router_signature`.
- Sandboxing por tenant: cada cliente da plataforma tem seu próprio RegexRouter, isolado.
- Skill A/B: dois `version`s coexistem, traffic-split controlado por flag.

**Fase 5 — Híbrido reverso**
- Quando domínio começar a abrir, considerar **regex-first + embeddings como segundo estágio** (ainda sem LLM). Cobre casos onde o usuário usa sinônimos não previstos, mas sem cair em latência de LLM.
- Migração de skills "regex-first" para "híbrido" é por skill: a estrutura permite.

---

## 13. Stack de mercado e quando subir de nível

### 13.1 Engines de regex/string matching

| Tecnologia | Quando faz sentido | Latência típica |
|---|---|---|
| **`re` (stdlib)** | Sandbox local, protótipos | ~1–10µs por pattern, mas P99 imprevisível |
| **`regex` (PyPI)** | Quando precisa de features avançadas (Unicode classes, recursão) | Igual a `re`, com mais features |
| **`re2` (google-re2)** ⭐ | **Padrão recomendado.** Produção, banking, qualquer hot path | ~1–5µs, P99 previsível |
| **`hyperscan` (Intel)** | >1000 patterns simultâneos, throughput >10k req/s | <100µs para milhares de patterns em paralelo |
| **`pyahocorasick`** | Multi-keyword scan (estágio 3 do template) | O(n+m), ~100µs para 10k keywords |
| **FlashText (Sujit Pal)** | Keyword extraction simples, sem regex | Mais rápido que regex para keyword puro |

### 13.2 Frameworks de roteamento e NLU

Vale conhecer mesmo que não use diretamente — vários implementam variações do pattern:

- **Rasa NLU** — pioneiro em intent classification + entity extraction com regex/lookup. `regex_features` em `pipeline` é exatamente o que este template faz. Boa fonte de inspiração para entity extractors.
- **Snips NLU** — focava em on-device, com gazetteer (Aho-Corasick) + CRF. Discontinuado, mas a arquitetura é referência.
- **Microsoft LUIS / Azure CLU** — combinam regex features e ML. O conceito de "list entity" e "regex entity" são análogos a `triggers.keywords` e `triggers.regex`.
- **Botpress NLU** — usa Duckling para entities + regex.
- **Duckling (Facebook)** — extração de datas, valores, durações, números. Vale **integrar** ao template como entity extractor padrão para reduzir regex próprio.

### 13.3 Quando este pattern para de servir

Sinais de que regex-first está saturando:

- `no_match_total` cresce semana a semana, sem novos triggers cobrindo.
- Time gasta >20% do sprint mantendo triggers.
- Inputs em produção começam a usar linguagem muito variada (causa: novo segmento de usuário, mudança de canal).
- Skills passam de ~100 e o overhead de manutenção fica insustentável.

**Caminho de evolução:** migrar para **híbrido com embedding como segundo estágio** (regex-first + embedding fallback antes do LLM). Isso preserva a velocidade do regex para o que é previsível, e dá flexibilidade onde o vocabulário diverge — sem entrar de cabeça em LLM-routing.

Em última instância, alguns subdomínios podem migrar inteiros para o pattern híbrido (com LLM fallback) enquanto outros (Pix operacional, cobrança L1) permanecem em regex-first. **O template suporta os dois lado a lado**: skills declaram seu próprio modo no frontmatter (campo `routing_mode: regex | hybrid | llm`), e o middleware roteia conforme.

---

## Apêndice A — Cheat sheet do desenvolvedor

```bash
# Criar nova skill com scaffold de triggers
python scripts/new_skill.py --name boleto-pagar --domain pagamentos.boleto --mode regex

# Validar triggers de todas as skills (CI)
python scripts/lint_skills.py

# Benchmark de latência do router
python scripts/benchmark_router.py --iterations 100000

# Coverage report: qual % de inputs reais é coberto
python scripts/coverage_report.py --logs ./samples/inputs.jsonl

# Hash do router atual (deploy gating)
python -c "from agent.skills import build_router; print(build_router().signature())"
```

## Apêndice B — Template do `SKILL.md` (regex-first)

```markdown
---
name: <kebab-case>
description: <descrição humana, sem se preocupar com routing>
priority: 50
triggers:
  exact:
    - "<frase canônica 1>"
    - "<frase canônica 2>"
  keywords:
    - "<keyword 1>"
    - "<keyword 2>"
    - "<keyword 3>"
  regex:
    - pattern: '<regex re2-safe>'
      priority: 80
      captures: []
  exclude:
    - pattern: '<regex que desqualifica>'
allowed-tools:
  - <tool_1>
  - <tool_2>
domain: <area.subarea>
owner: <squad>
governance:
  audit_log: true
version: 0.1.0
---

# Skill: <Nome>

## Quando usar
- <gatilho humano 1>
- NÃO use quando: <exclusão>

## Workflow
1. <passo 1>
2. <passo 2>

## Anti-patterns
- ❌ <evitar>

## Critérios de sucesso
- <condição>

## Escalação
- <quando escalar>
```

## Apêndice C — `test_triggers.yaml` template

```yaml
skill: <skill-name>

positive_examples:
  - input: "<frase do usuário>"
    expected_method: exact   # exact | keyword | regex
    expected_entities:
      <nome>: "<valor>"

  - input: "<outra frase>"
    expected_method: regex

negative_examples:
  - input: "<frase que NÃO deve casar>"
    expected_skill: null      # ou nome de outra skill

  - input: "<frase ambígua mas excluída>"
    expected_skill: null
    reason: "<por quê>"
```

---

## Referências

- Cox, R. — *Regular Expression Matching Can Be Simple And Fast* — https://swtch.com/~rsc/regexp/regexp1.html (fundamento teórico do re2)
- Google — `google-re2` — https://github.com/google/re2
- Intel — Hyperscan — https://www.hyperscan.io/
- Aho, A. V. & Corasick, M. J. (1975) — *Efficient string matching: an aid to bibliographic search* (algoritmo Aho-Corasick)
- pyahocorasick — https://github.com/WojciechMula/pyahocorasick
- Rasa — *Regex features in NLU* — https://rasa.com/docs/rasa/nlu-training-data/
- Facebook — Duckling — https://github.com/facebook/duckling
- Anthropic — Agent Skills overview — https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- LangChain — Skills pattern — https://docs.langchain.com/oss/python/langchain/multi-agent/skills
- OWASP — Regular Expression Denial of Service (ReDoS) — https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS
