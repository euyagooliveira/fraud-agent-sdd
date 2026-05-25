<!--
SYNC IMPACT REPORT
==================
Version change: (template placeholder) → 1.0.0
Constitution status: Initial population — all placeholders resolved

Added principles (8 total, template had 5 slots):
  I.   Determinismo Onde Possível
  II.  Type-Safe Python
  III. Prompts Versionados
  IV.  Observability Nativa
  V.   Guardrails de Segurança
  VI.  Test-First em Lógica de Negócio
  VII. Abstração de Modelo
  VIII.Local-First Development

Added sections:
  - Stack & Restrições de Ambiente
  - Padrões de Qualidade & Gates de CI
  - Governance

Templates reviewed:
  ✅ .specify/templates/plan-template.md
       Constitution Check gate is dynamic ("Gates determined based on constitution
       file") — no changes needed; 8 principles are now the gate source.
  ⚠  .specify/templates/tasks-template.md
       Tests are marked OPTIONAL ("only if explicitly requested"). Constitution
       principle VI mandates TEST-FIRST for all domain scoring/fraud-rule modules.
       When /speckit-tasks runs on this project, test tasks for domain modules
       MUST be generated regardless of explicit request — task generator must
       treat this project's constitution as overriding the template default.
  ✅ .specify/templates/spec-template.md
       Standard structure; no conflicts with any principle. FR sections align
       with determinism and security guardrail requirements.

Deferred TODOs: None
-->

# Agente de Análise de Fraude Conversacional — Constitution

## Core Principles

### I. Determinismo Onde Possível

Funções de cálculo de risco e scoring DEVEM ser puras e determinísticas: dado
o mesmo conjunto de inputs, o output é sempre idêntico e sem efeitos colaterais.
LLM é permitido somente nas camadas conversacional (interpretação de perguntas
em linguagem natural) e de explicação (geração de justificativas textuais).
Nenhuma decisão de fraude pode depender de saída não-determinística de modelo.

**Rationale**: Fraude tem consequências legais e financeiras. Resultados
inconsistentes para a mesma transação geram contestações impossíveis de
defender, dificultam auditorias regulatórias e corroem a confiança dos times
de operações. Determinismo é o contrato que torna o sistema auditável.

**Implications**:
- Funções de scoring recebem tipos primitivos ou dataclasses imutáveis; não
  acessam estado externo (banco, relógio, variáveis de ambiente) diretamente.
- Scores são calculados antes de qualquer chamada LLM; o LLM recebe o score
  já computado e apenas o explica.
- `random`, `datetime.now()`, ou I/O dentro de funções de scoring são vetados;
  devem ser injetados como parâmetros quando necessário.

**Exemplo válido**:
```python
def score_velocity(tx_count: int, window_seconds: int, threshold: int) -> float:
    return min(tx_count / threshold, 1.0)
```

**Contra-exemplo proibido**:
```python
def score_velocity(user_id: str) -> float:
    count = db.query(...)          # efeito colateral
    explanation = llm.explain(...) # não-determinístico
    return float(explanation["score"])
```

---

### II. Type-Safe Python

Todo código Python do projeto DEVE ser tipado com type hints. A ferramenta
`mypy --strict` DEVE passar sem erros em CI. O uso de `Any` é proibido exceto
em fronteiras externas (deserialização de JSON de terceiros, chamadas de SDK),
onde DEVE ser acompanhado de um comentário explicando por que `Any` é
necessário e qual validação de runtime compensa a ausência de tipagem estática.

**Rationale**: Em sistemas de fraude, um bug de tipo silencioso (ex.: score
retornado como `str` em vez de `float`) pode suprimir alertas críticos ou
gerar falsos positivos em escala. Tipagem estática é uma camada de segurança
tão importante quanto testes.

**Implications**:
- `pyproject.toml` inclui `[tool.mypy] strict = true`.
- PRs não passam em CI se `mypy --strict` reportar erros.
- Modelos de dados usam `dataclass` ou `pydantic.BaseModel` tipados; proibido
  usar dicts genéricos para representar entidades de domínio.
- Fronteiras de `Any` DEVEM conter validação Pydantic ou equivalente
  imediatamente após o ponto de entrada.

**Exemplo de fronteira justificada**:
```python
# Any justificado: resposta bruta de boto3 antes de parsing Pydantic
raw: Any = bedrock_client.invoke_model(...)
response = BedrockResponse.model_validate(raw)
```

---

### III. Prompts Versionados

Todo prompt enviado a um LLM DEVE residir em um arquivo separado dentro de
`prompts/`, com sufixo de versão no nome (ex.: `risk_explainer_v1.txt`,
`sql_generator_v2.txt`). Strings de prompt inline no código são proibidas.
Qualquer alteração de conteúdo em um prompt DEVE gerar um novo arquivo com
versão incrementada e um registro correspondente em `prompts/CHANGELOG.md`.
Prompts são tratados como parte do contrato público do sistema, não como
detalhe de implementação.

**Rationale**: Prompts determinam o comportamento observável do sistema para
usuários e reguladores. Sem versionamento, é impossível reproduzir um resultado
histórico ("por que o sistema classificou essa transação assim em março?"),
realizar rollback seguro, ou comparar A/B entre versões. Prompts são artefatos
de engenharia, não textos descartáveis.

**Implications**:
- `prompts/` é diretório versionado no repositório.
- `prompts/CHANGELOG.md` documenta: versão, data, autor, sumário da mudança,
  motivação (ex.: "v2: adicionado contexto de geolocalização — v1 produzia
  falsos negativos para transações internacionais").
- Código referencia prompt por `prompt_id` + `prompt_version` (ex.:
  `loader.get("risk_explainer", version="v1")`), nunca por string literal.
- Testes de regressão de prompt são executados antes de promover nova versão.

**Contra-exemplo proibido**:
```python
response = llm.complete(
    "Você é um analista de fraude. Explique o risco desta transação: ..."
)
```

---

### IV. Observability Nativa

Toda chamada a um LLM DEVE ser registrada em log estruturado (JSON) contendo
obrigatoriamente os campos: `timestamp`, `prompt_id`, `prompt_version`,
`model`, `latency_ms`, `input_tokens`, `output_tokens`, `estimated_cost_usd`,
`request_id`. Logs NUNCA podem conter PII (CPF, nome completo, número de
cartão, e-mail, endereço). O campo `request_id` DEVE ser rastreável do log
até a resposta entregue ao usuário.

**Rationale**: Sem observabilidade, não há como responder a auditorias ("qual
modelo respondeu a essa consulta?"), controlar custos de API em produção, ou
detectar degradação de latência. A exigência de ausência de PII em logs é
requisito legal e de compliance para sistemas financeiros.

**Implications**:
- Um decorator ou context manager centralizado (`@observe_llm_call`) envolve
  toda invocação de LLM; nenhuma chamada ocorre fora desse wrapper.
- `estimated_cost_usd` é calculado via tabela de preços por modelo mantida
  localmente (não consultada em runtime).
- Pipeline de log passa por um sanitizador de PII antes de escrita; qualquer
  campo que contenha padrão de CPF, cartão ou e-mail é rejeitado com erro,
  não silenciosamente omitido.
- Logs de desenvolvimento podem ir para stdout (JSON); produção usa destino
  configurável (ex.: CloudWatch, arquivo local).

---

### V. Guardrails de Segurança

**PII em prompts**: É proibido enviar PII a qualquer prompt LLM. Apenas IDs
internos e features anonimizadas/hasheadas são permitidas (ex.:
`user_hash`, `merchant_category_code`, `tx_amount_bucket`).

**SQL gerado por LLM**: Todo SQL produzido por LLM DEVE passar por validação
obrigatória antes de qualquer execução. A validação implementa: (a) whitelist
de comandos — somente `SELECT` é permitido; (b) bloqueio de DDL/DML (`CREATE`,
`DROP`, `INSERT`, `UPDATE`, `DELETE`, `ALTER`, `TRUNCATE`); (c) limite máximo
de rows retornados (configurável, padrão 1000). A fase de validação é
uma etapa separada e explícita no pipeline — não um best-effort.

**Rationale**: Um prompt LLM que receba CPF ou número de cartão representa
vazamento de dado sensível a um terceiro (provedor de LLM). SQL injection via
geração de linguagem natural é um vetor real; sem validação, um prompt
adversarial pode extrair ou corromper dados.

**Implications**:
- `SQLValidator` é uma classe dedicada com método `validate(sql: str) ->
  ValidationResult`; raise em caso de violação antes de executar.
- Testes de `SQLValidator` cobrem: SELECT válido, cada DDL/DML individualmente,
  SQL com subquery, SQL com comentário de obfuscação.
- A função de anonimização de PII (`anonymize_for_prompt`) é testada com
  exemplos reais de CPF, nome e número de cartão para confirmar que nenhum
  padrão passa.
- Code review verifica se qualquer ponto de construção de prompt contém
  interpolação de campo de origem externa sem passar por `anonymize_for_prompt`.

---

### VI. Test-First em Lógica de Negócio

Toda regra de fraude e toda função de scoring DEVE ter seu teste unitário
escrito e falhando (red) antes que a implementação seja iniciada. LLMs são
sempre substituídos por `MockProvider` em testes unitários. Módulos de
domínio (scoring, validação, regras) DEVEM manter cobertura mínima de 80%.
Código de glue (CLI, handlers de API, adaptadores de I/O) é excluído dessa
meta mínima.

**Rationale**: Regras de fraude codificam conhecimento de negócio crítico. Sem
testes escritos antes, a implementação tende a ser guiada pela conveniência
técnica em vez de pelo comportamento esperado. A cobertura mínima de 80% em
domínio garante que regressões em regras de negócio sejam detectadas antes de
chegar a produção.

**Implications**:
- O ciclo obrigatório é: escrever teste → confirmar falha (red) → implementar
  → confirmar verde → refatorar se necessário.
- PRs de nova regra de fraude ou função de scoring DEVEM conter os testes
  correspondentes no mesmo commit; PRs sem testes para módulos de domínio são
  bloqueados em CI.
- `pytest-cov` roda em CI com `--fail-under=80` para os módulos em
  `src/domain/` e `src/scoring/`.
- `MockProvider` é um `LLMProvider` determinístico cujas respostas são
  fixtures versionadas em `tests/fixtures/`.

---

### VII. Abstração de Modelo

Código de negócio NUNCA importa SDK de provedor de LLM diretamente. Todo
acesso a LLM ocorre através da interface `LLMProvider`, com implementações
concretas intercambiáveis: `BedrockProvider`, `AnthropicProvider`,
`MockProvider`. O provedor ativo é selecionado por variável de ambiente ou
arquivo de configuração; trocar de modelo é uma mudança de uma linha de config,
não uma mudança de código de negócio.

**Rationale**: Provedores de LLM mudam preços, deprecam modelos e introduzem
outages. Acoplamento direto ao SDK do provedor garante que qualquer mudança
exija refatoração em toda a base de código. A abstração protege a lógica de
negócio de volatilidade de infraestrutura e viabiliza testes sem dependência
de cloud.

**Implications**:
- `src/llm/provider.py` define `LLMProvider` como `Protocol` ou `ABC`
  tipado.
- `import boto3` e `import anthropic` ocorrem APENAS dentro de
  `BedrockProvider` e `AnthropicProvider` respectivamente.
- O container de injeção de dependência (ou factory function) recebe a
  string do provedor e retorna a instância concreta; módulos de serviço
  recebem `LLMProvider` por parâmetro, nunca instanciam diretamente.
- Adicionar suporte a novo provedor = criar nova classe que implementa
  `LLMProvider`; zero mudança em código de domínio.

**Configuração de provedor**:
```
LLM_PROVIDER=mock         # desenvolvimento local (padrão)
LLM_PROVIDER=bedrock      # produção AWS
LLM_PROVIDER=anthropic    # alternativa direta
```

---

### VIII. Local-First Development

Qualquer desenvolvedor DEVE conseguir clonar o repositório e executar o
sistema completo localmente em menos de 5 minutos, sem credenciais AWS e
sem conexão com serviços externos. SQLite é o banco de dados padrão para
desenvolvimento. `MockProvider` é o provedor LLM padrão. AWS e outros
serviços de cloud são opt-in, ativados por variável de ambiente.

**Rationale**: Dependência obrigatória de cloud em desenvolvimento cria
fricção de onboarding, impede trabalho offline, expõe custos de API
desnecessários durante iteração local, e torna CI mais lento e frágil.
Um loop de feedback local rápido é pré-requisito para qualidade de
engenharia sustentável.

**Implications**:
- `make dev` (ou `./scripts/setup.sh`) configura o ambiente completo:
  cria virtualenv, instala dependências, inicializa SQLite com schema e
  dados de seed, valida que `LLM_PROVIDER=mock` está ativo.
- O `README.md` contém um "Quick Start" de 3 comandos que resulta em
  sistema funcional com dados de exemplo.
- Nenhuma dependência de runtime importa `boto3`, `botocore` ou qualquer
  SDK de cloud fora das classes `*Provider` isoladas.
- `docker-compose.yml` (se presente) NÃO requer credenciais externas para
  subir.
- CI usa `MockProvider` por padrão; `BedrockProvider` é testado em pipeline
  separado e opcional (`test-integration-aws`).

---

## Stack & Restrições de Ambiente

Esta seção documenta as escolhas técnicas fixas que todos os módulos DEVEM
respeitar. Alterações aqui seguem o processo de emenda da Governance.

| Camada | Escolha | Observação |
|---|---|---|
| Linguagem | Python 3.11+ | Mínimo de versão; features de 3.11 (exception groups, `tomllib`) permitidas |
| LLM (produção) | AWS Bedrock — região `sa-east-1` | Preferência por latência e residência de dados no Brasil |
| LLM (desenvolvimento) | `MockProvider` | Padrão; sem custo, sem latência |
| Banco (desenvolvimento) | SQLite | Arquivo local; schema idêntico ao de produção |
| Banco (produção) | Configurável via `DATABASE_URL` | Deve suportar dialeto SQL compatível com SQLite para queries geradas |
| Prompts | Arquivos em `prompts/` | Versionados; ver Princípio III |
| Type checking | `mypy --strict` | Obrigatório em CI |
| Testes | `pytest` + `pytest-cov` | Cobertura mínima 80% em módulos de domínio |
| Linting | `ruff` | Configurado em `pyproject.toml` |

**Restrições adicionais**:
- O sistema é projetado para análise de transações do mercado brasileiro;
  features monetárias usam `Decimal`, não `float`.
- Respostas ao usuário devem ser produzidas em português por padrão;
  idioma é configurável via parâmetro, não hardcoded.

---

## Padrões de Qualidade & Gates de CI

### Gates obrigatórios em todo PR

Todos os itens abaixo DEVEM passar antes de merge; PRs que falhem em qualquer
gate são bloqueados automaticamente:

1. `mypy --strict` — zero erros de tipo
2. `ruff check` — zero violações de linting
3. `pytest tests/unit/ --fail-under=80` — cobertura mínima em domínio
4. `pytest tests/unit/ -k "sql_validator"` — todos os testes de guardrail SQL passam
5. Revisão humana confirmando ausência de PII em prompts e de imports de SDK
   fora das classes `*Provider`

### Gates de integração (pipeline separado, opcional em dev)

- `pytest tests/integration/` com `LLM_PROVIDER=mock` — sem custo, sem AWS
- `pytest tests/integration/ -m aws` com credenciais reais — executado em
  ambiente de staging sob demanda

### Checklist de code review

Reviewers DEVEM verificar explicitamente:
- [ ] Funções de scoring são puras (sem I/O, sem estado externo)
- [ ] Novos prompts estão em `prompts/` com versão e entrada em CHANGELOG
- [ ] Toda chamada LLM passa pelo wrapper de observabilidade
- [ ] Nenhum campo de PII alcança a construção de prompt
- [ ] SQL gerado passa por `SQLValidator` antes de execução
- [ ] Testes de domínio foram escritos antes da implementação (verificar
  histórico de commits no PR)
- [ ] Nenhum import direto de SDK de provedor fora de `*Provider`

---

## Governance

### Força normativa

Esta constituição tem precedência sobre qualquer outra convenção de código,
preferência pessoal ou decisão de PR individual. Em caso de conflito entre
um princípio aqui descrito e uma prática existente no repositório, o princípio
da constituição prevalece, e a prática existente DEVE ser atualizada.

### Processo de emenda

1. **Proposta**: Abrir PR com a alteração proposta em
   `.specify/memory/constitution.md` e descrição justificando a mudança.
2. **Discussão**: PR permanece aberto por no mínimo 48 horas para comentários
   do time.
3. **Aprovação**: Requer aprovação de pelo menos dois mantenedores do projeto.
4. **Versionamento**: A versão da constituição é incrementada conforme:
   - **MAJOR**: Remoção ou redefinição incompatível de princípio existente.
   - **MINOR**: Adição de novo princípio ou expansão material de um existente.
   - **PATCH**: Clarificações, correções de texto, ajustes não-semânticos.
5. **Propagação**: O autor da emenda é responsável por atualizar todos os
   templates e artefatos afetados no mesmo PR.
6. **Registro**: `LAST_AMENDED_DATE` é atualizado para a data de merge.

### Exceções temporárias

Exceções a princípios (ex.: uso de `Any` além das fronteiras permitidas)
podem ser aprovadas por PR com justificativa explícita, prazo de resolução
definido, e issue aberta de acompanhamento. Exceções sem prazo não são
aprovadas.

### Revisão periódica

A constituição DEVE ser revisada integralmente a cada 6 meses ou após
mudança significativa de stack (novo provedor LLM, novo banco de dados,
nova regulação aplicável). A revisão pode resultar em emenda ou apenas em
confirmação de que os princípios permanecem válidos.

---

**Version**: 1.0.0 | **Ratified**: 2026-05-25 | **Last Amended**: 2026-05-25
