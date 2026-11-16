# Pendências e Próximos Passos — Gerador-Traces-OpenTelemetry-Anomalias (`tracedrift`)

Documento técnico consolidando as correções aplicadas, o que ainda falta evoluir/validar no repositório e o que depende de credenciais, serviços ou ambiente externo fornecidos por você.

---

## Correções Implementadas nesta Rodada

1. **Registro do subcomando `judge` na CLI (`tracedrift/cli.py`):**
   - Movido `if __name__ == "__main__": app()` para o final do arquivo, garantindo que `python -m tracedrift.cli judge` funcione normalmente, com suporte a `--concurrency` e fechamento limpo do `LLMClient`.
2. **Exportação em streaming O(1) e preservação no `--resume` (`tracedrift/exporter.py`):**
   - Adicionado `asyncio.Lock` em `StreamingTraceExporter.write_trace()` evitando escrita concorrente intercalada em `traces.jsonl` e `labels.jsonl`.
   - `finalize()` agora faz streaming linha a linha de `traces.jsonl` para gerar `otlp_traces.json` e `otlp_logs.json` sem acumular o dataset inteiro em RAM (atendendo ao RNF06) e preservando traces anteriores ao retomar com `--resume`.
3. **Orçamento de tokens seguro sob concorrência (`tracedrift/client.py`, `tracedrift/generator.py`):**
   - Protegido `total_tokens_used` / `max_tokens_budget` com `asyncio.Lock` em `LLMClient` e adicionado `close()`.
   - Tratamento limpo de `TokenBudgetExceededError` em `TraceGenerator.run()` cancelando tasks pendentes, consolidando os arquivos já gerados e exibindo aviso na CLI.
   - `TraceGenerator` passa a registrar o `layer_failed` real no `rejected.jsonl` e valida `service_name` contra `flow.services`.
4. **Normalizer e Validator (`tracedrift/normalizer.py`, `tracedrift/validator.py`):**
   - Removido bloco inalcançável no `auto_fix` de spans e adicionado ajuste automático de `events` e `logs` para dentro do intervalo `[start, end]` do span.
   - `TraceValidator` agora valida `allowed_services` (Camada 2), timestamp dos logs dentro do span e limite de `<= 2%` de logs `ERROR` em traces normais (RF07), além de sempre executar a Camada 4 (PII/IPs/secrets) mesmo quando a Camada 3 falha.
5. **CLI `validate`, `report` e `send` (`tracedrift/cli.py`, `tracedrift/judge.py`, `tracedrift/report.py`):**
   - `tracedrift validate` agora detecta e valida `otlp_logs.json` via `validate_otlp_logs_conformance`.
   - `tracedrift report` lê `metadata.model` e `metadata.seed` reais de `traces.jsonl` ao gerar `DATASET_CARD.md`.
   - `tracedrift send --replay-realtime` também ajusta `timeUnixNano` de `otlp_logs.json` (`resourceLogs`).
   - `TraceJudge.evaluate_traces_file` ganhou execução concorrente com semáforo e fallback aprimorado de cenário.

---

## 1. O que falta fazer no projeto (Pendências Técnicas / Evolução)

### 1.1 Streaming real de `otlp_traces.json` em arquivos gigantes e envio em lotes (`tracedrift send`)
- **Situação atual:** `StreamingTraceExporter.finalize()` gera `otlp_traces.json` e `otlp_logs.json` em O(1) de RAM, porém `tracedrift send` ainda carrega o arquivo JSON inteiro via `json.load(f)` antes de enviar.
- **O que falta:** Para datasets muito grandes (ex.: 100.000 traces / RNF06), implementar leitura em chunks/lotes configuráveis no `tracedrift send` (ex.: `--batch-size 100`), além de suporte nativo a envio de `otlp_logs.json` via gRPC (`LogsServiceStub`).

### 1.2 Cancelamento preventivo de chamadas LLM em voo no `--max-tokens-budget`
- **Situação atual:** O lock em `LLMClient` garante consistência no contador e bloqueia novas chamadas assim que `total_tokens_used >= max_tokens_budget`, mas requisições HTTP já enviadas antes de atingir o limite ainda concluem antes da checagem seguinte.
- **O que falta:** Se for necessária precisão estrita de teto de custo, estimar/reservar tokens antes da chamada à API e liberar/reconciliar após o `usage` real.

### 1.3 Exportação de métricas/relatório do `tracedrift judge` em arquivo
- **Situação atual:** `tracedrift judge` exibe a tabela Rich no terminal, mas não salva o relatório (`judge_report.json`) em disco para pipelines automatizados.
- **O que falta:** Adicionar flag `--output` opcional no comando `judge`.

### 1.4 Validação rigorosa de assinatura de todas as 20 anomalias (`RF12`)
- **Situação atual:** `TraceValidator` verifica assinaturas explícitas de `retry_storm` (>= 3 retries) e `cascading_timeout`, emitindo *warnings*.
- **O que falta:** Expandir verificadores estruturais para mais padrões do catálogo (ex.: `circuit_breaker_tripping`, `n_plus_one_query`, `deadlock_abort`) e permitir flag `--strict` para promover warnings em rejeições.

### 1.5 Textos residuais opcionais em `prompts/*.j2` e `cli.py`
- **Situação atual:** Comentários e mensagens internas mencionam `TraceDrift` como nome histórico da ferramenta (`tracedrift` segue preservado como nome do pacote Python e comando CLI).
- **O que falta:** Caso deseje remover toda menção ao nome antigo até das mensagens de terminal/prompts, atualizar `cli.py` e `tracedrift/prompts/*.j2`.

---

## 2. O que depende de você (Credenciais, Infraestrutura e Decisões)

| Item | O que precisa ser definido / fornecido | Impacto direto |
|---|---|---|
| **1. Provedor LLM real (`.env`)** | Configurar `OPENAI_API_KEY`, `OPENAI_BASE_URL` e `OPENAI_MODEL` (OpenAI, Groq ou Ollama local em `http://localhost:11434/v1`). | Necessário para executar `tracedrift generate` e `tracedrift judge` com geração real fora de mocks. |
| **2. Docker ativo para `deploy/docker-compose.yml`** | Subir Docker Desktop / daemon e rodar `docker compose -f deploy/docker-compose.yml up -d`. | Necessário para testar fim a fim `tracedrift send` (HTTP `4318` e gRPC `4317`) no OpenTelemetry Collector, Jaeger (`16686`) e Tempo (`3200`). |
| **3. Benchmark real de escala (`RNF03` / `RNF06`)** | Executar geração em lote real (ex.: `--concurrency 16` e datasets extensos) contra o provedor escolhido. | Valida memória real (`< 1 GB`), latência e comportamento do provedor sob rate limit. |
| **4. Renomear a pasta local no Windows** | Sair/fechar o terminal ou editor que mantém lock em `C:\Projetos\TraceDrift` e executar `Rename-Item -LiteralPath "C:\Projetos\TraceDrift" -NewName "Gerador-Traces-OpenTelemetry-Anomalias"`. | Alinha o nome da pasta local ao repositório renomeado no GitHub. |
| **5. Commit e Push das correções atuais** | Confirmar se deseja fazer `commit` e `push` das correções + `PENDENCIAS.md` para `origin/main`. | Mantém o GitHub atualizado com as correções validadas. |

---

## 3. Referência rápida de verificação

```bash
uv run ruff check .
uv run ruff format --check .
uv run mypy --strict tracedrift
uv run pytest --cov=tracedrift --cov-report=term-missing
```
