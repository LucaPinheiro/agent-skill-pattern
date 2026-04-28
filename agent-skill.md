# Agent Skills Pattern — Guia de Implementação para Template Agentic em LangGraph

> **Escopo deste documento.** Esta referência descreve o padrão **Agent Skills** e detalha como incorporá-lo a um **template** de construção de agentes em LangGraph. Não estamos descrevendo a plataforma agentic builder como um todo — apenas o template que será o ponto de partida para qualquer novo agente construído sobre ela.
>
> **Audiência.** Engenheiros e arquitetos de IA que vão consumir o template, customizá-lo e estender com novas skills. Pressupõe familiaridade com LangGraph (StateGraph, nodes, edges, middleware) e com o conceito básico de tool calling.

---

## Sumário

1. [Por que Skills (e não só prompts ou só tools)](#1-por-que-skills-e-não-só-prompts-ou-só-tools)
2. [Conceitos centrais](#2-conceitos-centrais)
   - [2.4 Decisão arquitetural — tool `load_skill` vs filesystem-native (ADR)](#24-decisão-arquitetural--tool-load_skill-vs-filesystem-native-adr)
3. [Anatomia de uma Skill](#3-anatomia-de-uma-skill)
4. [O padrão AKU — Atomic Knowledge Unit](#4-o-padrão-aku--atomic-knowledge-unit)
5. [Estrutura de diretórios do template](#5-estrutura-de-diretórios-do-template)
6. [Implementação em LangGraph](#6-implementação-em-langgraph)
7. [Exemplos completos de skills](#7-exemplos-completos-de-skills)
8. [Tool gating, governança e segurança](#8-tool-gating-governança-e-segurança)
9. [Subagentes e isolamento de contexto](#9-subagentes-e-isolamento-de-contexto)
10. [Testes, evals e observabilidade](#10-testes-evals-e-observabilidade)
11. [Boas práticas e anti-patterns](#11-boas-práticas-e-anti-patterns)
12. [Roadmap de evolução do template](#12-roadmap-de-evolução-do-template)

---

## 1. Por que Skills (e não só prompts ou só tools)

### O problema que Skills resolvem

Quando você constrói um agente para um domínio rico (ex.: jornadas bancárias cobrindo Pix, TED, cartões, seguros, consórcio), três caminhos clássicos falham em escala:

| Abordagem | Falha em escala |
|---|---|
| **System prompt monolítico** com todo o conhecimento procedural | Estoura janela de contexto, sofre *context rot* (degradação da atenção em contextos longos), torna prompt frágil e impossível de versionar por subdomínio |
| **Múltiplas tools** (uma por procedimento) | Explode a lista de tools, infla o KV-cache, degrada raciocínio acima de ~70% de utilização do contexto, e mistura "como fazer" com "fazer" |
| **Múltiplos agentes** (um por jornada) | Custo de orquestração alto, perde reuso, dificulta governança transversal, retrabalho operacional para quem mantém os fluxos |

### A proposta de Skills

Skills são **unidades modulares de conhecimento procedural** baseadas em sistema de arquivos. Cada skill empacota:

- **Quando** ela deve ser ativada (intent, gatilhos)
- **O que** ela ensina o agente a fazer (workflow procedural)
- **Quais tools** ela tem permissão de usar (tool gating)
- **Quais regras de negócio e governança** se aplicam (políticas, escalonamentos)

A skill **não executa nada por si só**. Ela é texto + metadados que **moldam o contexto** e **restringem capacidades** do LLM antes que ele decida o próximo passo.

### Skills vs. Tools — diferença arquitetural

| Aspecto | **Tools** | **Skills** |
|---|---|---|
| Papel | "Mãos" — executam ações | "Cérebro/treinamento" — ensinam comportamento |
| Conteúdo | Função executável + schema | Markdown + frontmatter |
| Carregamento | Schema sempre presente no contexto | Apenas metadata; corpo carregado on-demand |
| Versionamento | Código | Texto (ideal para revisão por produto/jurídico/compliance) |
| Quem edita | Engenharia | Engenharia + Produto + Compliance |

A regra prática: **se é determinístico e tem efeito colateral, é tool. Se é procedural e influencia decisão, é skill.**

---

## 2. Conceitos centrais

### 2.1 Progressive Disclosure (ou Progressive Discovery)

Princípio arquitetural central: o agente **descobre o que existe primeiro, e só carrega detalhes quando vai usar**. Três níveis:

```
┌─────────────────────────────────────────────────────────────────┐
│ NÍVEL 1 — METADATA (sempre carregado, ~80 tokens por skill)     │
│   • name + description de cada skill no system prompt           │
│   • Permite o LLM saber que a skill existe e quando aplicar     │
└─────────────────────────────────────────────────────────────────┘
                              ↓ (LLM decide ativar)
┌─────────────────────────────────────────────────────────────────┐
│ NÍVEL 2 — INSTRUÇÕES (carregado on-demand)                      │
│   • Corpo do SKILL.md injetado no contexto                      │
│   • Workflow, regras, anti-patterns, tool bindings              │
└─────────────────────────────────────────────────────────────────┘
                              ↓ (skill referencia recursos)
┌─────────────────────────────────────────────────────────────────┐
│ NÍVEL 3 — RECURSOS (carregado seletivamente)                    │
│   • Arquivos auxiliares: REFERENCE.md, FORMS.md, schemas        │
│   • Scripts de validação, templates                             │
│   • Lidos via tool de filesystem só quando o passo exigir       │
└─────────────────────────────────────────────────────────────────┘
```

**Quem decide a transição entre níveis é o LLM**, com base no input do usuário e no estado atual da conversa. Não é um classificador externo, não é uma rule engine — é o próprio modelo, guiado pelas descrições.

### 2.2 Skill Catalog

A lista de metadatas (Nível 1) injetada no system prompt forma o **catálogo de skills**. O catálogo é o "índice" que o LLM consulta para decidir o que ativar. Skills bem descritas → ativações precisas. Skills mal descritas → confusão de roteamento.

### 2.3 Skill Activation

Acionar uma skill é equivalente a **carregar seu corpo no contexto e potencialmente reescrever a lista de tools disponíveis**. Há duas estratégias para isso, abordadas em detalhe na seção 6:

- **Via tool calling**: o LLM chama uma tool `load_skill(name)` que retorna o corpo da skill.
- **Via middleware**: um middleware intercepta a chamada e, baseado em sinais (estado, último input), injeta a skill antes do LLM decidir.

A primeira é a mais robusta e auditável; é o caminho recomendado para o template. A próxima seção defende essa escolha em detalhe.

### 2.4 Decisão arquitetural — tool `load_skill` vs filesystem-native (ADR)

> **Contexto.** Existem hoje (2026) duas abordagens dominantes para implementar skill activation. Esta seção registra a decisão do template e os tradeoffs, no formato de Architecture Decision Record (ADR). Engenheiros que forem revisitar esta escolha no futuro devem ler aqui antes.

**As duas abordagens em jogo:**

| | **Tool `load_skill`** (escolhida) | **Filesystem-native** (Claude Code-style) |
|---|---|---|
| Mecanismo de ativação | LLM chama `tool_call` explícito → handler controlado retorna corpo da skill via `Command.update` | LLM usa `bash`/`read_file` para `cat skills/x/SKILL.md` |
| Onde mora a lógica | Servidor (sob seu controle) | Sandbox/filesystem do agente |
| Quem precisa de filesystem | Ninguém — é tudo RPC | O agente, sempre |
| Como o gating é aplicado | Mesma operação atômica que ativa a skill | Middleware separado observando leituras de arquivo |
| Audit trail nativo | Sim — `tool_call` é evento de primeira classe | Não — leitura de arquivo é semanticamente ambígua |

**Decisão.** O template usa **tool `load_skill` como mecanismo central de ativação**, com middleware fino apenas para injetar o catálogo (Nível 1) no system prompt. Filesystem-native fica reservado a um caso muito específico (Nível 3, recursos pesados), e mesmo aí, escondido atrás de uma tool RPC `read_skill_resource(skill_name, path)`, nunca expondo `bash` ao agente.

**Razões.**

1. **Auditabilidade de domínio.** Cada ativação vira um evento `load_skill` rastreável no LangSmith. Compliance consegue responder "que skill foi ativada para o cliente X às 14h32?" com uma query simples. Em filesystem-native, "ativação" é só um `bash: cat ...` — tecnicamente registrável, mas semanticamente ambíguo (o agente pode ter lido a skill sem segui-la). Em ambiente regulado, isso pesa.

2. **Atomicidade do gating.** A tool faz três coisas no mesmo `Command.update`: injeta o corpo da skill, reescreve `allowed_tool_names`, grava em `skill_history`. Em filesystem-native, o middleware precisa **inferir** que houve ativação a partir de uma leitura de arquivo e aplicar o gating no turno seguinte — janela de inconsistência por construção. A tool elimina essa janela.

3. **Superfície de ataque.** Em Claude Code faz sentido dar filesystem ao agente — o trabalho dele é mexer em arquivos. Em jornada bancária, dar `bash`/`read_file` é superfície de ataque desnecessária e abre porta para prompt injection direcionado a leitura de arquivos sensíveis. A tool `load_skill` é um RPC: você valida o nome, aplica políticas, retorna apenas conteúdo aprovado.

4. **Portabilidade entre runtimes.** A interface `load_skill(name) -> Command` funciona em LangGraph, em `deepagents`, no OpenAI Agents SDK e em qualquer harness que suporte tool calling. Filesystem-native amarra você a um runtime que tenha sandbox/bash configurados.

5. **Backend trocável.** Hoje filesystem, amanhã Postgres ou Langfuse Prompt Management — sem o agente perceber. A tool isola o backend de armazenamento da interface de ativação. Filesystem-native acopla os dois.

**Custos aceitos.**

- Você precisa **escrever e manter** o `SkillsMiddleware` e o handler da tool. Em filesystem-native isso "vem de graça" se o runtime for Claude Code-like.
- A tool `load_skill` precisa estar sempre disponível ao LLM (mesmo com gating ativo), o que ocupa um slot de tool. Trivial.
- Para skills com muitos recursos auxiliares, o segundo nível de RPC (`read_skill_resource`) adiciona uma tool a mais. Pequeno custo cognitivo para o LLM.

**Quando reconsiderar.**

Reabra esta decisão se uma destas condições mudar:

- O agente vira fundamentalmente um **agente de código** (Claude Code-like), com sandbox/filesystem como ferramenta de trabalho central. Aí filesystem-native é natural.
- Catálogo passa de **~100 skills**. Aí o custo do catálogo no system prompt obriga a um pré-roteador semântico (embedding similarity) — vale revisitar a arquitetura de descoberta como um todo. Veja o paper *Tool Attention* nas referências.
- As skills passam a depender de **recursos extensos** (>10 arquivos auxiliares por skill, lidos em padrões imprevisíveis). O filesystem-native fica mais ergonômico, mesmo com os custos de auditoria.

**Implicações no design.**

- A tool `load_skill` é a **espinha dorsal** do template (§6.4).
- O middleware é fino: injeta catálogo + filtra tools. Não tenta inferir nada (§6.5).
- Recursos de Nível 3 são acessados via tool RPC dedicada, nunca via filesystem cru.
- Backend de skills é abstraído atrás de uma interface (`SkillRegistry`) — filesystem hoje, outro storage amanhã, sem mexer no agente.

---

## 3. Anatomia de uma Skill

Uma skill é um **diretório** contendo, no mínimo, um arquivo `SKILL.md`. Pode conter mais arquivos (referências, schemas, scripts).

### 3.1 Estrutura mínima

```
skills/
└── pix-disputa-med/
    └── SKILL.md
```

### 3.2 Estrutura expandida (recomendada para domínios complexos)

```
skills/
└── pix-disputa-med/
    ├── SKILL.md                    # Ponto de entrada (Nível 2)
    ├── REFERENCE.md                # Referência técnica (Nível 3)
    ├── examples/
    │   ├── caso_fraude.md
    │   └── caso_erro_operacional.md
    ├── schemas/
    │   └── payload_med.json
    └── validators/
        └── validate_end_to_end_id.py
```

### 3.3 O `SKILL.md`

Começa **obrigatoriamente** com YAML frontmatter, seguido de markdown.

```yaml
---
name: pix-disputa-med
description: >
  Conduz fluxo de contestação de Pix via MED (Mecanismo Especial de Devolução).
  Use quando o usuário relatar transferência indevida, fraude, golpe, ou pedir
  para "contestar Pix", "abrir MED", "devolver Pix recebido errado".
  NÃO use para Pix agendado não executado (use pix-agendamento) ou para
  Pix de pessoa jurídica acima de R$ 50.000 (escalar para canal humano).
allowed-tools:
  - consulta_transacao_pix
  - abre_chamado_med
  - notifica_destinatario
  - consulta_status_med
domain: pagamentos.pix
owner: squad-pix-disputas
governance:
  requires_human_approval: false
  escalation_threshold_brl: 50000
  audit_log: true
sla_minutes: 30
version: 2.1.0
---

# Skill: Disputa de Pix via MED

## Quando usar
... (instruções procedurais em markdown)
```

### 3.4 Campos do frontmatter

Os dois campos **obrigatórios** pelo padrão Anthropic original são `name` e `description`. O template estende isso com campos próprios, descritos abaixo.

| Campo | Obrigatório | Descrição |
|---|---|---|
| `name` | ✅ | Identificador único, kebab-case. É a chave usada para `load_skill(name)`. |
| `description` | ✅ | **Crítico.** Determina ativação. Escrita em terceira pessoa, com gatilhos explícitos ("Use quando…", "NÃO use para…"). Veja §11.1. |
| `allowed-tools` | Recomendado | Lista de nomes de tools que ficam disponíveis quando esta skill está ativa. Sem esta lista, o agente herda todas as tools do contexto. |
| `domain` | Recomendado | Domínio hierárquico (`pagamentos.pix`, `cartoes.credito.fatura`). Permite filtros de catálogo. |
| `owner` | Recomendado | Squad ou time responsável. Essencial para governança. |
| `governance` | Opcional | Bloco com regras: aprovação humana, limites, audit. |
| `sla_minutes` | Opcional | SLA da jornada. Pode alimentar timeouts no grafo. |
| `version` | Recomendado | SemVer. Permite skills A/B e rollback. |
| `requires_skills` | Opcional | Lista de skills pré-requisito (ex.: `kyc-validacao`). |
| `next_skills` | Opcional | Sucessoras lógicas (continuação do fluxo). |

### 3.5 Corpo do markdown — seções recomendadas

Estrutura sugerida que o template já vem com snippet/scaffold:

```markdown
# Skill: <Nome legível>

## Quando usar
- Gatilho 1
- Gatilho 2
- NÃO use quando: ...

## Pré-condições
- O usuário deve estar autenticado (tool `verify_session`)
- ...

## Workflow
1. Passo 1 — chamar `consulta_transacao_pix(end_to_end_id)`.
2. Passo 2 — se status = SUCCEEDED, prosseguir. Caso contrário, escalar.
3. ...

## Anti-patterns (o que NÃO fazer)
- Não confirme MED antes de validar end-to-end ID.
- Não exponha dados do destinatário ao reclamante.

## Tool bindings
- `consulta_transacao_pix(end_to_end_id: str)` — usar SEMPRE primeiro.
- `abre_chamado_med(...)` — só após validação completa.

## Critérios de sucesso
- Chamado MED aberto com ID retornado.
- Confirmação enviada ao usuário.

## Escalação
- Valor > R$ 50.000 → transferir para canal humano via `escala_humano(reason)`.
- Cliente PJ → mesma regra.

## Recursos adicionais
- Detalhes técnicos do payload MED: ver [REFERENCE.md](REFERENCE.md).
- Casos de exemplo: ver [examples/](examples/).
```

---

## 4. O padrão AKU — Atomic Knowledge Unit

Para ambientes corporativos (especialmente regulados), uma skill bem feita é uma **Atomic Knowledge Unit** com sete elementos. Use isso como checklist de qualidade.

| # | Elemento | Onde no SKILL.md |
|---|---|---|
| 1 | **Intent** — gatilhos exatos de ativação | `description` (frontmatter) + seção "Quando usar" |
| 2 | **Conhecimento procedural** — passo a passo + anti-patterns | Seções "Workflow" e "Anti-patterns" |
| 3 | **Tool bindings** — quais tools, com que parâmetros | `allowed-tools` (frontmatter) + seção "Tool bindings" |
| 4 | **Metadados organizacionais** — owner, domínio, ambientes | `owner`, `domain` (frontmatter) |
| 5 | **Restrições de governança** — limites, aprovações, blast radius | `governance` (frontmatter) + seção "Escalação" |
| 6 | **Caminhos de continuação** — sucesso, falha, escalonamento | `next_skills`, `requires_skills` + seção "Critérios de sucesso" |
| 7 | **Validadores** — scripts determinísticos | Diretório `validators/` referenciado no SKILL.md |

Uma skill que cumpre os sete elementos é **revisável por compliance, testável isoladamente, versionável e auditável** — exatamente o que um agente de produto regulado precisa.

---

## 5. Estrutura de diretórios do template

A organização abaixo é o **layout padrão do template**. Adapte para seu repositório.

```
my-agent/                          # nome do agente gerado a partir do template
├── pyproject.toml
├── README.md
├── .env.example
│
├── src/
│   └── agent/
│       ├── __init__.py
│       ├── graph.py                # construção do StateGraph principal
│       ├── state.py                # TypedDict do estado
│       ├── nodes.py                # nodes (model, tool executor, etc.)
│       ├── prompts/
│       │   └── system_base.md      # system prompt genérico do domínio
│       ├── tools/
│       │   ├── __init__.py
│       │   ├── registry.py         # registro de tools
│       │   └── pix.py              # implementações por domínio
│       ├── skills/
│       │   ├── __init__.py
│       │   ├── registry.py         # SkillRegistry — carrega/indexa
│       │   ├── loader.py           # parser de SKILL.md (frontmatter + body)
│       │   ├── middleware.py       # SkillsMiddleware
│       │   └── catalog.py          # geração do catálogo (Nível 1)
│       └── config.py               # config geral (modelo, paths, etc.)
│
├── skills/                         # 👈 onde vivem as SKILLS do agente
│   ├── pix-envio/
│   │   └── SKILL.md
│   ├── pix-disputa-med/
│   │   ├── SKILL.md
│   │   ├── REFERENCE.md
│   │   └── examples/
│   ├── ted-agendamento/
│   │   └── SKILL.md
│   ├── cartao-segunda-via/
│   │   └── SKILL.md
│   └── _shared/                    # skills transversais (auth, kyc, escala)
│       ├── kyc-validacao/
│       └── escala-humano/
│
├── tests/
│   ├── test_skill_activation.py
│   ├── test_tool_gating.py
│   └── evals/
│       └── pix-disputa-med.yaml    # cenários de eval por skill
│
└── scripts/
    ├── new_skill.py                # CLI: scaffold de nova skill
    └── lint_skills.py              # valida frontmatter de todas as skills
```

### 5.1 Convenções do template

- Skills vivem em `skills/` na **raiz** do projeto, não dentro de `src/`. Isso reforça que skills são **conteúdo**, não código.
- `skills/_shared/` é descoberto antes das skills específicas. Skills compartilhadas servem como dependências (`requires_skills`).
- O template inclui um `scripts/new_skill.py` que gera o esqueleto de uma nova skill com o frontmatter pré-preenchido. Isso garante consistência.
- `scripts/lint_skills.py` valida no CI: frontmatter válido, descrição em terceira pessoa, tools listadas existem, sem skills com `name` duplicado.

---

## 6. Implementação em LangGraph

Esta seção mostra o **núcleo da implementação** do template. Os snippets são intencionalmente didáticos; em produção você adicionaria logging, métricas, tratamento de erro, etc.

### 6.1 Estado

```python
# src/agent/state.py
from typing import Annotated, Optional
from typing_extensions import TypedDict
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages


class AgentState(TypedDict):
    # Conversa
    messages: Annotated[list[AnyMessage], add_messages]

    # Skill ativa no turno atual (se houver)
    active_skill: Optional[str]

    # Histórico de skills ativadas — para auditoria e métricas
    skill_history: list[str]

    # Tools liberadas pelo gating (ids); None = todas
    allowed_tool_names: Optional[list[str]]

    # Contexto de governança propagado entre nodes
    governance: dict
```

### 6.2 Loader e Registry

```python
# src/agent/skills/loader.py
from dataclasses import dataclass, field
from pathlib import Path
import yaml


@dataclass
class Skill:
    name: str
    description: str
    body: str
    path: Path
    allowed_tools: list[str] = field(default_factory=list)
    domain: str | None = None
    owner: str | None = None
    governance: dict = field(default_factory=dict)
    version: str = "0.1.0"
    requires_skills: list[str] = field(default_factory=list)
    next_skills: list[str] = field(default_factory=list)

    @property
    def metadata_line(self) -> str:
        """Linha que entra no catálogo (Nível 1)."""
        return f"- `{self.name}`: {self.description.strip()}"


def parse_skill_file(skill_md_path: Path) -> Skill:
    """Lê um SKILL.md, separa frontmatter (YAML) e body (markdown)."""
    raw = skill_md_path.read_text(encoding="utf-8")
    if not raw.startswith("---"):
        raise ValueError(f"{skill_md_path}: faltou frontmatter YAML")

    _, fm_raw, body = raw.split("---", 2)
    fm = yaml.safe_load(fm_raw)

    if "name" not in fm or "description" not in fm:
        raise ValueError(f"{skill_md_path}: 'name' e 'description' são obrigatórios")

    return Skill(
        name=fm["name"],
        description=fm["description"],
        body=body.strip(),
        path=skill_md_path.parent,
        allowed_tools=fm.get("allowed-tools", []) or [],
        domain=fm.get("domain"),
        owner=fm.get("owner"),
        governance=fm.get("governance", {}) or {},
        version=fm.get("version", "0.1.0"),
        requires_skills=fm.get("requires_skills", []) or [],
        next_skills=fm.get("next_skills", []) or [],
    )
```

```python
# src/agent/skills/registry.py
from pathlib import Path
from .loader import Skill, parse_skill_file


class SkillRegistry:
    """Descobre, indexa e serve skills.

    Estratégia de precedência: skills carregadas por último vencem em caso
    de colisão de nome (permite override por subdomínio/projeto).
    """

    def __init__(self):
        self._skills: dict[str, Skill] = {}

    def load_dir(self, root: Path) -> None:
        """Varre recursivamente um diretório por SKILL.md e os indexa."""
        for skill_md in sorted(root.rglob("SKILL.md")):
            skill = parse_skill_file(skill_md)
            self._skills[skill.name] = skill

    def get(self, name: str) -> Skill:
        if name not in self._skills:
            raise KeyError(f"Skill '{name}' não encontrada")
        return self._skills[name]

    def all(self) -> list[Skill]:
        return list(self._skills.values())

    def catalog(self, domain_prefix: str | None = None) -> str:
        """Gera o catálogo (Nível 1) — vai para o system prompt."""
        skills = self.all()
        if domain_prefix:
            skills = [s for s in skills if s.domain and s.domain.startswith(domain_prefix)]
        if not skills:
            return "(nenhuma skill disponível)"
        return "\n".join(s.metadata_line for s in sorted(skills, key=lambda s: s.name))
```

### 6.3 System prompt com catálogo injetado

```markdown
<!-- src/agent/prompts/system_base.md -->
Você é o assistente de produtos financeiros do <ORGANIZAÇÃO>.

Princípios:
- Trate o cliente com clareza, sem jargão.
- Confirme dados sensíveis antes de executar ações.
- Em caso de dúvida ou risco, escale para humano.

Você tem acesso a um catálogo de **skills** — fluxos especializados que você
pode ativar conforme a intenção do usuário. Para ativar uma skill, chame
`load_skill(name)`. Após ativá-la, siga RIGOROSAMENTE o workflow descrito
nela e use APENAS as tools que ela autoriza.

Skills disponíveis:
{skill_catalog}

Regras de ativação:
- Identifique a intenção do usuário e ative a skill mais aderente.
- Se nenhuma skill se aplica, responda apenas com base nestes princípios e
  ofereça ajuda geral, sem inventar procedimentos.
- Se duas skills concorrem, prefira a mais específica e justifique.
- Você pode ativar uma nova skill durante a conversa se a intenção mudar.
```

### 6.4 Tool `load_skill` (mecanismo de ativação)

> Esta é a peça central do template. A justificativa para escolher tool ao invés de filesystem-native está documentada em §2.4 (ADR).

```python
# src/agent/skills/middleware.py — parte 1: tool de ativação
from langchain_core.tools import tool
from langgraph.prebuilt import InjectedState
from langgraph.types import Command
from typing import Annotated


def make_load_skill_tool(registry):
    """Factory que cria a tool `load_skill` ligada ao registry."""

    @tool
    def load_skill(
        name: str,
        state: Annotated[dict, InjectedState],
    ) -> Command:
        """Ativa uma skill pelo nome.

        Após chamar esta tool, o conteúdo da skill é injetado no contexto
        e suas tools permitidas passam a estar disponíveis.

        Args:
            name: nome exato da skill (ver catálogo no system prompt).
        """
        try:
            skill = registry.get(name)
        except KeyError:
            return f"Skill '{name}' não existe. Veja o catálogo no system prompt."

        # Atualiza o estado: skill ativa + tool gating + histórico
        return Command(
            update={
                "active_skill": skill.name,
                "skill_history": state.get("skill_history", []) + [skill.name],
                "allowed_tool_names": skill.allowed_tools or None,
                "messages": [
                    {
                        "role": "tool",
                        "content": (
                            f"Skill '{skill.name}' ativada (v{skill.version}).\n\n"
                            f"--- INSTRUÇÕES DA SKILL ---\n{skill.body}"
                        ),
                    }
                ],
            }
        )

    return load_skill
```

### 6.5 Middleware de tool gating

A parte sensível: depois que uma skill ativa, **filtramos as tools** apresentadas ao LLM no próximo turno. Isso é o que dá garantia de governança.

```python
# src/agent/skills/middleware.py — parte 2: gating

def gate_tools(all_tools, allowed_names):
    """Retorna apenas as tools cujo nome está em allowed_names.

    A tool `load_skill` está SEMPRE disponível (permite trocar de skill).
    """
    if allowed_names is None:
        return all_tools  # nenhuma skill ativa = sem restrição
    keep = set(allowed_names) | {"load_skill"}
    return [t for t in all_tools if t.name in keep]
```

Em LangGraph com `create_agent` ou `langchain.agents`, isso vira um middleware:

```python
# src/agent/skills/middleware.py — parte 3: integração via middleware
from langchain.agents.middleware import AgentMiddleware


class SkillsMiddleware(AgentMiddleware):
    """Aplica progressive disclosure + tool gating a cada turno.

    Antes do model:
      - Substitui {skill_catalog} no system prompt
      - Filtra a lista de tools com base em state['allowed_tool_names']
    """

    def __init__(self, registry, all_tools):
        self.registry = registry
        self.all_tools = all_tools

    def before_model(self, state, request):
        # 1) Catálogo no system prompt
        catalog = self.registry.catalog()
        request.system_prompt = request.system_prompt.replace(
            "{skill_catalog}", catalog
        )

        # 2) Tool gating
        request.tools = gate_tools(
            self.all_tools, state.get("allowed_tool_names")
        )
        return request
```

### 6.6 Construção do grafo

```python
# src/agent/graph.py
from pathlib import Path
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import InMemorySaver
from langchain_anthropic import ChatAnthropic

from .state import AgentState
from .skills.registry import SkillRegistry
from .skills.middleware import make_load_skill_tool, SkillsMiddleware
from .tools.registry import build_domain_tools


def build_graph(skills_root: Path, model_name: str = "claude-opus-4-7"):
    # 1) Carrega skills
    registry = SkillRegistry()
    registry.load_dir(skills_root / "_shared")  # primeiro: compartilhadas
    registry.load_dir(skills_root)              # depois: específicas (override)

    # 2) Monta tools de domínio + a tool de ativação
    domain_tools = build_domain_tools()
    load_skill = make_load_skill_tool(registry)
    all_tools = domain_tools + [load_skill]

    # 3) System prompt
    system_prompt = (Path(__file__).parent / "prompts/system_base.md").read_text()

    # 4) Modelo + middleware
    llm = ChatAnthropic(model=model_name)
    middleware = SkillsMiddleware(registry, all_tools)

    # 5) Nodes
    def call_model(state: AgentState):
        request = type("Req", (), {
            "system_prompt": system_prompt,
            "tools": all_tools,
            "messages": state["messages"],
        })()
        request = middleware.before_model(state, request)
        bound = llm.bind_tools(request.tools)
        response = bound.invoke(
            [{"role": "system", "content": request.system_prompt}] + request.messages
        )
        return {"messages": [response]}

    tool_node = ToolNode(all_tools)

    def route(state: AgentState):
        last = state["messages"][-1]
        return "tools" if getattr(last, "tool_calls", None) else END

    # 6) Grafo
    g = StateGraph(AgentState)
    g.add_node("model", call_model)
    g.add_node("tools", tool_node)
    g.add_edge(START, "model")
    g.add_conditional_edges("model", route, {"tools": "tools", END: END})
    g.add_edge("tools", "model")

    return g.compile(checkpointer=InMemorySaver())
```

### 6.7 Fluxo de execução, do começo ao fim

```
[user] "preciso contestar um Pix que mandei errado"
   │
   ▼
[model node]
   • SkillsMiddleware.before_model:
       - injeta {skill_catalog} no system prompt
       - tools liberadas: domínio inteiro (sem skill ativa ainda)
   • LLM lê o catálogo, identifica intent "disputa Pix"
   • LLM emite tool_call: load_skill(name="pix-disputa-med")
   │
   ▼
[tool node — load_skill]
   • registry.get("pix-disputa-med")
   • Command.update:
       active_skill = "pix-disputa-med"
       allowed_tool_names = [consulta_transacao_pix, abre_chamado_med, ...]
       messages += [tool_msg com corpo da SKILL.md]
   │
   ▼
[model node]
   • before_model agora aplica gating:
       tools = [load_skill, consulta_transacao_pix, abre_chamado_med, ...]
   • LLM tem acesso ao workflow completo + tools restritas ao subdomínio
   • Segue passo a passo da skill: pede end_to_end_id, valida, abre MED
   │
   ▼
[tool node — consulta_transacao_pix → abre_chamado_med → ...]
   │
   ▼
[response final ao usuário]
```

---

## 7. Exemplos completos de skills

### 7.1 Skill simples — `pix-envio`

```markdown
---
name: pix-envio
description: >
  Realiza envio de Pix para uma chave (CPF/CNPJ/email/telefone/aleatória).
  Use quando o usuário pedir para "enviar Pix", "pagar via Pix", "transferir
  por Pix". NÃO use para Pix agendado (use pix-agendamento) ou contestação
  (use pix-disputa-med).
allowed-tools:
  - consulta_chave_pix
  - valida_saldo
  - executa_pix
  - notifica_usuario
domain: pagamentos.pix
owner: squad-pix-emissao
governance:
  requires_human_approval: false
  daily_limit_brl: 20000
sla_minutes: 1
version: 1.4.0
next_skills:
  - pix-comprovante
---

# Skill: Envio de Pix

## Quando usar
Usuário deseja enviar valor a um terceiro via Pix imediato.

## Pré-condições
- Sessão autenticada (`verify_session`).
- Saldo suficiente.
- Chave Pix do destinatário em formato válido.

## Workflow
1. Solicite a chave Pix do destinatário (se não fornecida).
2. Chame `consulta_chave_pix(chave)` e CONFIRME o nome do titular ao usuário
   antes de prosseguir. Nunca pule esta confirmação.
3. Solicite o valor.
4. Chame `valida_saldo(valor)`. Se insuficiente, informe e finalize.
5. Chame `executa_pix(chave, valor, descricao_opcional)`.
6. Chame `notifica_usuario(canal="push", template="pix_enviado")`.
7. Ofereça gerar comprovante (skill `pix-comprovante`).

## Anti-patterns
- ❌ Executar Pix sem confirmar nome do titular.
- ❌ Aceitar valor acima de R$ 20.000 sem alertar limite.
- ❌ Inferir chave Pix; sempre solicitar explicitamente.

## Critérios de sucesso
- `executa_pix` retorna `status: SUCCEEDED` com `end_to_end_id`.
- Notificação enviada.

## Escalação
- Erro de saldo após validação positiva → escalar para `escala-humano`.
```

### 7.2 Skill complexa — `pix-disputa-med`

```markdown
---
name: pix-disputa-med
description: >
  Conduz contestação de Pix recebido indevidamente, abrindo MED (Mecanismo
  Especial de Devolução do Bacen). Use para casos de fraude, golpe, erro
  operacional ou recebimento equivocado. NÃO use para Pix agendado não
  executado, nem para Pix de PJ acima de R$ 50.000 (escalar humano).
allowed-tools:
  - consulta_transacao_pix
  - abre_chamado_med
  - consulta_status_med
  - notifica_destinatario
  - escala_humano
domain: pagamentos.pix
owner: squad-pix-disputas
governance:
  requires_human_approval: false
  escalation_threshold_brl: 50000
  audit_log: true
sla_minutes: 30
version: 2.1.0
requires_skills:
  - kyc-validacao
---

# Skill: Disputa de Pix via MED

## Quando usar
Usuário relata transferência indevida, fraude (golpe do Pix, falso
suporte, etc.), ou erro operacional em Pix realizado nos últimos 80 dias.

## Pré-condições
- Sessão autenticada.
- KYC validado (skill `kyc-validacao`).
- Transação dentro da janela de 80 dias.

## Workflow
1. Pergunte o `end_to_end_id` (E2E) ou peça para o usuário localizar a
   transação no extrato.
2. Chame `consulta_transacao_pix(e2e_id)`.
   - Se não encontrada → informar e finalizar.
   - Se valor > R$ 50.000 OU pagador é PJ → chamar `escala_humano(reason="MED acima do threshold")` e encerrar.
3. Classifique o motivo com o usuário (escolher 1):
   - `FRAUDE`
   - `GOLPE`
   - `ERRO_OPERACIONAL`
4. Chame `abre_chamado_med(e2e_id, motivo, descricao)`.
5. Informe ao usuário o número do chamado MED e o SLA (até 11 dias úteis
   para resposta do PSP recebedor).
6. Chame `notifica_destinatario(e2e_id)` (notificação obrigatória pelo Bacen).

## Anti-patterns
- ❌ Abrir MED sem validar o E2E ID.
- ❌ Expor dados pessoais do destinatário ao reclamante.
- ❌ Garantir devolução; o MED não garante, apenas formaliza pedido.

## Critérios de sucesso
- `abre_chamado_med` retorna ID de protocolo.
- Mensagem ao usuário com número do protocolo e SLA.
- Notificação ao destinatário disparada.

## Escalação
- Valor > R$ 50.000 → `escala_humano`.
- Cliente PJ → `escala_humano`.
- Falha técnica em `abre_chamado_med` → tentar 2x; persistindo, `escala_humano`.

## Recursos adicionais
- Detalhes técnicos do payload MED, incluindo enum de motivos: ver
  [REFERENCE.md](REFERENCE.md).
- Casos típicos: [examples/caso_fraude.md](examples/caso_fraude.md).
```

### 7.3 Skill compartilhada — `_shared/escala-humano`

```markdown
---
name: escala-humano
description: >
  Transfere a conversa para atendimento humano via canal apropriado.
  Use SEMPRE que outra skill instruir escalação, ou quando o usuário pedir
  explicitamente "falar com atendente", "humano", "gerente".
allowed-tools:
  - cria_protocolo_atendimento
  - transfere_canal_humano
domain: atendimento
owner: squad-omnichannel
governance:
  audit_log: true
version: 1.0.0
---

# Skill: Escala para humano

## Workflow
1. Crie protocolo: `cria_protocolo_atendimento(motivo, contexto)`.
2. Informe ao usuário o número do protocolo e tempo médio de espera.
3. Chame `transfere_canal_humano(protocolo)`.

## Anti-patterns
- ❌ Não diga "vou transferir" sem efetivamente chamar a tool.
```

---

## 8. Tool gating, governança e segurança

### 8.1 Por que gating é não-negociável em domínios regulados

Sem gating, mesmo com a skill ativa, o LLM **vê todas as tools do agente** e pode chamar qualquer uma. Para banco, isso é inaceitável — uma skill de "consulta de saldo" não deveria nunca conseguir invocar `executa_pix`.

O `SkillsMiddleware.before_model` aplicado na seção 6.5 garante isso a cada turno.

### 8.2 Patrão de defesa em camadas

```
1. allowed-tools no SKILL.md          ← contrato declarativo
2. SkillsMiddleware.before_model      ← enforcement em runtime
3. Validators determinísticos         ← bloqueio antes de side-effects
4. Audit log + LangSmith trace        ← detecção a posteriori
```

A camada 3 é importante: tools sensíveis (ex.: `executa_pix`, `abre_chamado_med`) devem ter pré-condições verificadas por código antes de executar, não dependendo só do LLM ter "lido" a skill.

### 8.3 Aprovação humana — Human-in-the-Loop

Skills com `governance.requires_human_approval: true` devem usar `interrupt()` do LangGraph para pausar antes do passo crítico. Padrão:

```python
from langgraph.types import interrupt

@tool
def executa_pix_com_aprovacao(chave: str, valor: float, state):
    if valor > state["governance"]["limite_aprovacao"]:
        decision = interrupt({
            "type": "approval_required",
            "operation": "executa_pix",
            "chave": chave,
            "valor": valor,
        })
        if not decision.get("approved"):
            return "Operação não aprovada pelo usuário."
    return _executa_pix_real(chave, valor)
```

### 8.4 Audit trail

Toda ativação de skill é registrada em `state["skill_history"]`. Combine com LangSmith ou seu sistema de tracing interno para:

- Reproduzir um turno específico
- Métricas: taxa de ativação por skill, taxa de sucesso, fallback para humano
- Auditoria regulatória: que skill foi usada para qual cliente em qual momento

---

## 9. Subagentes e isolamento de contexto

Para fluxos longos (ex.: jornada de consórcio com múltiplas etapas e dezenas de tools), o pattern recomendado é **delegar para um subagente** que tem seu próprio escopo de skills.

```python
# src/agent/subagents.py
from .skills.registry import SkillRegistry

def build_consorcio_subagent():
    sub_registry = SkillRegistry()
    sub_registry.load_dir(Path("skills/_shared"))
    sub_registry.load_dir(Path("skills/consorcio"))
    # ... mesma fábrica de grafo, mas com o sub_registry
    return build_graph_with_registry(sub_registry)
```

**Regra**: o agente principal **não enxerga** as skills do subagente. Isso é isolamento de contexto: o pai delega uma tarefa de alto nível, o filho lida com seu próprio universo, retorna um resultado consolidado.

Isso converge com o padrão **Deep Agents** da LangChain, onde subagents são criados com `CompiledSubAgent` e cada um tem sua própria configuração de skills.

---

## 10. Testes, evals e observabilidade

### 10.1 Três níveis de teste

| Nível | O que testa | Ferramenta |
|---|---|---|
| **Unit (skill)** | Frontmatter válido, descrição em terceira pessoa, tools listadas existem | `scripts/lint_skills.py` no CI |
| **Activation (eval)** | Dada uma frase do usuário, a skill correta é ativada | LangSmith evals + dataset por skill |
| **End-to-end (cenário)** | Jornada completa cumpre critérios de sucesso definidos no SKILL.md | LangSmith + asserts em `tests/` |

### 10.2 Dataset de ativação por skill

```yaml
# tests/evals/pix-disputa-med.yaml
skill: pix-disputa-med
positive_examples:
  - "caí num golpe e mandei Pix sem querer"
  - "preciso contestar uma transferência"
  - "como abrir MED?"
  - "recebi um Pix por engano, como devolvo?"
negative_examples:
  - "quero enviar um Pix"               # → pix-envio
  - "agendei um Pix e não saiu"         # → pix-agendamento
  - "qual meu limite de Pix?"           # → pix-limites
```

O CI roda esses datasets contra o agente compilado e mede precision/recall de ativação. Skill com recall < 80% ou precision < 90% **falha o build**.

### 10.3 Métricas em produção

- `skill_activation_total{name=...}` — contagem de ativações
- `skill_completion_total{name=..., outcome=...}` — sucesso/falha/escalação
- `skill_latency_ms{name=...}` — latência fim-a-fim
- `tool_gate_violation_total{skill=..., tool=...}` — alertas (não deveria acontecer; se acontece, há bug ou prompt injection)

---

## 11. Boas práticas e anti-patterns

### 11.1 A descrição é o gatilho

A `description` no frontmatter é o **único** texto que o LLM vê no Nível 1. Ela carrega o peso de toda a ativação. Regras práticas:

✅ **Faça**:
- Terceira pessoa: "Conduz contestação de Pix..."
- Padrão "Use quando…" + "NÃO use para…"
- Inclua palavras-chave que o usuário realmente diz: "golpe", "contestar", "MED"
- Cite skills concorrentes para desambiguar

❌ **Evite**:
- Primeira pessoa: "Eu consigo processar Pix..."
- Vago: "Lida com Pix"
- Marketing: "A melhor solução para seus problemas com Pix"

> Pequena mudança na descrição pode levar a taxa de ativação de ~20% para >80%. Trate descrição como código de produção.

### 11.2 Tamanho da skill

- `SKILL.md` < 500 linhas. Acima disso, a atenção do modelo cai.
- Conteúdo extenso (esquemas técnicos, listas de erros, payloads completos) → mover para `REFERENCE.md` referenciado, e o LLM lê via tool de filesystem só se precisar.

### 11.3 Micro-skills > mega-skills

Não crie `gestao-bancaria` que faz tudo. Crie:
- `consulta-saldo`
- `consulta-extrato`
- `consulta-limite-pix`
- `pix-envio`
- ...

Vantagens: gating mais preciso, eval por skill, evolução independente, ownership claro.

### 11.4 Skills como contrato com produto/compliance

Em ambiente regulado, jurídico e produto **deveriam** revisar SKILL.md como revisam runbooks. O markdown é o ponto de encontro entre engenharia e business — aproveite isso. O template já vem com PR template específico para mudança de skill, exigindo aprovação do `owner`.

### 11.5 Anti-patterns recorrentes

| Anti-pattern | Por que dói |
|---|---|
| Carregar **todas as skills** no system prompt | Some o ganho de contexto; vira monólito |
| `allowed-tools: ["*"]` em skills sensíveis | Anula o gating; furo de governança |
| Skill que **executa código** diretamente em vez de instruir o LLM | Confunde papéis Skill vs Tool; quebra reuso |
| `description` curta demais ("Trata Pix") | Ativação errada ou nenhuma |
| Skills com nomes em inglês num agente em PT-BR | Reduz acerto de roteamento por LLM |
| **Mega-skill** de 2.000 linhas | Context rot, manutenção impossível |
| Sem versionamento (`version`) | Impossível auditar mudança de comportamento |
| Sem `owner` | Fica órfã; ninguém atualiza quando regra muda |

---

## 12. Roadmap de evolução do template

Sugestão de fases para amadurecer o template ao longo do tempo:

**Fase 1 — Fundação (este documento)**
- Registry, loader, middleware, tool gating, scaffold de skill, lint no CI.

**Fase 2 — Operação**
- Hot-reload de skills (sem restart do agente).
- Versionamento e A/B test de skills (tráfego split por `version`).
- Storage de skills em Postgres ou Langfuse Prompt Management (em vez de só filesystem).

**Fase 3 — Governança avançada**
- Pipeline de aprovação de skill (PR review → staging eval → produção).
- Assinatura criptográfica de SKILL.md (cadeia de custódia).
- Painel de governança: skills por owner, cobertura de testes, taxa de ativação.

**Fase 4 — Inteligência sobre skills**
- Auto-sugestão de novas skills a partir de logs de fallback (turnos onde nenhuma skill ativou).
- Skill analytics: dependências reais entre skills (a partir de `next_skills` observado).
- Geração assistida de skills via "skill-creator skill" (meta-skill).

**Fase 5 — Multi-tenant**
- Catálogo de skills por tenant (cliente da plataforma agentic builder).
- Marketplace interno de skills reutilizáveis (`_shared/` virando primeiro nível).

---

## Apêndice A — Cheat sheet do desenvolvedor de skills

```bash
# Criar nova skill
python scripts/new_skill.py --name pix-cancelamento --domain pagamentos.pix

# Validar todas as skills
python scripts/lint_skills.py

# Rodar evals de ativação de uma skill
pytest tests/evals/test_activation.py -k pix-disputa

# Rodar agente local com hot-reload de skills
python -m agent.dev_server --reload skills/
```

## Apêndice B — Template do `SKILL.md`

```markdown
---
name: <kebab-case-unique>
description: >
  <1-3 frases. Terceira pessoa. "Use quando…" + "NÃO use para…".>
allowed-tools:
  - <tool_name_1>
  - <tool_name_2>
domain: <area.subarea>
owner: <squad-name>
governance:
  requires_human_approval: false
  audit_log: true
sla_minutes: <int>
version: 0.1.0
---

# Skill: <Nome legível>

## Quando usar
- <gatilho 1>
- NÃO use quando: <exclusão>

## Pré-condições
- <pré-req>

## Workflow
1. <passo 1>
2. <passo 2>

## Anti-patterns
- ❌ <o que não fazer>

## Critérios de sucesso
- <condição mensurável>

## Escalação
- <quando escalar para humano>
```

---

## Referências

- Anthropic — *Equipping agents for the real world with Agent Skills* — https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- Anthropic — Agent Skills (docs) — https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- LangChain — Skills pattern — https://docs.langchain.com/oss/python/langchain/multi-agent/skills
- LangChain — Subagents (Deep Agents) — https://docs.langchain.com/oss/python/deepagents/subagents
- LangChain — `langchain-skills` repo — https://github.com/langchain-ai/langchain-skills
- LangChain — `deepagents` repo — https://github.com/langchain-ai/deepagents
- L. Pessini — *Stop Stuffing Your System Prompt* — https://pessini.medium.com/stop-stuffing-your-system-prompt-build-scalable-agent-skills-in-langgraph-a9856378e8f6
- A B Vijay Kumar — *Building Deep Agents + SKILL.md with LangChain* — https://abvijaykumar.medium.com/building-deep-agents-skill-md-with-langchain-074176c66dec
- P. Whittaker — *Progressive Discovery: A Better Mental Model* — https://dev.to/phil-whittaker/progressive-discovery-a-better-mental-model-for-agent-skills-51bd
- *Tool Attention* (arxiv) — https://arxiv.org/html/2604.21816