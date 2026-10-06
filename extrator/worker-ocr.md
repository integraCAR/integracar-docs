# Worker de OCR - `integracar-ocr-extrator`

Serviço que consome a fila de jobs no Postgres, lê cada PDF escaneado com um
modelo de visão e grava os campos estruturados de volta no banco. É o único
componente do sistema de extração que fala com o Ollama.

Não expõe HTTP. Conversa com a [API](api.md) só pela tabela `jobs` e pelo disco
compartilhado (`data/`, `logs/`). Visão geral em [README.md](README.md).

---

## Sumário

1. [Estrutura do repositório](#1-estrutura-do-repositório)
2. [O laço do worker](#2-o-laço-do-worker)
3. [O pipeline de um PDF](#3-o-pipeline-de-um-pdf)
4. [Releitura dirigida](#4-releitura-dirigida)
5. [Campos extraídos](#5-campos-extraídos)
6. [Configuração](#6-configuração)
7. [Rodar em desenvolvimento](#rodar-em-desenvolvimento)
8. [Testes e CI](#8-testes-e-ci)
9. [Antes de mexer nos extratores](#9-antes-de-mexer-nos-extratores)

---

## 1. Estrutura do repositório

```
worker.py               Laço do serviço: pega job, processa, marca done/error, heartbeat
worker_control.py       Status do worker pelo heartbeat
core/
  pdf_processor.py      PDF -> imagem de página (PyMuPDF)
  image_utils.py        Redimensiona e fatia a página; gate de qualidade da imagem
  ocr_engine.py         Chama o glm-ocr no Ollama; detecta alucinação
  releitura.py          Relê páginas quando a conferência acusa problema
  extractors/
    orchestrator.py     Ponto de entrada: regex primeiro, LLM só para lacunas
    capa.py             Capa do processo (regex)
    requerimento.py     Requerimento digital (regex)
    ccir.py             CCIR (regex + correção fuzzy de rótulos)
    quadro_areas.py     ATP/AVN/APP/ARL do recibo de inscrição no CAR
    semantico.py        Fallback com LLM (qwen2.5)
    common.py           Datas, localização do requerimento, utilitários
utils/
  file_manager.py       Saídas em data/extracted/<hash>/
  logger.py             Log de processamento por PDF (lido pela API)
  fuzzy_corrector.py    Corrige rótulos corrompidos pelo scanner (só no caminho do CCIR)
  fuzzy_matchings.json  Tabela de correções
db/                     Cópia da camada de dados da API (ver abaixo)
tests/                  Espelha core/ e utils/
Dockerfile              Imagem do worker (uv + Python 3.12). O Ollama não está nela
```

### `db/` é um espelho

O schema pertence ao `integracar-backend-extrator`. O `db/` daqui é cópia da
camada de dados de lá. Mudou o schema na API, espelhe aqui e confira, com os
repositórios lado a lado:

```bash
diff -r ../integracar-backend-extrator/db db
```

---

## 2. O laço do worker

1. Ao iniciar, devolve para `pending` todo job que esteja `running`. Como há
   um só worker, qualquer `running` no boot é órfão de uma execução anterior.
2. Abre uma thread de heartbeat que grava `data/worker.heartbeat` a cada 5 s,
   mesmo enquanto um PDF longo está sendo processado. A API usa esse arquivo
   para dizer se o worker está online.
3. A cada 3 s tenta pegar o próximo job com `SELECT ... FOR UPDATE SKIP LOCKED`.
   Isso já permite rodar vários workers em paralelo, se um dia precisar.
4. Processa o PDF (seção 3). Entre uma página e outra, confere se o job ainda
   existe; se foi apagado pelo gestão, aborta (`JobCancelado`) em vez de
   continuar por horas um trabalho que ninguém vai ler.
5. Marca `done` ou `error`, grava o histórico e dispara o callback para a
   `callback_url` do job, com `X-Service-Token: CALLBACK_TOKEN`.

---

## 3. O pipeline de um PDF

```
PDF --> página (PyMuPDF) --> fatias <= 600 px de altura, cortadas em whitespace
    --> glm-ocr (Ollama, temperatura 0, timeout por fatia)
    --> texto por página  --> data/extracted/<hash>/<hash>_resultado.json
    --> releitura dirigida, se preciso
    --> extratores regex (capa, requerimento, CCIR, quadro de áreas)
    --> qwen2.5 só para campos semânticos que ficaram vazios
    --> documentos.campos + documento_versoes (Postgres)
```

- **Fatiamento.** As dimensões são ajustadas a múltiplos de 28 px e limitadas
  em número de patches, que é o que o modelo de visão aceita. O corte procura
  uma faixa em branco entre linhas; se não encontra, corta "duro".
- **Detecção de alucinação.** A cada 2.000 caracteres o `ocr_engine` verifica
  se alguma frase se repetiu 3 vezes ou mais. Se sim, corta o texto antes da
  repetição e registra um aviso no log.
- **Progresso.** O job recebe `progress_stage` = `ocr` (com página atual e
  total) e depois `llm`.
- **Híbrido regex + LLM.** O regex extrai tudo o que é determinístico. O
  `qwen2.5` só é chamado quando algum campo semântico do requerimento (razão
  social, RG, data, elaborado por) ficou nulo. Regra fixa: sem responsável
  técnico confirmado, não há telefone do responsável técnico.
- **Rastreabilidade.** A seção `_fontes` de `campos` guarda a página de origem
  de cada campo. É o que alimenta o "ver no PDF" do sistema de gestão.

---

## 4. Releitura dirigida

Quando o fatiamento corta "duro" em cima de uma linha de valores, o modelo lê
dígitos plausíveis e errados. Exemplo real: `25.1850` e `0.2021` no lugar de
`35,1859` e `9,3031`. O erro é determinístico: reprocessar do mesmo jeito
reproduz o mesmo erro.

Ajustar o critério de corte foi medido e descartado: as quatro variantes
testadas nos 111 PDFs do histórico mudavam o corte em 22% a 92% das páginas, e
nenhum sinal de pixel separava corte inofensivo de corte destrutivo. A solução
adotada (`core/releitura.py`) usa a conferência cruzada como juiz:

1. Só roda em documento já sinalizado: divergência entre as duas fontes do
   quadro de áreas, ou telefone do interessado vazio com o rótulo presente.
2. Relê a página com outra altura de fatia, o que desloca todos os cortes.
3. Só aceita a nova leitura se ela zerar a divergência (ou recuperar o
   telefone sem perder outro campo). Senão, mantém a leitura original e o
   documento segue sinalizado.

**Não mexa no limiar de corte de `image_utils.py`.** O motivo está acima.

---

## 5. Campos extraídos

### Capa do processo (`capa`)

| Campo | Descrição |
| --- | --- |
| `objetivo` | Tipo do processo (ex.: Cadastro Ambiental Rural) |
| `is_car` | Se o processo é de CAR |
| `interessado` | Nome do interessado ou requerente |
| `cpf_cnpj` | CPF ou CNPJ do interessado |
| `codigo_empreendimento` | Código do empreendimento |
| `responsavel_recebimento` | Responsável pelo recebimento |
| `cargo_responsavel` | Cargo do responsável |

### Requerimento digital (`requerimento_digital`)

| Campo | Descrição |
| --- | --- |
| `numero` | Número do requerimento |
| `rg_interessado` | RG ou inscrição estadual |
| `telefone_interessado` | Telefone de contato |
| `codigo_empreendimento` | Código do empreendimento |
| `razao_social_empreendimento` | Nome da propriedade ou imóvel |
| `municipio_empreendimento` | Município e UF |
| `latitude`, `longitude` | Coordenadas UTM |
| `informacoes_complementares` | Observações |
| `elaborado_por` | Quem elaborou o requerimento |
| `data` | Data do requerimento |
| `responsavel_tecnico` | Responsável técnico |
| `telefone_responsavel_tecnico` | Telefone do responsável técnico |

Quando há mais de um requerimento no processo, vale o mais recente.

### CCIR (`ccir`)

| Campo | Descrição |
| --- | --- |
| `codigo_imovel_rural` | Código do imóvel rural |
| `nomes_interessados` | Declarante e titulares |
| `numero_ccir` | Número de controle do CCIR |
| `data` | Data de geração |

### Quadro de áreas (`quadro_areas`)

| Campo | Descrição |
| --- | --- |
| `area_total_propriedade` | ATP - área total da propriedade |
| `area_vegetacao_nativa` | AVN - área de vegetação nativa |
| `area_preservacao_permanente` | APP - total |
| `area_reserva_legal` | ARL - total |

A fonte principal é a seção "3. QUADRO DE ÁREAS" (singular) do recibo de
solicitação de inscrição no CAR. O OCR embaralha rótulo e valor de jeitos
diferentes, então a extração emparelha cada valor com o rótulo pendente mais
antigo (fila FIFO). Sem recibo no processo, usa a tabela "QUADROS DE ÁREAS"
(plural) do croqui/SICAR. Quando as duas fontes existem e divergem, os campos
em conflito vão para `quadro_areas_conferencia`.

---

## 6. Configuração

No Docker, as variáveis vêm do compose do [infra](infra.md). O `.env` daqui é
só para rodar fora do Docker.

| Variável | Uso |
| --- | --- |
| `DATABASE_URL` | O **mesmo** Postgres da API (a fila mora lá) |
| `OLLAMA_HOST` | Ollama com GPU. Padrão `http://localhost:11434` |
| `CALLBACK_TOKEN` | Assinatura do callback. Igual ao configurado no gestão. Sem ele o callback sai sem autenticação |
| `CALLBACK_TIMEOUT_S`, `CALLBACK_TENTATIVAS`, `CALLBACK_BACKOFF_S` | Ajuste do disparo |

Pré-requisitos no host:

- Ollama **0.16.x**. Versões 0.17 ou mais novas quebram o `glm-ocr` em
  silêncio (ver [README.md](README.md#7-cuidados-que-já-causaram-incidente)).
- Modelos: `ollama pull glm-ocr` e `ollama pull qwen2.5`.

---

## Rodar em desenvolvimento

```bash
cp .env.example .env     # aponte DATABASE_URL para o Postgres da API
uv sync
uv run python worker.py
```

O Postgres precisa estar de pé com as migrações da API aplicadas
(`uv run alembic upgrade head`, no `integracar-backend-extrator`).

Em produção o worker sobe como serviço `worker` da stack do infra
(`docker compose logs -f worker`), com teto de CPU e memória.

---

## 8. Testes e CI

```bash
uv sync --group dev
uv run pytest
```

Rodam em milissegundos: sem Postgres, sem Docker, sem GPU e **sem chamar o
Ollama** (sempre mockado com `monkeypatch`). A suíte espelha `core/`:

- `core/`: `image_utils` (redimensionamento, corte, gate de qualidade) e
  `ocr_engine` (detecção de alucinação, fluxo com Ollama mockado);
- `extractors/`: `capa`, `requerimento`, `ccir`, `quadro_areas`, `common`,
  `semantico`, `orchestrator`, `_fontes` e padronização de CPF/CNPJ;
- `test_releitura.py`, `test_fuzzy_corrector.py`, `test_normaliza_tipos.py`.

Fora de escopo: integração real com `db/`.

**CI**: GitHub Actions (`.github/workflows/ci.yml`) roda `ruff check` e
`pytest` em todo push para `main` e em todo PR. O lint é conservador de
propósito (`E4`, `E7`, `E9`, `F`). Sem deploy.

---

## 9. Antes de mexer nos extratores

- **Meça contra o corpus histórico antes de propor.** O OCR bruto de todos os
  processos já está em
  `../integracar-backend-extrator/data/extracted/<hash>/<hash>_resultado.json`.
  Mudança de extrator valida offline em segundos, sem GPU nem reprocessar.
- **Baseline correto:** `git worktree add <tmp> <commit-da-main>`, rode o
  extrator lá e compare com o HEAD. Imprima **quantos documentos foram
  comparados**: um "0 alterados" vindo de uma comparação que não casou nada já
  foi reportado como verificação.
- **Comentário registra o porquê**, com o caso real que motivou a regra,
  inclusive o valor que saiu errado. Ver `core/extractors/quadro_areas.py`. As
  regras parecem arbitrárias sem o documento que as gerou, e já houve mais de
  uma tentativa de "simplificar" uma delas de volta para o bug.
- **Mudou a estrutura de uma seção de `campos`?** Avise a API e o gestão: os
  dois leem esse JSON.
- Português do Brasil em comentários, commits, PRs e revisões.
