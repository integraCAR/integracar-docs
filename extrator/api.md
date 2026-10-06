# API de Extração - `integracar-backend-extrator`

API FastAPI + Postgres do sistema de extração. É dona de quatro coisas:

- a **fila de jobs** (tabela `jobs`), que o worker consome;
- os **documentos extraídos** (`documentos.campos`, JSONB) e o **histórico de
  versões** de cada um (`documento_versoes`);
- o **schema do banco**, versionado com Alembic. Só este repositório roda
  migrações;
- a **API pública** `/v1`, para clientes autorizados fora do sistema de gestão.

Não tem usuários, login nem interface. Quem a consome é o sistema de gestão,
servidor-a-servidor. Visão geral do sistema em [README.md](README.md).

---

## Sumário

1. [Estrutura do repositório](#1-estrutura-do-repositório)
2. [Autenticação](#2-autenticação)
3. [Rotas internas](#3-rotas-internas)
4. [API pública `/v1`](#4-api-pública-v1)
5. [Banco de dados](#5-banco-de-dados)
6. [Configuração](#6-configuração)
7. [Rodar em desenvolvimento](#rodar-em-desenvolvimento)
8. [Scripts de manutenção](#8-scripts-de-manutenção)
9. [Testes e CI](#9-testes-e-ci)
10. [Convenções](#10-convenções)

---

## 1. Estrutura do repositório

```
api/
  main.py                 App FastAPI, /health, monta a sub-app pública em /v1
  security.py             X-Service-Token (interno) e X-Api-Key (público)
  schemas.py              Schemas Pydantic de resposta
  routes/
    documentos.py         Documentos, uploads, PDF, páginas, exportação, performance
    jobs.py               Fila: listagem, estatísticas, histórico, manutenção
    worker.py             Status e controle do worker (via heartbeat)
    extrator_publico.py   API pública /v1 (2 rotas)
db/                       Camada de dados (SQLAlchemy). Fonte da verdade do schema
  config.py, base.py      DATABASE_URL, engine, sessão
  models.py               Tabelas (ver seção 5)
  queue.py                Operações da fila (claim com FOR UPDATE SKIP LOCKED)
  documentos.py           Gravar/ler campos, versões, listagem consolidada
  callbacks.py            Disparo do callback de conclusão (espelhado no worker)
  clientes_api.py         Clientes da API pública e posse de documentos
alembic/                  Migrações do schema (9 versões)
utils/
  exporta.py              Exportação XLSX/CSV consolidada
  log_parser.py           Lê os logs de processamento (tela de performance)
  rate_limit.py           Rate limit por cliente da API pública
scripts/                  Manutenção: criar cliente, apagar dados, reset
worker_control.py         Status do worker pelo arquivo de heartbeat
docker-compose.yml        Ambiente isolado de dev (Postgres + API + worker vizinho)
docker-up.sh              Setup do host (expõe o Ollama, confere versão) + sobe o compose
docs/                     OLLAMA_SETUP.md, DEPLOY_INTEGRACAO.md, pesquisa de modelos de OCR
tests/                    Unitários e de integração com Postgres real
```

Diretórios de runtime, compartilhados com o worker por bind mount:

| Diretório | Conteúdo | Quem grava |
| --- | --- | --- |
| `data/raw/` | PDFs originais, nomeados pelo SHA-256 | API |
| `data/extracted/<hash>/` | Fatias de imagem e OCR bruto (`<hash>_resultado.json`) | worker |
| `data/worker.heartbeat` | Batimento do worker | worker |
| `logs/` | Log de processamento por PDF | worker |

---

## 2. Autenticação

| Área | Header | Conferido contra | Observação |
| --- | --- | --- | --- |
| Rotas internas (`documentos`, `jobs`, `worker`) | `X-Service-Token` | `SERVICE_TOKEN` (env) | Um consumidor só: o sistema de gestão. Comparação em tempo constante (`hmac.compare_digest`). A API **não sobe** sem `SERVICE_TOKEN` |
| `/health` | nenhum | - | Público, usado pelo blackbox_exporter |
| API pública `/v1` | `X-Api-Key` | hash em `clientes_api.chave_hash` | Uma chave por cliente; cada cliente só vê os próprios documentos |
| Callback (saída) | `X-Service-Token` | `CALLBACK_TOKEN` | Enviado pelo worker ao gestão |

Não há CORS: ninguém chama esta API de um navegador.

---

## 3. Rotas internas

Documentação interativa em `http://localhost:8000/docs`. O `cod_usuario` é o
id do usuário **no sistema de gestão**; ele viaja como parâmetro.

### Saúde e performance

| Método | Rota | Descrição |
| --- | --- | --- |
| `GET` | `/health` | Healthcheck, sem token |
| `GET` | `/performance?cod_usuario=` | Resumo de performance do OCR por documento, lido dos logs |
| `GET` | `/documentos/edicoes-manuais` | Quantas correções manuais cada usuário fez (painel do coordenador no gestão) |

### Documentos

| Método | Rota | Descrição |
| --- | --- | --- |
| `GET` | `/documentos?cod_usuario=` | Lista consolidada (capa + requerimento + CCIR + quadro de áreas) |
| `GET` | `/documentos/export.xlsx` | Consolidado em XLSX, multi-aba |
| `GET` | `/documentos/export.csv` | Consolidado em CSV |
| `GET` | `/documentos/{id}` | Campos de um documento |
| `PUT` | `/documentos/{id}` | Salva campos editados e arquiva uma versão |
| `DELETE` | `/documentos/{id}?cod_usuario=` | Remove só o vínculo do usuário. Não apaga documento, campos, PDF nem versões, que podem ser de outros usuários |
| `GET` | `/documentos/{id}/versoes` | Histórico de versões |
| `GET` | `/documentos/{id}/pdf` | PDF original, inline |
| `GET` | `/documentos/{id}/pdf-info` | Metadados do PDF |
| `GET` | `/documentos/{id}/paginas/{n}/imagem` | Página `n` renderizada em JPEG |
| `GET` | `/documentos/{id}/performance` | Performance do OCR daquele documento |
| `POST` | `/documentos/{id}/reprocessar` | Reenfileira (`?cod_usuario=&callback_url=`) |

### Upload

| Método | Rota | Descrição |
| --- | --- | --- |
| `POST` | `/uploads` | Multipart: `files` + `cod_usuario` + `callback_url`. Deduplica por hash; novo é salvo e enfileirado |
| `POST` | `/uploads/iniciar` | Inicia upload em partes, para arquivos acima de `MAX_UPLOAD_MB` (padrão 50) |
| `POST` | `/uploads/{upload_id}/chunk` | Envia uma parte |
| `POST` | `/uploads/{upload_id}/finalizar` | Junta as partes, deduplica e enfileira |

### Fila

| Método | Rota | Descrição |
| --- | --- | --- |
| `POST` | `/jobs` | Enfileira um job |
| `GET` | `/jobs?cod_usuario=` | Lista jobs |
| `GET` | `/jobs/stats` | Contagem por status |
| `GET` | `/jobs/fila-atual` | Estado atual da fila |
| `GET` | `/jobs/historico` | Jobs concluídos (tabela `historico_jobs`) |
| `GET` | `/jobs/fila-snapshots` | Série histórica do tamanho da fila |
| `GET` | `/jobs/{id}` | Detalhe e progresso de um job |
| `GET` | `/jobs/{id}/posicao` | Posição do job na fila |
| `DELETE` | `/jobs/{id}` | Remove o job. O worker checa entre páginas e aborta |
| `POST` | `/jobs/clear-terminados` | Limpa jobs terminados |
| `POST` | `/jobs/reset-orfaos` | Devolve para `pending` jobs `running` sem worker |

Status de um job: `pending`, `running`, `done`, `error`. Durante o
processamento, `progress_stage` indica a etapa (`ocr`, depois `llm`) e
`progress_page`/`progress_total`, a página.

### Worker

| Método | Rota | Descrição |
| --- | --- | --- |
| `GET` | `/worker/status` | Online ou offline, pelo heartbeat |
| `POST` | `/worker/start` · `/worker/stop` | Controle do worker fora do Docker. No Docker, use `docker compose start/stop worker` |

---

## 4. API pública `/v1`

Sub-aplicação FastAPI montada em `/v1`, com `/v1/docs` e `/v1/openapi.json`
**próprios**. Assim um parceiro externo não descobre, nem pelo Swagger, as
rotas administrativas. Exposta para fora pelo `server_name api.integracar.agr.br`
no gateway nginx (`integracar-infra-extrator/web/nginx.conf`).

| Método | Rota | Descrição | Limite |
| --- | --- | --- | --- |
| `POST` | `/v1/documentos` | Envia um PDF (até `MAX_UPLOAD_MB`). Devolve `documento_id`, `job_id`, `status`, `ja_existia` | 30/min por cliente |
| `GET` | `/v1/documentos/{id}` | Status e campos extraídos | 120/min por cliente |

- O rate limit é **por cliente**, não por IP (dois clientes atrás do mesmo NAT
  não dividem o limite). Fica em memória (`slowapi`, `memory://`): reinício
  zera os contadores, e com mais de um processo o limite não é compartilhado.
- A deduplicação por hash vale aqui também: mesmo PDF, mesmo documento, sem
  reprocessar. Só muda o vínculo de posse (`documento_clientes`).
- Novos clientes são criados à mão:
  `python3 -m scripts.criar_cliente_api --nome "Parceiro X" [--callback URL]`.
  A chave aparece **uma vez** na tela; só o hash é gravado. Repasse por canal
  seguro.

---

## 5. Banco de dados

PostgreSQL 16. Schema em `db/models.py`, migrações em `alembic/versions/`.

| Tabela | Conteúdo |
| --- | --- |
| `jobs` | Fila. `pdf_path`, `status`, `usuario_id` ou `cliente_api_id`, `documento_id`, `callback_url`, tempos, progresso, `error_msg` |
| `documentos` | Um por PDF distinto (`hash` único). `campos` (JSONB com as seções extraídas) e colunas indexadas para busca: `numero`, `municipio`, `interessado`, `cpf_cnpj`, `codigo_empreendimento` |
| `documento_usuarios` | Vínculo N:N documento ↔ usuário do gestão. "Excluir" no gestão remove só o vínculo |
| `documento_versoes` | Snapshot de `campos` a cada extração ou edição, com `origem` e `usuario_id`. Sem poda |
| `clientes_api` | Clientes da API pública: `nome`, `chave_hash`, `callback_url`, `ativo`, `ultimo_uso_em` |
| `documento_clientes` | Vínculo N:N documento ↔ cliente da API pública |
| `historico_jobs` | Registro de cada job concluído: origem, status final, tempo de fila, tempo de processamento, páginas |
| `fila_snapshots` | Fotografia periódica do tamanho da fila (gravada pelo serviço `fila-snapshot` do infra) |

### A coluna `campos`

JSON livre, com uma chave por seção: `capa`, `requerimento_digital`, `ccir`,
`quadro_areas`, `quadro_areas_conferencia` (só quando há divergência) e
`_fontes` (página de origem de cada campo). **Quem grava é o worker**, em outro
repositório. Mudança na estrutura de uma seção lá quebra em silêncio a listagem
(`carregar_consolidado`) e as exportações daqui. Já aconteceu: a coluna de
divergência ficaria vazia para sempre, e vazio ali significa "as fontes
batem", então ninguém notaria.

O `jsonb` **não preserva a ordem das chaves**. Onde a ordem importa (planilha,
listagem), ela é fixada no código; ver `CAMPOS_QUADRO_AREAS` em
`db/documentos.py`. Lista dos campos em
[worker-ocr.md](worker-ocr.md#5-campos-extraídos).

### Migrações

```bash
uv run alembic upgrade head            # aplica
uv run alembic revision -m "descricao" # nova migração
```

Mudou o schema? **Espelhe o `db/` no `integracar-ocr-extrator`** e confira:

```bash
cd ../integracar-ocr-extrator && diff -r ../integracar-backend-extrator/db db
```

---

## 6. Configuração

Variáveis (modelo em `.env.example`; o `.env` não é versionado):

| Variável | Obrigatória | Uso |
| --- | --- | --- |
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | Sim (Docker) | Credenciais do Postgres; o compose monta a URL |
| `DATABASE_URL` | Sim (bare-metal) | URL SQLAlchemy, ex. `postgresql+psycopg://...` |
| `SERVICE_TOKEN` | Sim | Token das rotas internas. Gere com `openssl rand -hex 32` |
| `CALLBACK_TOKEN` | Recomendada | Token do callback (usado pelo worker) |
| `CALLBACK_TIMEOUT_S`, `CALLBACK_TENTATIVAS`, `CALLBACK_BACKOFF_S` | Não | Ajuste do disparo do callback |
| `OLLAMA_HOST` | Não | Padrão no Docker: `http://host.docker.internal:11434` |
| `MAX_UPLOAD_MB` | Não | Limite do upload simples. Padrão 50 |

---

## Rodar em desenvolvimento

Em produção, a API sobe pela stack do [infra](infra.md). Para trabalhar só
nela:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh   # se não tiver o uv
uv sync
cp .env.example .env                               # preencha SERVICE_TOKEN e o Postgres

# Postgres local (uma vez)
sudo -u postgres psql -c "CREATE USER integracar WITH PASSWORD 'SUA_SENHA';" \
                      -c "CREATE DATABASE integracar OWNER integracar;"
uv run alembic upgrade head

uv run uvicorn api.main:app --host 127.0.0.1 --port 8000
```

Alternativa com Docker (Postgres + API + worker do repositório vizinho):

```bash
./docker-up.sh
```

O `docker-up.sh` expõe o Ollama do host em `0.0.0.0:11434`, **aborta se o
Ollama não for 0.16.x**, migra dados do Postgres nativo para o container se ele
estiver vazio (pule com `SKIP_MIGRATE=1`) e sobe o compose. O compose procura
o worker em `${WORKER_PATH:-../integracar-ocr}` (nome antigo); defina
`WORKER_PATH=../integracar-ocr-extrator` no `.env`.

Para trocar a senha do banco depois de criado, rode `ALTER USER` no Postgres;
mudar só o `.env` não altera a senha real.

---

## 8. Scripts de manutenção

Todos fazem **dry-run** sem `--sim`.

| Script | O que faz |
| --- | --- |
| `scripts/criar_cliente_api.py --nome N [--callback URL]` | Cria um cliente da API pública e mostra a chave uma única vez |
| `scripts/apagar_dados_usuario.py --usuario ID [--sim]` | Apaga os documentos de um usuário (PDF, campos, versões, jobs), respeitando documentos compartilhados com outros usuários |
| `scripts/reset_dados.py [--sim] [--arquivos]` | Zera `documentos`, `jobs`, `documento_usuarios` e `documento_versoes`; com `--arquivos`, também `data/raw` e `data/extracted`. Mostra host e banco antes |

---

## 9. Testes e CI

```bash
uv sync --group dev
uv run pytest            # sem banco: só os unitários
```

Sem `TEST_DATABASE_URL`, os testes de integração **pulam** (79 deles). Rodar
local sem banco dá "passou" com cobertura bem menor do que parece. Suíte
completa:

```bash
docker run -d --rm --name integracar-pg-teste \
  -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=integracar_test \
  -p 55432:5432 postgres:16-alpine
export TEST_DATABASE_URL="postgresql+psycopg://test:test@localhost:55432/integracar_test"
uv run pytest
docker stop integracar-pg-teste
```

O nome do banco de teste **precisa conter `test`**: os testes se recusam a
rodar em qualquer outro, para nunca apagar dado real.

Cobertura: token de serviço, parser de log, helpers de `db/documentos.py`,
exportação CSV, fila (incluindo concorrência real do `SKIP LOCKED`), rotas
`/jobs`, `/uploads` e `/documentos`. Fora: renderização de PDF e páginas,
`/performance` e `.xlsx`.

**CI**: GitHub Actions (`.github/workflows/ci.yml`) roda a suíte inteira em
todo push e PR, com Postgres descartável no próprio job. Não roda `ruff`. Sem
deploy: build e restart em produção são manuais.

---

## 10. Convenções

- Português do Brasil em comentários, docstrings, commits, PRs e revisões.
- Revisão de código: comece com uma frase dizendo qual é o problema; cite
  `arquivo.py:linha`, nunca URL completa do GitHub; depois explique como
  reproduzir.
- Mudança em resposta de rota precisa ser combinada com a equipe do sistema de
  gestão, que consome a API.
- O repositório `integracar-frontend` está fora de uso. Não atualize nada lá.
