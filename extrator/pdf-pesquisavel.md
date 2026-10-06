# PDF Pesquisável - `PDF-Pesquisavel`

Serviço assíncrono que transforma PDFs escaneados em PDFs pesquisáveis, com
camada de texto. A API recebe os arquivos e guarda os metadados no Redis; um
worker RQ roda OCRmyPDF/Tesseract, armazena o PDF final e avisa o sistema
chamador por callback.

Visão geral do sistema de extração em [README.md](README.md).

---

## Sumário

1. [Papel no IntegraCAR](#1-papel-no-integracar)
2. [Arquitetura](#2-arquitetura)
3. [API](#3-api)
4. [Estratégia de OCR](#4-estratégia-de-ocr)
5. [Configuração](#5-configuração)
6. [Implantação](#6-implantação)
7. [Desenvolvimento e testes](#7-desenvolvimento-e-testes)
8. [Operação](#8-operação)
9. [Segurança](#9-segurança)

---

## 1. Papel no IntegraCAR

Sem camada de texto, o visualizador do sistema de gestão não tem palavra
nenhuma para encontrar, e a busca dentro do PDF não funciona. Isso inclui a
lupa e o "ver no PDF" de cada campo extraído.

```
bolsista envia PDF
   -> integracar-gestao (VPS)
        ├─ integracar-backend-extrator (workstation) .. extração de campos
        └─ PDF-Pesquisavel (workstation) .............. camada de texto

visualização
   integracar-gestao (VPS) <- PDF já pesquisável <- workstation
```

- Os dois caminhos são **independentes**. A extração de campos roda sobre o
  PDF original; este serviço não muda o hash nem substitui o arquivo de origem.
- Quando o PDF pesquisável fica pronto, o gestão passa a servi-lo no lugar do
  original.
- Se este serviço estiver fora do ar, o gestão serve o PDF original e segue
  funcionando, só sem busca dentro do documento.

---

## 2. Arquitetura

| Serviço | Função |
| --- | --- |
| `searchable-api` | FastAPI/Uvicorn na porta `8010` |
| `searchable-worker` | Consome a fila RQ `searchable-pdf` e executa o OCRmyPDF |
| `redis` | Fila, jobs e metadados |

Arquivos no volume `searchable_documents`; dados do Redis no volume
`searchable_redis`.

```
app/
  main.py            API e endpoints
  worker.py          Execução assíncrona do OCR
  pdf_prepare.py     Escolha da estratégia redo/force
  poppler_ocr.py     Renderização com Poppler e fatiamento para o Tesseract
  redis_store.py     Fila e metadados no Redis
  storage.py         Armazenamento em partes e SHA-256
  callbacks.py       Callback com retentativas
  range_response.py  Download parcial (Range)
  auth.py            Conferência do X-Service-Token
  config.py          Leitura das variáveis de ambiente
docs/                Notas de versão e deduplicação
docker-compose.yml   API, worker, Redis e volumes (só para desenvolvimento)
Dockerfile           Imagem baseada em OCRmyPDF 17.10
testar.py            Testes manuais contra a API
teste_terminal.ps1   Menu de testes para Windows
testar.bat           Atalho para o menu PowerShell
```

---

## 3. API

Todos os endpoints, exceto `GET /health` e a documentação OpenAPI
(`/docs`), exigem `X-Service-Token: <SEARCHABLE_SERVICE_TOKEN>`. Sem ele, `401`.

| Método | Endpoint | Descrição |
| --- | --- | --- |
| `GET` | `/health` | Verifica a API e a conexão com o Redis. Resposta: `{"ok": true}` |
| `POST` | `/uploads` | Um ou vários PDFs, multipart |
| `POST` | `/uploads/stream` | Um PDF como corpo bruto (`application/pdf`). Recomendado para arquivos grandes |
| `GET` | `/jobs/{job_id}` | Metadados pelo job |
| `GET` | `/jobs/{job_id}/metricas` | Dados prontos para o gráfico de OCR por página |
| `GET` | `/documentos/{document_id}` | Metadados pelo documento |
| `GET` | `/documentos/{document_id}/pdf` | PDF pesquisável, com suporte a `Range` |
| `GET` | `/documentos/{document_id}/texto` | Até 2.000 caracteres do texto reconhecido |
| `POST` | `/documentos/{document_id}/reprocessar` | Novo job a partir do PDF original |
| `DELETE` | `/documentos/{document_id}` | Remove metadados e arquivos |

Parâmetros do upload: `cod_usuario` (obrigatório), `filename` (padrão
`documento.pdf`) e `callback_url` (opcional).

Status: `pending`, `running`, `done`, `error` (ver `error_msg`).

O status do documento também traz métricas de desempenho: `ocr_page_count`,
`ocr_elapsed_ms`, `ocr_ms_per_page`, `ocr_attempt_count` e `ocr_page_timings`
(tempo do Tesseract por página e tentativa).

### Callback

Ao concluir ou falhar, o worker faz `POST` na `callback_url` com
`X-Service-Token: <SEARCHABLE_CALLBACK_TOKEN>`. Tenta até seis vezes, com
espera progressiva, então o receptor precisa ser idempotente. No gestão, o
receptor é `POST /searchable-callback`
(ver [gestao/integracoes.md](../gestao/integracoes.md)).

### Deduplicação

O SHA-256 é calculado enquanto o upload é gravado, sem reler o arquivo nem
carregá-lo inteiro em memória, e devolvido nos metadados. A **deduplicação por
hash está desativada** na versão atual: a chamada a `claim_content_hash` está
comentada em `app/main.py`, então cada upload cria documento e job novos, mesmo
com conteúdo repetido. `docs/DEDUPLICACAO.md` e `docs/VERSAO_V8.txt` descrevem
o comportamento de quando ela estava ligada. O reaproveitamento entre
documentos iguais é feito do lado do gestão.

---

## 4. Estratégia de OCR

O original é sempre preservado. A estratégia depende do formulário encontrado
no PDF:

1. Sem AcroForm: `--mode redo`.
2. AcroForm vazio: `--mode redo`.
3. AcroForm só com campos de assinatura (`/Sig`), comum nos PDFs do E-Docs:
   cria uma cópia intermediária sem esses campos e usa `--mode redo`.
4. Formulário interativo ou XFA: `--mode force`.
5. Se o `redo` for recusado por causa de formulário, tenta de novo com `force`.

A cópia intermediária é apagada no fim.

---

## 5. Configuração

| Variável | Obrigatória | Padrão | Descrição |
| --- | --- | --- | --- |
| `SEARCHABLE_SERVICE_TOKEN` | Sim | - | Autentica as chamadas recebidas |
| `SEARCHABLE_CALLBACK_TOKEN` | Sim | - | Autentica os callbacks enviados |
| `REDIS_URL` | Não | `redis://redis:6379/0` | Conexão Redis |
| `SEARCHABLE_STORAGE_ROOT` | Não | `/data/documents` | Diretório dos documentos no container |
| `MAX_UPLOAD_MB` | Não | `0` | Limite por arquivo; `0` desativa o limite da aplicação |
| `OCR_LANGUAGE` | Não | `por` | Idioma do Tesseract |
| `OCR_JOBS` | Não | `2` | Paralelismo do OCRmyPDF |
| `OCR_TIMEOUT_SECONDS` | Não | `21600` | Prazo total por documento (tentativa normal + fallback) |
| `OCR_JOB_TIMEOUT_GRACE_SECONDS` | Não | `300` | Margem para limpeza, status e callback |
| `OCR_TESSERACT_TIMEOUT_SECONDS` | Não | `14` | Limite do Tesseract por página |
| `OCR_TESSERACT_NON_OCR_TIMEOUT_SECONDS` | Não | `14` | Limite para orientação e operações auxiliares |
| `OCR_TESSERACT_RENDER_DPI` | Não | `150` | Em `force`, DPI da imagem enviada ao Tesseract; `0` desativa |
| `OCR_TESSERACT_PAGESEGMODE` | Não | `4` | Modo de segmentação (0 a 13); `4` funciona melhor em documento digitalizado |
| `OCR_TESSERACT_SLICE_HEIGHT` | Não | `900` | Altura máxima das faixas enviadas ao Tesseract; `0` desativa |
| `OCR_NORMALIZE_TEXT_BOXES` | Não | `1` | Reduz caixas de seleção anormalmente altas |
| `OCR_ROTATE_PAGES` | Não | `0` | Corrige rotação |
| `OCR_DESKEW` | Não | `0` | Corrige inclinação |
| `OCR_OVERSAMPLE` | Não | `0` | DPI de oversampling; `0` desativa |
| `OCR_OPTIMIZE` | Não | `1` | Otimização do PDF final (0 a 3) |

Rotação, deskew e oversampling custam CPU, tempo e espaço. Ative só se a
qualidade dos scans justificar.

---

## 6. Implantação

Em produção, o serviço **sobe junto da stack da workstation**, pelo compose do
[`integracar-infra-extrator`](infra.md): serviços `searchable-redis`,
`searchable-api` e `searchable-worker`, buildados de `SEARCHABLE_PATH`, expostos
pelo gateway em `/pesquisavel/`. O `docker-compose.yml` deste repositório é só
para desenvolvimento.

No `.env` do `integracar-gestao` (VPS), com os **mesmos** tokens do infra:

```bash
SEARCHABLE_API_BASE=https://<host-da-workstation>/pesquisavel
SEARCHABLE_SERVICE_TOKEN=<mesmo valor do .env do infra>
SEARCHABLE_CALLBACK_TOKEN=<mesmo valor do .env do infra>
```

Sem essas três variáveis o gestão não usa o recurso, e nada quebra.

Os tetos de recurso (`OCR_CPUS`, `OCR_MEMORY`) existem porque a extração de
campos é a prioridade da máquina: se um OCR pesado estourar o teto, morre esse
job, em vez de o OCRmyPDF disputar CPU com o modelo de extração.

---

## 7. Desenvolvimento e testes

Requisitos: Docker com Compose v2. OCRmyPDF, Tesseract e as dependências Python
vêm na imagem.

```bash
cp .env.example .env
python -c "import secrets; print(secrets.token_urlsafe(48))"   # um para cada token
docker compose up -d --build
curl http://localhost:8010/health
```

OpenAPI em `http://localhost:8010/docs`.

Upload de teste por streaming:

```bash
curl -X POST "http://localhost:8010/uploads/stream?cod_usuario=1&filename=arquivo.pdf" \
  -H "X-Service-Token: $SEARCHABLE_SERVICE_TOKEN" \
  -H "Content-Type: application/pdf" \
  --data-binary "@arquivo.pdf"
```

No Windows, `testar.bat` abre um menu PowerShell para processar um PDF, uma
pasta inteira e acompanhar os logs. Os resultados ficam em `resultados/`.

Quando o callback roda no host, use um endereço que o container alcance, como
`http://host.docker.internal:PORTA/...`, e não `localhost`.

---

## 8. Operação

```bash
docker compose ps
docker compose logs -f searchable-api
docker compose logs -f searchable-worker
docker compose restart searchable-api searchable-worker
docker compose up -d --build searchable-api                     # mudou só a API
docker compose up -d --build searchable-api searchable-worker   # mudou o worker ou dependências
```

O código não é montado como volume e o Uvicorn não usa `--reload`: reiniciar
sem build não aplica mudanças.

`docker compose down -v` **apaga** PDFs, metadados e a fila.

---

## 9. Segurança

- Nunca versione o `.env`.
- Tokens longos, aleatórios e **diferentes** para API e callback.
- Em produção, exponha só por HTTPS atrás do proxy, e aplique também lá o
  limite de upload.
- Restrinja o acesso à porta `8010` e ao Redis.
- Valide autenticação e idempotência no receptor do callback.
