# Backend — API FastAPI

Catálogo dos 83 endpoints, autenticação, middlewares e convenções de resposta.

Pré-requisitos: [README.md](README.md) e [arquitetura.md](arquitetura.md).

Medido em 2026-10-06. Para reconferir a contagem:
`grep -rh '^@router\.' routes | wc -l`

---

## 1. Montagem do app

`main.py` faz tudo em ordem: carrega `.env`, configura log, detecta ambiente,
define o `lifespan`, registra handlers de exceção, registra os middlewares e
inclui os routers.

### `lifespan`

Na subida: inicializa o pool MySQL e dispara a tarefa de reconciliação
periódica de jobs de OCR. Na descida: cancela a tarefa e solta o pool.

### `root_path` em produção

```python
if IS_PRODUCTION:
    app_args["root_path"] = "/gestao/api"
```

`root_path` ajusta o que o FastAPI anuncia no OpenAPI e nos links do `/docs`;
os caminhos reais continuam servidos na raiz do app. Quem monta o prefixo é o
nginx. Consequência prática: de dentro da rede Docker, `http://backend:8000/login`
funciona sem prefixo; de fora, o caminho é
`https://www.integracar.agr.br/gestao/api/login`.

### Documentação interativa

| Caminho | Conteúdo |
| --- | --- |
| `/docs` | Swagger UI |
| `/redoc` | ReDoc |
| `/openapi.json` | Esquema OpenAPI |

Em produção esses caminhos ficam sob `/gestao/api/`. **Não há autenticação
sobre eles.**

---

## 2. Middlewares

Registrados em `main.py`, executam na ordem inversa do registro. Para a
requisição que chega, a ordem efetiva é:

| # | Middleware | O que faz |
| --- | --- | --- |
| 1 | `catch_exceptions_middleware` | Exceção não tratada vira `500 {"erro": "Erro interno do servidor. Tente novamente mais tarde."}`, com `logger.exception` antes |
| 2 | `security_headers_middleware` | `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `X-XSS-Protection: 1; mode=block`. Em produção acrescenta `Content-Security-Policy` e `Strict-Transport-Security` (1 ano, `includeSubDomains`) |
| 3 | `csrf_middleware` | `GET`, `HEAD`, `OPTIONS`, `TRACE` só garantem que existe token na sessão. Os demais exigem `X-CSRF-Token` presente e válido, ou `403` |
| 4 | `request_context_middleware` | `request_id` (UUID4), `user_id`, `role`, `ip`, `user_agent` em `request.state.log_extra` |
| 5 | `SessionMiddleware` | Cookie `integracar_session`, assinado com `SECRET_KEY`, `max_age` 2 592 000 s (30 dias), `same_site="lax"`, `https_only` em produção |
| 6 | `CORSMiddleware` | Em produção, só `FRONTEND_URL`. Em desenvolvimento, também `localhost:3000`, `:3001`, `:5173` e os equivalentes em `127.0.0.1`. `expose_headers=["X-Total-Count"]` |

### Isenções de CSRF

`/login`, `/ocr-callback`, `/searchable-callback`.

- `/login` não tem sessão prévia de onde carregar token. Protegido por rate
  limit de 5 por minuto por IP.
- Os dois callbacks são webhooks servidor-a-servidor, sem cookie e sem de onde
  tirar token CSRF. Autenticados por `X-Service-Token`.

A comparação usa `get_route_path(request.scope)` justamente porque
`request.url.path` carrega o `root_path` em produção — ver
[arquitetura.md, seção 3](arquitetura.md#detalhes-que-já-causaram-incidente).

### `expose_headers=["X-Total-Count"]`

Sem isso o navegador recebe o header mas o JavaScript não consegue lê-lo em
requisição cross-origin (frontend em `:5173`, API em `:8000`). É usado pela
paginação de `/bolsista/ocr/consolidado` e `/bolsista/ocr/jobs`.

---

## 3. Autenticação e autorização

### `@requer_autenticacao(perfis)`

Definido em `util/auth_decorator.py`, aplicado **abaixo** do decorador de rota:

```python
@router.get("/bolsista/ocr")
@requer_autenticacao(["bolsista", "coordenador"])
async def listar_documentos_ocr(request: Request, usuario_logado: dict = None):
    ...
```

Passo a passo:

1. Localiza o `Request` entre os argumentos (posicionais ou nomeados). Não
   encontrando, `500`.
2. `obter_usuario_logado(request)` lê `session["usuario"]` e valida duas
   expirações: a absoluta (`_expira_em`) e a de inatividade
   (`_last_activity`, limite de **90 minutos**). Expirado, limpa a sessão.
3. Sem usuário: `401 {"error": "not_authenticated", "redirect": ...}` quando o
   cliente pede JSON ou envia `X-Requested-With: XMLHttpRequest`; senão
   `303 See Other` para `FRONTEND_URL/login?redirect=...`.
4. Com `perfis_autorizados`, compara `role_usuario` em minúsculas. Fora da
   lista: `403` com `WARNING` no log registrando perfil, perfis exigidos,
   caminho e método.
5. Injeta `usuario_logado` nos kwargs, mas só se a função declarar o
   parâmetro — e remove esse parâmetro da assinatura que o FastAPI inspeciona
   (ver [arquitetura.md](arquitetura.md#detalhes-que-já-causaram-incidente)).

O bloco que forçava troca de senha no primeiro acesso está **comentado**. A
tela e o endpoint existem, mas nada obriga a passar por eles.

### Sessão

| Aspecto | Valor |
| --- | --- |
| Transporte | Cookie `integracar_session`, assinado (não cifrado) |
| Validade do cookie | 30 dias |
| Inatividade | 90 minutos, renovados por `GET /session/atividade` |
| Conteúdo | `usuario` (sem campo de senha), `_last_activity`, `_expira_em`, `_max_age`, `_csrf_token` |

`criar_sessao` remove `senha` e `senha_usuario` antes de gravar. `GET /session`
filtra os mesmos campos na leitura.

> A sessão é *stateless*: não há tabela de sessões. Trocar a `SECRET_KEY`
> invalida todas as sessões ativas de uma vez.

---

## 4. Convenções de resposta

| Situação | Formato |
| --- | --- |
| Erro de negócio ou validação | `{"erro": "mensagem em português"}` |
| Erro de autenticação | `{"error": "not_authenticated", "redirect": "..."}` |
| Erro de permissão | `{"error": "forbidden", "message": "...", "redirect": "..."}` |
| Rate limit | `{"erro": "Muitas tentativas. Tente novamente em alguns instantes.", "detalhes": "..."}` + header `Retry-After: 60` |

Códigos em uso: `200`, `201` (upload), `400`, `401`, `403`, `404`, `413`
(PDF acima de 300 MB), `422` (validação Pydantic, já reduzida a texto), `429`,
`500`, `502` (API de extração indisponível).

`MENSAGENS_HTTP_TRADUZIDAS` em `main.py` traduz as mensagens padrão do
Starlette: `Not Found` vira `Recurso não encontrado`, `Method Not Allowed` vira
`Método não permitido para este recurso`.

---

## 5. Rate limiting

`util/rate_limit.py`, via `slowapi`, chave por IP (`get_remote_address`).

| Alvo | Limite |
| --- | --- |
| Global padrão | 200 por hora |
| `POST /login` | 5 por minuto |
| `POST /esqueci-senha` | 3 por minuto |
| `POST /redefinir-senha` | 3 por minuto |
| `api_geral` (definido, ainda não aplicado) | 100 por minuto |

> **Limitação real.** `storage_uri="memory://"`: o contador vive no processo.
> Reiniciar zera; mais de um worker uvicorn multiplica o limite efetivo pelo
> número de workers. Para valer em produção, trocar por Redis.

---

## 6. Catálogo de endpoints

### 6.1 Público — 10 endpoints

Sem sessão. `routes/publico/`.

| Método | Caminho | O que faz |
| --- | --- | --- |
| POST | `/login` | Autentica. Corpo JSON `{email, senha, manter_conectado, redirect?}`. Responde `{"usuario": {...}}` e grava a sessão. Isento de CSRF, 5/min |
| POST | `/logout` | Destrói a sessão |
| GET | `/csrf-token` | Devolve `{"csrf_token": "..."}` criando-o na sessão se não existir |
| GET | `/session` | Usuário logado, ou `401`. Filtra campos de senha |
| GET | `/session/atividade` | Renova o marcador de inatividade. `{"renovada": true}` ou `401` |
| POST | `/esqueci-senha` | Dispara e-mail de recuperação em background. 3/min |
| POST | `/redefinir-senha` | Troca a senha a partir do token recebido por e-mail. 3/min |
| POST | `/configurar-senha` | Define a senha no primeiro acesso |
| POST | `/ocr-callback` | **Webhook.** Conclusão de job da API de extração. `X-Service-Token` = `OCR_CALLBACK_TOKEN` |
| POST | `/searchable-callback` | **Webhook.** Conclusão do PDF pesquisável. `X-Service-Token` = `SEARCHABLE_CALLBACK_TOKEN` |

Os dois webhooks estão detalhados em [integracoes.md](integracoes.md).

### 6.2 Bolsista — 46 endpoints

`routes/bolsista/bolsista.py`. Todos com
`@requer_autenticacao(["bolsista", "coordenador"])`, exceto os de atividade,
restritos a `["bolsista"]`.

#### Atividades (6)

| Método | Caminho |
| --- | --- |
| GET | `/bolsista/orientadores` |
| GET | `/bolsista/atividades` |
| GET | `/bolsista/atividades/{cod_atividade}` |
| POST | `/bolsista/atividades` |
| PUT | `/bolsista/atividades/{cod_atividade}` |
| DELETE | `/bolsista/atividades/{cod_atividade}` |

#### Documentos e extração (22)

| Método | Caminho | Observação |
| --- | --- | --- |
| POST | `/bolsista/ocr/upload` | `multipart/form-data`, um ou mais PDFs, `cod_pasta` opcional. Validação tudo-ou-nada. Teto de 300 MB por arquivo. Responde `201 {"documentos": [...]}` |
| GET | `/bolsista/ocr` | Vínculos do usuário logado |
| GET | `/bolsista/ocr/consolidado` | Vínculos com os campos já extraídos. `limit`/`offset` opcionais; sem `limit` traz tudo. Total em `total` e no header `X-Total-Count` |
| GET | `/bolsista/ocr/jobs` | Progresso ao vivo de todos os jobs em uma chamada. `status`, `limit=200`, `offset=0`. Evita uma chamada de `/progresso` por documento |
| POST | `/bolsista/ocr/limpar-terminados` | Remove da lista os já `concluido` ou `erro` |
| GET | `/bolsista/ocr/performance-resumo` | Agregado para o gráfico do dashboard |
| GET | `/bolsista/ocr/export.csv` | Exportação, proxy da API de extração |
| GET | `/bolsista/ocr/export.xlsx` | Exportação, proxy da API de extração |
| GET | `/bolsista/ocr/{id}` | Vínculo + campos extraídos + anotação + feedback |
| DELETE | `/bolsista/ocr/{id}` | Apaga o vínculo e notifica os outros vinculados ao mesmo `documento_id` |
| PUT | `/bolsista/ocr/{id}/campo` | Edita um campo. Grava feedback de origem `correcao` |
| PUT | `/bolsista/ocr/{id}/anotacao` | Texto livre do processo. Upsert; texto vazio apaga a linha |
| PUT | `/bolsista/ocr/{id}/feedback-campos` | Avaliação de um campo, origem `voto` |
| DELETE | `/bolsista/ocr/{id}/feedback-campos/{campo}` | Desfaz a avaliação (grava `correto = NULL`) |
| GET | `/bolsista/ocr/{id}/pdf` | Proxy do PDF. Serve a versão pesquisável quando pronta; senão a original. Repassa `Range` |
| GET | `/bolsista/ocr/{id}/pdf-info` | Número de páginas e dimensões |
| GET | `/bolsista/ocr/{id}/paginas/{n}/imagem` | Imagem renderizada da página `n` |
| GET | `/bolsista/ocr/{id}/progresso` | Estado do job e posição na fila |
| POST | `/bolsista/ocr/{id}/reprocessar` | Reenfileira. Incrementa `vezes_reprocessado` |
| GET | `/bolsista/ocr/{id}/versoes` | Histórico de versões dos campos na API de extração |
| GET | `/bolsista/ocr/{id}/performance` | Tempos de processamento do documento |
| GET | `/bolsista/ocr/{id}/vinculos` | Quem mais está vinculado ao mesmo `documento_id` |

> **Ordem das rotas importa.** `/bolsista/ocr/jobs`,
> `/bolsista/ocr/consolidado`, `/bolsista/ocr/limpar-terminados`,
> `/bolsista/ocr/performance-resumo`, `/bolsista/ocr/export.csv` e
> `/bolsista/ocr/export.xlsx` são declaradas **antes** de
> `/bolsista/ocr/{cod_ocr_documento}`. Invertida a ordem, o parâmetro de
> caminho captura o literal e o endpoint vira `404` ou `422`. Há comentários
> marcando isso no arquivo; ao acrescentar rota literal nova, mantenha a
> posição.

#### Notificações de desvínculo (3)

| Método | Caminho |
| --- | --- |
| GET | `/bolsista/notificacoes` |
| GET | `/bolsista/notificacoes/contagem` |
| POST | `/bolsista/notificacoes/{cod_solicitacao}/responder` |

#### Pastas (15)

| Método | Caminho | Observação |
| --- | --- | --- |
| POST | `/bolsista/pastas` | Cria categoria ou processo. Tipo fixado aqui |
| GET | `/bolsista/pastas` | Filhas diretas; `cod_pasta_pai` ausente = raiz |
| GET | `/bolsista/pastas/todas` | Lista achatada por `tipo`, para seletores |
| GET | `/bolsista/pastas/{cod_pasta}` | Dados da pasta |
| PUT | `/bolsista/pastas/{cod_pasta}` | Renomeia |
| PUT | `/bolsista/pastas/{cod_pasta}/mover` | Muda de pasta-mãe. Recusa ciclo |
| DELETE | `/bolsista/pastas/{cod_pasta}` | Apaga. `Pasta` cascateia nas filhas; documento vira `cod_pasta = NULL` |
| PUT | `/bolsista/pastas/documentos/{cod_ocr_documento}/mover` | Move um PDF entre processos |
| POST | `/bolsista/pastas/{cod_pasta}/upload` | Upload direto para um processo |
| GET | `/bolsista/pastas/{cod_pasta}/documentos` | PDFs do processo |
| GET | `/bolsista/pastas/{cod_pasta}/campos-combinados` | Visão "ver junto". `services/campos_merge.py` |
| PUT | `/bolsista/pastas/{cod_pasta}/campos-combinados` | Edita um campo na visão combinada |
| PUT | `/bolsista/pastas/{cod_pasta}/feedback-campos` | Avaliação na visão combinada |
| DELETE | `/bolsista/pastas/{cod_pasta}/feedback-campos/{campo}` | Desfaz a avaliação |
| GET | `/bolsista/pastas/{cod_pasta}/download.zip` | ZIP com os PDFs do processo, nomes desambiguados |

### 6.3 Coordenador — 15 endpoints

`routes/coordenador/coordenador.py`, todos `@requer_autenticacao(["coordenador"])`.
O coordenador também acessa os 46 endpoints `/bolsista/*`.

| Método | Caminho | O que faz |
| --- | --- | --- |
| GET | `/coordenador/orientadores` | Orientadores disponíveis. Funciona para coordenador sem campus |
| GET | `/coordenador/atividades` | Todas as atividades |
| GET | `/coordenador/atividades/{cod_atividade}` | Detalhe |
| POST | `/coordenador/atividades` | Cria |
| PUT | `/coordenador/atividades/{cod_atividade}` | Atualiza |
| DELETE | `/coordenador/atividades/{cod_atividade}` | Exclui |
| GET | `/coordenador/visao-geral` | Números do projeto |
| GET | `/coordenador/monitoramento` | Fila de extração agora e histórico. `dias=7`, `granularidade="dia"`. Separa gestão de API pública: é saúde da workstation, não dado do gestão |
| GET | `/coordenador/graficos` | Séries por campus, orientador e bolsista |
| GET | `/coordenador/bolsistas` | Lista |
| GET | `/coordenador/bolsistas/{cod_usuario}` | Detalhe |
| POST | `/coordenador/bolsistas` | Cadastra |
| PUT | `/coordenador/bolsistas/{cod_usuario}` | Atualiza |
| DELETE | `/coordenador/bolsistas/{cod_usuario}` | Exclui |
| GET | `/coordenador/campus` | Campus cadastrados |

### 6.4 Orientador — 6 endpoints

`@requer_autenticacao(["orientador"])`. Atividades no escopo da equipe.

`GET /orientador/orientadores`, `GET|POST /orientador/atividades`,
`GET|PUT|DELETE /orientador/atividades/{cod_atividade}`.

### 6.5 Consultor — 6 endpoints

`@requer_autenticacao(["consultor"])`. Mesma forma, escopo de campus.

`GET /consultor/orientadores`, `GET|POST /consultor/atividades`,
`GET|PUT|DELETE /consultor/atividades/{cod_atividade}`.

---

## 7. Serviços

`services/` guarda a regra de negócio e não conhece HTTP.

| Módulo | Responsabilidade |
| --- | --- |
| `ocr_client.py` | Cliente HTTP da API de extração: 28 funções, uma por operação do contrato. Upload acima de 40 MB vai em pedaços de 20 MB (`iniciar`/`chunk`/`finalizar`), porque o proxy à frente da API trava o corpo de cada requisição em 64 MB |
| `searchable_client.py` | Cliente do serviço de PDF pesquisável. Toda função é best effort; quem chama trata `SearchableApiError` como "não deu, segue sem" |
| `searchable_service.py` | Decide enfileirar ou reaproveitar o pesquisável de um "irmão" (outro vínculo para o mesmo `documento_id`). Nenhuma função desta camada propaga erro |
| `conclusao_ocr_service.py` | Traduz status do job (`done`/`error`) para status local (`concluido`/`erro`) e roda a reconciliação periódica. Na primeira indisponibilidade da API, interrompe o ciclo (`break`) e adia para o próximo |
| `campos_merge.py` | Combina os `campos` de vários PDFs do mesmo processo. Seção divergente vira lista de versões, uma por PDF. `capa.is_car` é booleano derivado e não conta como divergência |
| `pasta_service.py` | Árvore de pastas: validação de tipo, `Processos Avulsos`, movimentação sem ciclo, nome livre |
| `feedback_campo.py` | Normaliza o nome do campo para a métrica: `ccir_1_codigo` vira `ccir_*_codigo`, `capa.1.interessado` vira `capa.interessado` |
| `atividade_service.py` | Serialização, validação de escopo e de orientador |
| `solicitacao_desvinculo_service.py` | Cria e responde solicitações de desvínculo. Best effort: nunca derruba a exclusão que a chamou |
| `email_service.py` | E-mail transacional. Falha de SMTP é registrada e não propagada |

### Detalhe de `ocr_client`: limiares de upload

```python
LIMIAR_CHUNKING_BYTES = 40 * 1024 * 1024   # 40 MB
TAMANHO_CHUNK_BYTES   = 20 * 1024 * 1024   # 20 MB
UPLOAD_TIMEOUT = httpx.Timeout(120.0, connect=10.0)
READ_TIMEOUT   = httpx.Timeout(15.0,  connect=5.0)
```

O teto de 300 MB (`TAMANHO_MAXIMO_PDF`, em
`controllers/bolsista/ocr_controller.py`) é limite de bom senso para o arquivo
inteiro, não para cada requisição.

---

## 8. Dados

| Camada | Conteúdo |
| --- | --- |
| `data/model/` | `dataclasses` do domínio: `Usuario`, `Pasta`, `OcrDocumento`, `Atividade`, `Campus`, `FeedbackCampo`, `FeedbackCampoPasta`, `AnotacaoProcesso`, `SolicitacaoDesvinculo` |
| `data/repo/` | Uma função por operação. Cada repo expõe `criar_tabela()`, chamada por `initialize_database.py` |
| `data/sql/` | SQL literal em constantes, com `%s`. Inclui `CRIAR_TABELA` e os `ALTER` de migração |
| `data/schemas/` | Pydantic de entrada (`EsqueciSenhaRequest`, `CriarBolsistaRequest`, ...) e validadores compartilhados em `validators.py` |

Esquema completo em [banco-de-dados.md](banco-de-dados.md).

---

## 9. Scripts

### `scripts/` — migrações pontuais

Treze scripts, cada um rodado **uma vez** por banco, idempotentes por
construção (conferem antes de alterar). Fazem o que o `CREATE TABLE IF NOT
EXISTS` não faz: alterar tabela já existente.

```
migrar_pasta.py                        cria Pasta, adiciona Ocr_Documento.cod_pasta
migrar_processos_avulsos.py            adiciona Pasta.padrao
migrar_avulsos_para_categoria.py       converte "Avulsos" de processo em categoria
migrar_searchable.py                   colunas searchable_id / searchable_status
backfill_searchable.py                 enfileira documentos antigos no pesquisável
migrar_vezes_reprocessado.py           coluna vezes_reprocessado
migrar_feedback_campo.py               cria Feedback_Campo
migrar_feedback_campo_append_only.py   remove o UNIQUE, vira histórico
migrar_feedback_campo_origem.py        coluna origem ('voto' | 'correcao')
migrar_feedback_campo_pasta.py         cria Feedback_Campo_Pasta
migrar_anotacao_processo.py            cria Anotacao_Processo
migrar_solicitacao_desvinculo.py       cria Solicitacao_Desvinculo
corrigir_fuso_horario.py               ajusta datas gravadas em UTC
```

### `seeders/` — carga inicial

`importar_campus.py` e `importar_usuario.py` leem `processos.csv`,
`bolsistas.csv` e `orientadores.csv`. Bolsistas importados recebem senha
temporária em **texto puro** (decisão de negócio registrada em
`docs/security/relatorio-seguranca.md`).

### `check-deploy.py` — validação pré-deploy

Nove verificações, saída `0` quando todas passam: existência do `.env`,
variáveis obrigatórias, coerência do ambiente (`FRONTEND_URL` sem
`localhost` e com HTTPS em produção), versão do Python (3.10+), dependências
principais importáveis, `SECRET_KEY` gerada e com 32+ caracteres, `.env`
ignorado pelo git, conectividade com o MySQL, e existência de `tests/`.

### `initialize_database.py` — recriação do schema

**Destrutivo.** Derruba as tabelas, recria pela ordem de dependência e roda os
seeders. Em produção, nunca; use o script da migração específica.

Dois cuidados:

- O `DROP` usa `Atividades` (plural) e a tabela é `Atividade` (singular): o
  `DROP` é um no-op e as atividades antigas sobrevivem apontando para ids de
  usuário recriados.
- Há um `except` que tenta a sintaxe de PostgreSQL como alternativa, resquício
  de uma migração de banco considerada e não concluída (ver o branch
  `origin/db/postgresql`).
