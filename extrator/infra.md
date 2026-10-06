# Infraestrutura da Extração — `integracar-infra-extrator`

Orquestração da stack de extração num único `docker compose`: API, worker de
OCR, Postgres, serviço de PDF pesquisável, gateway nginx, observabilidade
(Prometheus + Grafana com alertas por e-mail) e backup local e externo.

**Isto é produção.** Roda na workstation com GPU. Visão geral do sistema em
[README.md](README.md).

---

## Sumário

1. [Serviços do compose](#1-serviços-do-compose)
2. [Pré-requisitos e subida](#2-pré-requisitos-e-subida)
3. [Gateway nginx](#3-gateway-nginx)
4. [Configuração (`.env`)](#4-configuração-env)
5. [Observabilidade e alertas](#5-observabilidade-e-alertas)
6. [Backup](#6-backup)
7. [Operação](#7-operação)
8. [Rede e acesso externo](#8-rede-e-acesso-externo)
9. [Próximos passos](#9-próximos-passos)

---

## 1. Serviços do compose

| Serviço | Imagem / origem | Função |
| --- | --- | --- |
| `postgres` | `postgres:16` | Banco da extração (volume `pgdata`). Usuário e base: `integracar` |
| `api` | build de `BACKEND_PATH` | API FastAPI ([api.md](api.md)). Roda `alembic upgrade` ao subir |
| `worker` | build de `WORKER_PATH` | Worker de OCR ([worker-ocr.md](worker-ocr.md)). Teto `WORKER_CPUS`/`WORKER_MEMORY` |
| `searchable-redis` | `redis:7-alpine` | Fila e metadados do PDF pesquisável |
| `searchable-api` | build de `SEARCHABLE_PATH` | API do PDF pesquisável ([pdf-pesquisavel.md](pdf-pesquisavel.md)) |
| `searchable-worker` | build de `SEARCHABLE_PATH` | OCRmyPDF. Teto `OCR_CPUS`/`OCR_MEMORY` |
| `web` | nginx (`web/`) | Gateway, único ponto de entrada (`:8080`) |
| `frontend` | build de `FRONTEND_PATH` | SPA antiga. **Em depreciação**: a interface real é a do gestão |
| `stuck-jobs-check` | `postgres:16-alpine` | A cada 10 min conta jobs `running` há mais de 2 h |
| `fila-snapshot` | `postgres:16-alpine` | Grava o tamanho da fila em `fila_snapshots` |
| `postgres-backup` | `prodrigestivill/postgres-backup-local:16-alpine` | `pg_dump` diário com rotação |
| `backup-restore-test` | `postgres:16-alpine` | Restaura o último dump num banco descartável, toda semana |
| `offsite-backup` | `rclone/rclone:1.75.0` | Espelha os dumps no Google Drive, diariamente |
| `prometheus` | `prom/prometheus:v3.14.0` | Coleta de métricas |
| `grafana` | `grafana/grafana:13.2.0` | Painéis e alertas (`:3000`) |
| `node_exporter`, `cadvisor`, `postgres_exporter`, `blackbox_exporter` | versões fixas | Métricas do host, dos containers, do Postgres e probes HTTP |

Volumes: `pgdata`, `prometheus_data`, `grafana_data`, `searchable_redis` e
`searchable_documents`. Os diretórios `data/` e `logs/` da API e do worker são
bind mounts que apontam para dentro do clone do `integracar-backend-extrator`
(`${BACKEND_PATH}/data` e `${BACKEND_PATH}/logs`).

Todas as imagens de terceiros têm **versão fixa**, nunca `:latest`. Para
atualizar, troque a tag no `docker-compose.yml` (fica no git) e rode
`docker compose pull && docker compose up -d`.

Todo serviço usa rotação de log (`json-file`, 10 MB, 3 arquivos). Sem isso, o
log do worker, que roda o dia todo, enche o disco.

---

## 2. Pré-requisitos e subida

- Docker com Compose v2.
- Os repositórios irmãos clonados **lado a lado**: `integracar-backend-extrator`,
  `integracar-ocr-extrator`, `PDF-Pesquisavel` e este.
- Ollama **0.16.x** no host, com `glm-ocr` e `qwen2.5`.

```bash
cp .env.example .env     # preencha segredos e caminhos (seção 4)
./up.sh
```

O `up.sh` confere o Docker e o `.env`, exporta `HOST_UID`/`HOST_GID` (para os
arquivos em `data/` e `logs/` não ficarem com dono root), expõe o Ollama em
`0.0.0.0:11434` com um override do systemd (pede `sudo` na primeira vez) e roda
`docker compose up -d --build`.

| Endereço | O que é |
| --- | --- |
| `http://localhost:8080` | Gateway (API em `/api/`, PDF pesquisável em `/pesquisavel/`) |
| `http://localhost:3000` | Grafana |

O Postgres desta stack nasce **vazio** (volume próprio). Para trazer os dados
de um Postgres nativo:

```bash
PGPASSWORD=<senha> pg_dump -h localhost -p 5432 -U integracar -d integracar \
  | docker compose exec -T postgres psql -U integracar -d integracar
```

---

## 3. Gateway nginx

Configuração em `web/nginx.conf`.

| Host / rota | Destino | Observação |
| --- | --- | --- |
| `_` `/api/*` | `api:8000` | Remove o prefixo `/api`. Timeout de leitura 300 s |
| `_` `/openapi.json` | `api:8000` | Liberado só para a rede interna do Docker |
| `_` `/pesquisavel/*` | `searchable-api:8010` | Remove o prefixo. Corpo até 300 MB, timeouts de 1.800 s |
| `_` `/` | `frontend:80` | SPA em depreciação |
| `api.integracar.agr.br` `/v1/*` | `api:8000` | Só a API pública. Nenhuma outra rota é alcançável por esse host |

O host público fica num `server {}` separado de propósito: um parceiro externo
não alcança as rotas internas nem por engano.

---

## 4. Configuração (`.env`)

O compose **recusa subir** sem as variáveis marcadas como obrigatórias.

| Variável | Obrigatória | Uso |
| --- | --- | --- |
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | Sim | Banco |
| `SERVICE_TOKEN` | Sim | Token que o gestão envia à API (`openssl rand -hex 32`) |
| `CALLBACK_TOKEN` | Recomendada | Assinatura do callback de conclusão |
| `SEARCHABLE_SERVICE_TOKEN`, `SEARCHABLE_CALLBACK_TOKEN` | Sim | Tokens do PDF pesquisável |
| `GRAFANA_ADMIN_PASSWORD` | Sim | Senha do usuário `admin` do Grafana |
| `GRAFANA_SMTP_USER`, `GRAFANA_SMTP_PASSWORD` | Sim | Conta Gmail dedicada, com senha de app, para os alertas |
| `BACKEND_PATH`, `WORKER_PATH`, `SEARCHABLE_PATH`, `FRONTEND_PATH` | Na prática, sim | Caminhos dos repositórios irmãos |
| `WORKER_CPUS`, `WORKER_MEMORY` | Não | Teto do worker. Padrão 12 núcleos / 16G |
| `OCR_JOBS`, `OCR_CPUS`, `OCR_MEMORY` | Não | Paralelismo e teto do OCRmyPDF |
| `HOST_UID`, `HOST_GID` | Não | Preenchidos pelo `up.sh` |

**Os caminhos padrão do compose estão desatualizados** (`../lucasaltoe`,
`../integracar-ocr`, `../integracar-frontend`). Com os clones atuais, use:

```bash
BACKEND_PATH=../integracar-backend-extrator
WORKER_PATH=../integracar-ocr-extrator
SEARCHABLE_PATH=../PDF-Pesquisavel
```

Os tokens `SERVICE_TOKEN`, `CALLBACK_TOKEN` e `SEARCHABLE_*` precisam ser
idênticos aos do `.env` do `integracar-gestao`. O destino dos e-mails de alerta
não é variável: fica em `grafana/provisioning/alerting/contactpoints.yaml`.

O `.env` tem credenciais reais. Não cole valor de segredo em commit, PR ou
issue; para conferir uma variável, leia o nome, não o valor.

---

## 5. Observabilidade e alertas

**Prometheus** (`prometheus/prometheus.yml`) coleta de: node_exporter (host),
cAdvisor (containers), postgres_exporter, blackbox_exporter (`/health` da API e
o Ollama) e o `nvidia_gpu_exporter` rodando no host (porta 9835).

**Grafana** sobe já provisionado: data source, painel "IntegraCAR - Visão
Geral" na pasta IntegraCAR e regras de alerta. Notificação por e-mail via
Gmail, com horário em `America/Sao_Paulo` (template próprio, porque o padrão é
UTC). Só o Grafana publica porta no host, para a ponte com a VPN.

Alertas configurados. O que fazer em cada um está no `RUNBOOK.md` do
repositório:

| Alerta | Origem |
| --- | --- |
| API fora do ar | blackbox em `/health` |
| Ollama fora do ar | blackbox |
| Disco do host com menos de 10% livre | node_exporter |
| Postgres fora do ar | postgres_exporter |
| Exporter da GPU fora do ar | job `gpu` |
| Teste de restore falhou / atrasado (mais de 8 dias) | `integracar_restore_test_*` |
| Sync offsite falhou / atrasado (mais de 2 dias) | `integracar_offsite_backup_sync_*` |
| Worker perto do limite de memória (mais de 80% por 5 min) | cAdvisor |
| Job travado na fila (mais de 2 h em `running`) / checagem atrasada (mais de 1 h) | `integracar_stuck_jobs_*` |

O limite de 2 h para job travado dá folga sobre o maior job já concluído (cerca
de 78 min). A checagem só faz `SELECT` na tabela `jobs`.

---

## 6. Backup

| Camada | Serviço | Detalhe |
| --- | --- | --- |
| Local | `postgres-backup` | `pg_dump` diário, comprimido, em `./backups/`. Mantém 14 diários, 4 semanais, 6 mensais. Permissão `700`, fora do git |
| Verificação | `backup-restore-test` | Toda semana restaura o dump mais recente em `integracar_restore_test`, que é destruído em seguida. Nunca toca o banco real |
| Externo | `offsite-backup` | `rclone` espelha `./backups` em `IntegraCAR/backups/postgres/` no Google Drive, com escopo `drive.file` |

**Os PDFs originais (`data/raw`) não entram em nenhum backup.** Só o banco.

A credencial do rclone fica em `rclone/rclone.conf` (fora do git). Para
recriar: `rclone config`, tipo `drive`, escopo `drive.file`, remote chamado
`gdrive_backup`.

Restaurar manualmente:

```bash
# do backup local
gunzip -c backups/daily/integracar-<data>.sql.gz | \
  docker compose exec -T postgres psql -U integracar -d integracar

# baixar do Google Drive
rclone copy gdrive_backup:IntegraCAR/backups/postgres ./backups --config rclone/rclone.conf
```

---

## 7. Operação

### Antes de qualquer deploy

O código roda a partir da imagem, não do disco. Antes de `build`/`up -d` na
`api` ou no `worker`, **confira se a fila está vazia**:

```bash
docker compose exec -T postgres psql -U integracar -d integracar -t -A \
  -c "SELECT count(*) FROM jobs WHERE status IN ('running','pending');"
```

### Comandos do dia a dia

```bash
docker compose ps
docker compose logs -f worker
docker compose stop worker && docker compose start worker
docker compose down
```

### Testar código de uma branch sem subir para produção

```bash
docker compose cp arquivo.py worker:/app/caminho/arquivo.py
# testar
docker compose up -d --force-recreate worker   # volta para a imagem original
```

### Job travado

O worker pode estar saudável e preso num documento específico. Siga a seção
"Job travado na fila de extração" do `RUNBOOK.md`. Para devolver órfãos à
fila, a API tem `POST /jobs/reset-orfaos`.

### PDF pesquisável

```bash
docker compose exec searchable-api python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8010/health').read().decode())"
docker compose logs -f searchable-worker
```

---

## 8. Rede e acesso externo

O acesso público (`metrics.integracar.agr.br`, `api.integracar.agr.br` e o
consumo da API pelo gestão) passa por uma camada de **VPN + nginx da VPS**
mantida fora destes repositórios, por outra pessoa. Ela cuida do TLS e de
restringir quem alcança a workstation. Não há Certbot nem configuração de VPN
aqui. O `nginx.conf` e o `GF_SERVER_ROOT_URL` do Grafana só precisam saber os
domínios finais.

---

## 9. Próximos passos

Hoje `api`, `worker`, `frontend`, `searchable-api` e `searchable-worker` são
**buildados** a partir dos repositórios irmãos. Para um deploy mais robusto:

1. cada repositório publica a própria imagem num registry, via CI;
2. este compose passa a usar `image:` em vez de `build:`;
3. credenciais vêm de secrets do ambiente, não de `.env` em disco.

Também está pendente remover os serviços `frontend` e a rota `/` do gateway,
já que a interface é a do sistema de gestão.
