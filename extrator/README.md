# Sistema de Extração IntegraCAR

Conjunto de serviços que lê os PDFs escaneados dos processos de CAR e devolve
os dados estruturados: interessado, CPF/CNPJ, município, requerimento, CCIR,
quadro de áreas. Também gera uma versão pesquisável de cada PDF, com camada de
texto.

Ninguém abre este sistema no navegador. Quem o usa é o
[sistema de gestão](../gestao/README.md), que envia os PDFs, recebe o aviso de
conclusão e mostra os campos para o bolsista revisar.

---

## Sumário

1. [Os quatro repositórios](#1-os-quatro-repositórios)
2. [Como funciona, por cima](#2-como-funciona-por-cima)
3. [O caminho de um PDF](#3-o-caminho-de-um-pdf)
4. [Autenticação entre sistemas](#4-autenticação-entre-sistemas)
5. [Stack](#5-stack)
6. [Clonar e subir](#6-clonar-e-subir)
7. [Cuidados que já causaram incidente](#7-cuidados-que-já-causaram-incidente)
8. [Documentação detalhada](#8-documentação-detalhada)

---

## 1. Os quatro repositórios

| Repositório | Papel | Detalhes |
| --- | --- | --- |
| `integracar-backend-extrator` | API FastAPI + Postgres. Dono da fila de jobs, dos documentos extraídos, do histórico de versões e do **schema do banco** (Alembic) | [api.md](api.md) |
| `integracar-ocr-extrator` | Worker: consome a fila, roda o OCR (GLM-OCR via Ollama) e extrai os campos (regex + LLM) | [worker-ocr.md](worker-ocr.md) |
| `integracar-infra-extrator` | Sobe tudo num `docker compose`: API, worker, Postgres, gateway nginx, PDF pesquisável, Prometheus/Grafana, backups | [infra.md](infra.md) |
| `PDF-Pesquisavel` | Serviço independente que aplica OCRmyPDF/Tesseract e devolve o PDF com camada de texto | [pdf-pesquisavel.md](pdf-pesquisavel.md) |

Os quatro rodam na **workstation** (a máquina com GPU). O sistema de gestão
roda na VPS e chama a workstation por HTTPS.

> Os repositórios mudaram de nome. No código ainda aparecem `integracar-backend`,
> `integracar-backend-ocr`, `integracar-ocr`, `integracar-solucao-ocr` e
> `integracar-infra-ocr`. Ver a [tabela de nomes antigos](../README.md#nomes-antigos-dos-repositórios).
> O repositório `integracar-frontend` (SPA React) está **fora de uso**: a interface
> real é a do sistema de gestão.

---

## 2. Como funciona, por cima

```
 integracar-gestao (VPS)
     |  HTTPS + X-Service-Token
     v
 gateway nginx :8080  (integracar-infra-extrator/web/nginx.conf)
     |-- /api/*          --> api  (integracar-backend-extrator)
     |                        |  grava job
     |                        v
     |                     Postgres  <-- claim (FOR UPDATE SKIP LOCKED)
     |                        ^           worker (integracar-ocr-extrator)
     |                        |  grava campos       |
     |                        +---------------------+--> Ollama no host (GPU)
     |                                              |     glm-ocr, qwen2.5
     |                                              +--> callback para o gestão
     |
     |-- /pesquisavel/*  --> searchable-api (PDF-Pesquisavel) --> Redis/RQ --> searchable-worker
     |
 api.integracar.agr.br
     `-- /v1/*           --> API pública (clientes externos, X-Api-Key)
```

Pontos que valem a pena guardar:

- **API e worker não se chamam por HTTP.** O canal entre eles é a tabela
  `jobs` do Postgres e o disco compartilhado (`data/`, `logs/`, por bind mount).
- **O schema é da API.** Só `integracar-backend-extrator` roda migrações. O
  diretório `db/` do worker é uma cópia da camada de dados da API e precisa ser
  espelhado à mão quando o schema muda.
- **A identidade de usuário é do sistema de gestão.** A API de extração não
  tem login. O `cod_usuario` do gestão viaja como parâmetro no upload e nas
  listagens.
- **Extração de campos e PDF pesquisável são independentes.** Filas
  separadas (Postgres de um lado, Redis do outro). A extração roda sempre
  sobre o PDF original; o PDF pesquisável é só um artefato de visualização.
- **O Ollama roda no host, não em container.** Os containers falam com ele
  por `host.docker.internal:11434`.

---

## 3. O caminho de um PDF

1. **Upload.** O gestão faz `POST /uploads` (ou o upload em partes:
   `/uploads/iniciar`, `/uploads/{id}/chunk`, `/uploads/{id}/finalizar`) com
   o PDF, o `cod_usuario` e uma `callback_url`.
2. **Deduplicação por hash.** A API calcula o SHA-256. Se o documento já
   existe, só cria o vínculo do usuário com ele e não reprocessa. Se é novo,
   grava o PDF em `data/raw/<hash>.pdf`, cria a linha em `documentos` e
   enfileira um job em `jobs`.
3. **OCR.** O worker pega o job (`SELECT ... FOR UPDATE SKIP LOCKED`),
   renderiza cada página, corta em fatias e manda cada fatia para o modelo
   `glm-ocr`. O progresso é gravado página a página no job.
4. **Releitura dirigida.** Se a conferência do quadro de áreas acusa
   divergência entre as duas fontes do documento, ou se o telefone do
   interessado sumiu, o worker relê a página com outra altura de fatia e só
   aceita a nova leitura se ela resolver o problema.
5. **Extração de campos.** Regex extrai capa, requerimento, CCIR e quadro de
   áreas. O LLM `qwen2.5` só entra para preencher campos semânticos que o regex
   deixou vazios.
6. **Gravação.** Os campos vão para a coluna `campos` (JSONB) de `documentos`,
   e uma cópia vai para `documento_versoes` (histórico completo).
7. **Callback.** O worker faz `POST` na `callback_url` com o header
   `X-Service-Token: CALLBACK_TOKEN`. Se o callback se perder, o gestão ainda
   pode consultar `GET /jobs/{id}`.
8. **PDF pesquisável, em paralelo.** O gestão manda o mesmo PDF para
   `/pesquisavel/uploads`. Quando o OCRmyPDF termina, o serviço chama o
   callback do gestão, que passa a servir a versão pesquisável.

---

## 4. Autenticação entre sistemas

| Segredo | Quem envia | Quem confere | Para quê |
| --- | --- | --- | --- |
| `SERVICE_TOKEN` | gestão | API de extração | Toda rota interna (exceto `/health`) |
| `CALLBACK_TOKEN` | worker | gestão | Callback de conclusão da extração |
| `SEARCHABLE_SERVICE_TOKEN` | gestão | PDF-Pesquisavel | Upload e consulta do PDF pesquisável |
| `SEARCHABLE_CALLBACK_TOKEN` | worker do PDF-Pesquisavel | gestão | Callback de conclusão do PDF pesquisável |
| `X-Api-Key` (uma por cliente) | cliente externo | API pública `/v1` | Clientes fora do gestão; cada um só vê os próprios documentos |

Os tokens precisam ser **idênticos** nos dois lados (`.env` do
`integracar-infra-extrator` e `.env` do `integracar-gestao`). Divergindo, o
sintoma é `401`: no envio, ou no callback, caso em que o documento fica parado
em `pending`. Detalhes do lado do gestão em
[gestao/integracoes.md](../gestao/integracoes.md).

---

## 5. Stack

| Componente | Tecnologia |
| --- | --- |
| API | Python 3.12, FastAPI, Uvicorn, SQLAlchemy 2, Alembic, PyMuPDF, openpyxl, slowapi |
| Banco | PostgreSQL 16 |
| Worker | Python 3.12, Ollama (`glm-ocr` para OCR, `qwen2.5` para extração semântica), Pillow, NumPy, PyMuPDF |
| PDF pesquisável | FastAPI, Redis 7, RQ, OCRmyPDF 17.10, Tesseract (`por`) |
| Infra | Docker Compose, nginx, Prometheus, Grafana, node_exporter, cAdvisor, postgres_exporter, blackbox_exporter, rclone |
| Gestão de dependências | `uv` (API e worker), `pip` (PDF-Pesquisavel) |

---

## 6. Clonar e subir

Os repositórios precisam estar **lado a lado**, porque o compose do infra
builda os outros por caminho relativo:

```bash
mkdir -p ~/gits/integraCAR && cd ~/gits/integraCAR
for r in integracar-backend-extrator integracar-ocr-extrator \
         integracar-infra-extrator PDF-Pesquisavel; do
  gh repo clone "integraCAR/$r"
done

cd integracar-infra-extrator
cp .env.example .env
```

No `.env`, além dos segredos, **ajuste os caminhos**. Os valores padrão do
`docker-compose.yml` ainda usam nomes antigos e não batem com os clones atuais:

```bash
BACKEND_PATH=../integracar-backend-extrator
WORKER_PATH=../integracar-ocr-extrator
SEARCHABLE_PATH=../PDF-Pesquisavel
```

Depois:

```bash
./up.sh      # expõe o Ollama na rede (sudo na 1ª vez) e sobe a stack
```

Pré-requisito no host: Ollama **0.16.x** com os modelos `glm-ocr` e `qwen2.5`
(ver a seção 7).

Para desenvolver só a API ou só o worker, sem a stack inteira, ver
[api.md](api.md#rodar-em-desenvolvimento) e
[worker-ocr.md](worker-ocr.md#rodar-em-desenvolvimento).

---

## 7. Cuidados que já causaram incidente

- **Versão do Ollama.** Só a faixa **0.16.x** funciona com o `glm-ocr`. Da 0.17
  em diante (confirmado até a 0.30.10) o modelo devolve texto alucinado ou em
  chinês, e o job termina como "sucesso". Já aconteceu duas vezes. Se a
  qualidade cair do nada, confira `ollama --version` antes de qualquer outra
  coisa. Instalação: `curl -fsSL https://ollama.com/install.sh | OLLAMA_VERSION=0.16.3 sh`.
- **A workstation é produção.** Antes de `build`/`up -d` na `api` ou no
  `worker`, confira se a fila está vazia. Subir no meio de um job perde horas
  de OCR (comando em [infra.md](infra.md#antes-de-qualquer-deploy)).
- **`campos` é JSON livre.** Quem grava é o worker; quem lê é a API e o
  gestão. Mudar a estrutura de uma seção no worker quebra em silêncio a
  listagem e as exportações da API. E o `jsonb` não preserva a ordem das
  chaves: onde a ordem importa, ela é fixada no código.
- **`db/` espelhado.** Mudou o schema na API, espelhe no worker. Conferir com
  `diff -r ../integracar-backend-extrator/db db`, de dentro do
  `integracar-ocr-extrator`. Hoje os dois `db/` já não são idênticos
  (`documentos.py` e `__init__.py` diferem, e `clientes_api.py` só existe na
  API); vale conferir se a divergência é intencional.

---

## 8. Documentação detalhada

| Documento | Conteúdo |
| --- | --- |
| [api.md](api.md) | Rotas internas e públicas, autenticação, tabelas, migrações, testes, CI |
| [worker-ocr.md](worker-ocr.md) | Pipeline, extratores, releitura dirigida, campos extraídos, testes |
| [infra.md](infra.md) | Serviços do compose, gateway, observabilidade, alertas, backup, operação |
| [pdf-pesquisavel.md](pdf-pesquisavel.md) | API do serviço, estratégia de OCR, configuração, operação |
