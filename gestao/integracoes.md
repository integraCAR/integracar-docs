# Integrações — API de extração e PDF pesquisável

Contratos entre o sistema de gestão (VPS) e os dois serviços que rodam na
workstation. Tudo aqui é comunicação **servidor-a-servidor**: o navegador nunca
toca nessas APIs e nunca vê nenhum desses tokens.

Pré-requisitos: [README.md](README.md) e [arquitetura.md](arquitetura.md).

---

## 1. As duas integrações, em uma tabela

| | API de extração | PDF pesquisável |
| --- | --- | --- |
| Repositório | `integracar-backend` + `integracar-ocr` | `PDF-Pesquisavel` |
| O que entrega | Os **campos** lidos do documento | O mesmo PDF **com camada de texto** |
| Cliente local | `services/ocr_client.py` | `services/searchable_client.py` |
| Orquestração local | `controllers/bolsista/ocr_controller.py` | `services/searchable_service.py` |
| Configuração | `util/ocr_config.py` | `util/searchable_config.py` |
| Obrigatória? | **Sim.** Variável faltando derruba o boot | **Não.** Variável faltando desliga o recurso |
| Callback local | `POST /ocr-callback` | `POST /searchable-callback` |
| Chave do callback | `job_id` | `documento_id` (o id **do serviço**, guardado em `searchable_id`) |
| Se cair | Upload falha com `502`; documentos existentes seguem utilizáveis | PDF original continua sendo servido, sem busca interna |

---

## 2. Autenticação: quatro tokens, duas direções

Não há sessão nem cookie entre as máquinas. Cada chamada leva o token no header
`X-Service-Token`, e **cada direção tem o seu**.

```
                 OCR_SERVICE_TOKEN  ───────────────────►
   integracar-gestao                                     API de extração
                 ◄───────────────  OCR_CALLBACK_TOKEN

              SEARCHABLE_SERVICE_TOKEN  ────────────────►
   integracar-gestao                                     PDF-Pesquisavel
                 ◄───────  SEARCHABLE_CALLBACK_TOKEN
```

| Token | Quem envia | Quem valida |
| --- | --- | --- |
| `OCR_SERVICE_TOKEN` | gestão, em toda chamada que faz | API de extração |
| `OCR_CALLBACK_TOKEN` | API de extração, no callback | gestão, em `/ocr-callback` |
| `SEARCHABLE_SERVICE_TOKEN` | gestão, em toda chamada que faz | PDF-Pesquisavel |
| `SEARCHABLE_CALLBACK_TOKEN` | PDF-Pesquisavel, no callback | gestão, em `/searchable-callback` |

Cada token tem que ser **idêntico nos dois `.env`**, do lado que envia e do lado
que valida. Combinar isso com a outra equipe é parte do deploy.

A validação local usa `hmac.compare_digest` — comparação em tempo constante,
para não vazar o token por diferença de tempo de resposta.

> **O erro mais comum desta integração** é trocar `OCR_SERVICE_TOKEN` por
> `OCR_CALLBACK_TOKEN`. O sintoma: o upload funciona (ou o callback funciona),
> mas o outro sentido devolve `401` silencioso e o documento fica preso em
> `na_fila` para sempre.

---

## 3. API de extração

### 3.1 Configuração

`util/ocr_config.py` lê e **exige** quatro variáveis, falhando no boot se
faltar qualquer uma — mesmo padrão de `conexao_db.py`:

| Variável | Conteúdo |
| --- | --- |
| `OCR_API_BASE` | URL base da API. Barra final é removida |
| `OCR_SERVICE_TOKEN` | Token enviado |
| `OCR_CALLBACK_TOKEN` | Token exigido no callback |
| `OCR_PUBLIC_BASE` | **Base pública própria**, usada para montar a `callback_url`. Precisa ser alcançável pela internet, porque é nela que a API de extração faz o `POST`. Em produção, lembre do prefixo: `https://www.integracar.agr.br/gestao/api` |
| `OCR_RECONCILIACAO_INTERVALO_S` | Opcional, padrão `300`. `0` desliga a reconciliação |

### 3.2 Chamadas que o gestão faz

Todas com `X-Service-Token: OCR_SERVICE_TOKEN`.

#### Envio e reprocessamento

| Chamada | Quando | Resposta |
| --- | --- | --- |
| `POST /uploads` | Upload de arquivos até 40 MB, um ou vários no mesmo `multipart`. Campos: `cod_usuario`, `callback_url`, `files[]` | Lista de `{documento_id, job_id, nome_pdf, ja_existia}`, na mesma ordem dos arquivos |
| `POST /uploads/iniciar` | Primeiro passo do upload em pedaços, para arquivo acima de 40 MB. Campos: `nome_arquivo`, `tamanho_total`, `cod_usuario` | `{upload_id}` |
| `POST /uploads/{upload_id}/chunk` | Um por pedaço de 20 MB, sequencial | — |
| `POST /uploads/{upload_id}/finalizar` | Fecha a sessão. Campo: `callback_url` | `{documento_id, job_id, nome_pdf, ja_existia}` |
| `POST /documentos/{id}/reprocessar` | Reprocessamento pedido pelo bolsista. Query: `cod_usuario`, `callback_url` | `{id, ja_existia}`, em que `id` é o `job_id` novo |

O upload em pedaços existe porque o proxy à frente da API de extração trava o
corpo de cada requisição em 64 MB (`client_max_body_size`). Limiar de 40 MB e
pedaços de 20 MB deixam folga. É sequencial de propósito: arquivo grande já é
caso raro, e paralelizar chunks não paga a complexidade.

`ja_existia: true` significa que a API deduplicou por hash: o documento já
estava lá, não houve job novo. O gestão grava o vínculo com status
`concluido` direto e `job_id` nulo.

#### Leitura de documentos e campos

| Chamada | Para que |
| --- | --- |
| `GET /documentos` | Listagem, com paginação e filtro |
| `GET /documentos/{id}` | Documento e seus campos |
| `PUT /documentos/{id}` | **Salva campos corrigidos.** Corpo `{campos, origem, cod_usuario}`. A API arquiva a versão anterior; `cod_usuario` fica gravado como autor — sem ele o histórico não sabe quem alterou |
| `GET /documentos/{id}/versoes` | Histórico de versões dos campos |
| `GET /documentos/{id}/performance` | Tempos de processamento do documento |
| `GET /documentos/edicoes-manuais` | Contagem de edições manuais, para a métrica |
| `DELETE /documentos/{id}` | Desvincula o documento de um usuário |
| `GET /documentos/export.csv` e `/export.xlsx` | Exportações |

#### Visualização

| Chamada | Para que |
| --- | --- |
| `GET /documentos/{id}/pdf` | PDF original. Repassa `Range`, devolve `206` parcial |
| `GET /documentos/{id}/pdf-info` | `{paginas, render_servidor, filtros}` |
| `GET /documentos/{id}/paginas/{n}/imagem` | Página renderizada, `?dpi=` opcional |

#### Fila e jobs

| Chamada | Para que |
| --- | --- |
| `GET /jobs` | Listagem de jobs, com `status`, `limit`, `offset` |
| `GET /jobs/{id}` | Estado de um job. **Base da reconciliação** |
| `GET /jobs/{id}/posicao` | Posição na fila |
| `DELETE /jobs/{id}` | Remove o job |
| `GET /jobs/fila-atual` | Profundidade da fila agora |
| `GET /jobs/historico` | Histórico de throughput e tempo médio. Query: `desde`, `ate`, `granularidade` |
| `GET /jobs/fila-snapshots` | Série temporal da profundidade da fila |
| `GET /performance` | Resumo agregado, `cod_usuario` opcional |

### 3.3 Síncrono ou assíncrono: a exceção do PDF

Quase todo o `ocr_client` usa `httpx.Client` síncrono. Duas funções usam
`httpx.AsyncClient`: `get_pdf` do `ocr_client` e do `searchable_client`.

O motivo é concreto. O uvicorn sobe **sem `--workers`**: há uma única thread de
event loop. Usar o cliente síncrono dentro de uma rota `async def` bloqueia
essa thread pela duração inteira da chamada de rede. E o pdf.js dispara vários
`GET` com `Range` em sequência conforme o usuário rola o documento, contra uma
máquina **física separada** (rede de verdade, não localhost). As chamadas se
enfileiram atrás umas das outras — e atrás de qualquer outra requisição que a
aplicação esteja servindo. Em documento com muitas páginas, isso basta para
algumas requisições de range estourarem timeout ou serem abortadas pelo próprio
pdf.js, e **a página fica em branco para sempre, sem tentar de novo**.

Migrar todo o `_request` para assíncrono não se pagou: dezenas de chamadas
estão fora do caminho crítico de visualização.

### 3.4 Timeouts e erros

```python
UPLOAD_TIMEOUT = httpx.Timeout(120.0, connect=10.0)   # upload e proxy de PDF
READ_TIMEOUT   = httpx.Timeout(15.0,  connect=5.0)    # leituras
```

`_verificar_resposta` traduz tudo para `OcrApiError`, com `status_code`:

| Resposta da API | O que o gestão faz |
| --- | --- |
| `401` | `logger.error` ("token de serviço rejeitado") e `OcrApiError(401)` |
| `5xx` | `OcrApiError` com o status |
| Timeout | `OcrApiError("Tempo esgotado ...")` |
| Falha de rede | `OcrApiError("Falha de comunicação ...")` |

Nos controllers, `OcrApiError` vira `502` com mensagem em português. `404` é
tratado caso a caso: na reconciliação, por exemplo, job inexistente é apenas
registrado e o laço continua.

### 3.5 Callback: `POST /ocr-callback`

Chamado **pela API de extração**. Sem sessão, sem CSRF.

```
Headers:  X-Service-Token: <OCR_CALLBACK_TOKEN>
Corpo:    { "job_id": 123, "documento_id": 456,
            "cod_usuario": 42, "status": "done" | "error",
            "error_msg": "..." }
```

| Situação | Resposta | Por que |
| --- | --- | --- |
| Token diferente | `401 {"erro": "Token inválido"}` | Comparado com `hmac.compare_digest` |
| Corpo não é JSON | `400` | |
| `job_id` ausente, ou `status` fora de `done`/`error` | `400 {"erro": "Payload inválido"}` | |
| `job_id` sem vínculo local | **`200 {"ok": true}`** | Não é responsabilidade da API de extração que o vínculo local não exista mais. Responder 2xx evita retentativa infinita |
| Vínculo já em `concluido` ou `erro` | **`200 {"ok": true}`** | Idempotência: o mesmo `job_id` pode chegar mais de uma vez |
| Caso normal | `200 {"ok": true}` | Grava `concluido` ou `erro` (com `error_msg`) |

O mapeamento é `{"done": "concluido", "error": "erro"}`
(`services/conclusao_ocr_service.py`).

### 3.6 Reconciliação: a rede de segurança

Callback se perde. Basta o processo reiniciar na hora errada, ou a rede piscar.
Por isso existe um laço assíncrono iniciado no `lifespan` de `main.py`:

```
a cada OCR_RECONCILIACAO_INTERVALO_S segundos (padrão 300):
    para cada job local ainda em na_fila ou processando:
        GET /jobs/{job_id}
          404  -> registra WARNING e segue para o próximo
          erro -> registra WARNING, INTERRUPCAO do ciclo (break), tenta depois
          done | error -> grava o status local, conta como reconciliado
```

O `break` na primeira indisponibilidade é deliberado: se a API está fora, não
faz sentido insistir em cada job do laço e multiplicar timeouts. `404` não
interrompe, porque é problema de um job específico, não do serviço.

`OCR_RECONCILIACAO_INTERVALO_S=0` desliga o laço. Nesse caso, um callback
perdido deixa o documento preso em `na_fila` **indefinidamente**.

---

## 4. PDF pesquisável

### 4.1 O que é, e o que não é

O serviço recebe o PDF original, roda OCRmyPDF (Tesseract e Ghostscript) e
devolve o **mesmo PDF com camada de texto**. É isso que faz a lupa — a busca
dentro do documento — funcionar em documento escaneado: sem camada de texto, o
visualizador não tem palavra nenhuma para casar.

Papel no sistema: **artefato de visualização, nada além disso.** Não altera o
`documento_id`, não muda hash, não substitui arquivo de origem e não interfere
na extração de campos, que continua rodando sobre o PDF original.

Roda na workstation, e não na VPS, porque OCR é rasterização página a página:
a VPS tem dois núcleos dividindo com o próprio site que os bolsistas usam.

### 4.2 Configuração opcional por projeto

`util/searchable_config.py` **não exige nada**:

```python
SEARCHABLE_ATIVO = bool(SEARCHABLE_API_BASE
                        and SEARCHABLE_SERVICE_TOKEN
                        and SEARCHABLE_CALLBACK_TOKEN)
```

Faltando qualquer uma das três, `SEARCHABLE_ATIVO` é falso e o sistema segue
exatamente como antes, servindo o PDF original.

Isso é deliberado: tornar obrigatório derrubaria no boot toda instalação que
ainda não subiu o serviço — e o PDF pesquisável é melhoria de visualização, não
requisito de funcionamento.

`SEARCHABLE_API_BASE` aponta para o mesmo host de `OCR_API_BASE`, no caminho
`/pesquisavel` (o gateway nginx do `integracar-infra` encaminha e remove o
prefixo). É **outra máquina**, não nome de container em rede Docker
compartilhada.

### 4.3 Chamadas que o gestão faz

Todas com `X-Service-Token: SEARCHABLE_SERVICE_TOKEN`.

| Chamada | Para que |
| --- | --- |
| `POST /uploads/stream` | Enfileira o OCR. Query: `cod_usuario`, `filename`, `callback_url`; corpo é o PDF bruto, `Content-Type: application/pdf`. Responde `{documento_id, job_id, status}` |
| `POST /documentos/{id}/reprocessar` | Refaz o OCR a partir do original que o próprio serviço guardou — não precisa reenviar o arquivo |
| `GET /documentos/{id}/pdf` | O PDF pesquisável pronto. Repassa `Range`, no mesmo formato de `ocr_client.get_pdf` |
| `DELETE /documentos/{id}` | Remove o pesquisável e o original guardado lá |

O upload usa **streaming de corpo bruto**, não `multipart`: é o caminho
recomendado pelo serviço para arquivo grande, e evita montar um corpo multipart
em memória para algo que já está em memória.

`get_pdf` devolve a mesma tupla
`(bytes, content-type, status, headers de range)` que `ocr_client.get_pdf` —
assim quem chama troca uma origem pela outra sem mudar mais nada. É o que
permite a preferência automática no proxy de PDF.

### 4.4 Reaproveitamento entre "irmãos"

"Irmão" é outro vínculo apontando para o **mesmo `documento_id`**. Acontece
quando dois bolsistas sobem o arquivo idêntico e a API de extração deduplica
por hash.

```
enfileirar(cod_ocr_documento, documento_id, conteudo, ...):
    se não SEARCHABLE_ATIVO: retorna
    irmao = obter_searchable_de_irmao(documento_id)
    se irmao existe:
        aponta este vínculo para o MESMO searchable_id e status  -> fim
    senão:
        POST /uploads/stream  ->  grava searchable_id, status 'pending'
```

Rodar OCR de novo no mesmo arquivo só gastaria CPU e disco em duplicidade.

### 4.5 Callback: `POST /searchable-callback`

```
Headers:  X-Service-Token: <SEARCHABLE_CALLBACK_TOKEN>
Corpo:    { "documento_id": "<searchable_id>",
            "status": "done" | "error", "error_msg": "..." }
```

| Situação | Resposta |
| --- | --- |
| `SEARCHABLE_ATIVO` falso | `404 {"erro": "Recurso não configurado"}` |
| Token diferente | `401` |
| Corpo não é JSON | `400` |
| `documento_id` ausente, ou `status` fora de `done`/`error` | `400` |
| Caso normal | `200 {"ok": true}` |

O serviço retenta **até seis vezes** se não receber 2xx, então o mesmo aviso
pode chegar mais de uma vez — o handler é idempotente, apenas regrava o status.
`error` é normalizado para `erro`, ficando no mesmo vocabulário do resto da
tabela.

Um erro aqui não é grave: o documento continua sendo servido na versão
original, sem busca.

### 4.6 Preferência automática no proxy de PDF

`GET /bolsista/ocr/{id}/pdf` decide a origem na hora:

```
se searchable_service.pronto(registro):          # searchable_status == 'done'
    tenta searchable_client.get_pdf(...)
    se falhar -> WARNING e CAI PARA o original
ocr_client.get_pdf(...)
    se falhar -> 502 {"erro": "Não foi possível carregar o PDF"}
```

A queda para o original em caso de falha é intencional: melhor um PDF sem busca
que um PDF que não abre.

---

## 5. O contrato `campos`

Os campos extraídos **não ficam no MySQL**. Vivem no Postgres de
`integracar-backend`, na coluna `campos` (`jsonb`), e chegam ao gestão pela API.

### Seções

| Chave | Conteúdo |
| --- | --- |
| `capa` | Dados da capa do processo. Inclui `is_car`, booleano derivado |
| `requerimento_digital` | Requerimento. Pode ter várias instâncias |
| `ccir` (ou `ccirs`) | Lista de objetos, uma entrada por CCIR |
| `quadro_areas` | Áreas em hectares |
| `quadro_areas_conferencia` | Conferência derivada. Metadado, não editável |
| `_fontes` | **Metadado**: a página do PDF de onde cada campo foi lido. É a âncora da busca no visualizador |

### Três armadilhas deste contrato

**1. Quem escreve é o worker, quem lê é a API — e o gestão também lê.**
Mudar a estrutura de uma seção só do lado de quem escreve quebra
silenciosamente quem lê. Já aconteceu de uma coluna ficar vazia para sempre sem
ninguém notar, porque célula vazia ali também é estado válido.

**2. `jsonb` não preserva a ordem das chaves.** Onde a ordem importa para quem
lê — planilha, listagem, painel de campos — ela precisa ser fixada
explicitamente do lado que lê. No frontend é `ORDEM_SECOES`, em
`app/ui/organisms/CamposEditor.tsx`; sem isso `quadro_areas` (12 letras)
aparecia antes de `requerimento_digital` (20), por ordenação interna por
tamanho de chave.

**3. Campo booleano editado como texto.** O `PUT .../campo` recebe sempre
string, porque o modal de edição é texto livre.
`services/campos_merge.normalizar_valor_editado` converte de volta nos campos
conhecidamente booleanos — hoje só `capa.is_car`. Sem isso, abrir o lápis do
`is_car` e confirmar gravava a string `"true"` por cima do booleano; **seis
documentos do acervo ficaram assim** antes do conserto, que hoje existe nos
dois lados (aqui na origem, e em `integracar-backend/db/documentos.py` como
rede de segurança).

### Visão combinada

`services/campos_merge.py` mescla os `campos` de todos os PDFs de um processo:

- **Seção de objeto divergente** vira uma **lista de versões**, uma por valor
  distinto, cada uma com os dados do seu PDF. Antes, o valor divergente virava
  uma lista dentro de um container único e não se sabia de qual PDF vinha cada
  um. Versões iguais, ignorando campos vazios, se fundem.
- **Campo solto na raiz**: mesmo valor em todos vira valor único; valores
  diferentes viram lista.
- **Seção de lista de objetos** (`ccir`): concatena os itens de todos os
  documentos, sem repetir item exatamente igual.
- **Exceção `capa.is_car`**: booleano derivado, não termo lido do documento. Não
  conta como divergência, e basta um PDF da versão ter `is_car=True` para a
  versão contar como CAR.

---

## 6. Diagnóstico

| Sintoma | Primeira coisa a checar |
| --- | --- |
| Documento preso em `na_fila` para sempre | Callback chegando? `grep ocr-callback logs/app.log`. Se não, confira se `OCR_PUBLIC_BASE` é alcançável de fora e se inclui `/gestao/api` em produção |
| Callback chegando com `401` | `OCR_CALLBACK_TOKEN` do gestão diferente do configurado na API de extração |
| Upload falhando com `502` | `OCR_API_BASE` alcançável? Token de serviço aceito? Ver `logger.error` com "Token de serviço OCR rejeitado" |
| Callback levando `403` (não `401`) | Isenção de CSRF não casou o caminho. Confira se a comparação usa `get_route_path` e não `request.url.path` |
| Lupa não acha nada no PDF | `searchable_status` do documento. `NULL` = nunca enviado; `pending` = ainda rodando; `erro` = OCR falhou. Em todos, o PDF servido é o original, sem camada de texto |
| PDF abre em branco em documento longo | Chamada de PDF caiu no cliente síncrono, bloqueando o event loop. Confirme que o caminho usa `_request_async` |
| Páginas corrompidas na visualização | É o pdf.js decodificando mal formato bitonal de scanner. A saída é `render_servidor: true` em `/pdf-info` |
