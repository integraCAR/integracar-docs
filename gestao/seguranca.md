# Segurança — integracar-gestao

Como o sistema autentica, autoriza e se protege; o que já foi corrigido, o que
foi decidido não corrigir, e o que continua pendente.

Pré-requisitos: [README.md](README.md) e [backend.md](backend.md).

A auditoria de código que originou boa parte deste documento está em
`integracar-gestao/docs/security/` (`plano-analise.md` e
`relatorio-seguranca.md`), datada de 2026-08-28 e atualizada até 2026-09-02.
Este arquivo resume e mantém o que vale hoje; o relatório original guarda o
detalhe item por item.

---

## 1. Autenticação

### Senha

| Aspecto | Como é |
| --- | --- |
| Algoritmo | bcrypt, `bcrypt.gensalt()` com custo padrão (12 rounds) |
| Senha acima de 72 bytes | Reduzida por SHA-256 hexdigest (64 bytes ASCII) antes do bcrypt — o bcrypt trunca silenciosamente acima de 72 bytes |
| Verificação | `bcrypt.checkpw` quando o hash começa com `$2a$`/`$2b$`; senão comparação de texto puro com `hmac.compare_digest` |
| Força mínima | 8 caracteres, uma maiúscula, uma minúscula, um número, e não estar em uma lista de 14 senhas comuns |
| Caractere especial | Previsto no código, **comentado** — não é exigido hoje |

O caminho de texto puro existe para **contas legadas e importadas**. A
comparação usa `compare_digest` justamente para não vazar informação por tempo
de execução.

> **Decisão de negócio registrada.** Bolsistas importados por planilha recebem
> senha temporária (últimos 6 dígitos do telefone) gravada **em texto puro**. A
> aplicação de hash chegou a ser implementada e foi **revertida a pedido da
> equipe**: é preciso consultar a senha temporária para comunicá-la ao usuário
> no lançamento, e hash é irreversível. A alternativa oferecida — exportar as
> senhas em planilha separada e manter só hash no banco — foi recusada.
> Reavaliar depois do lançamento, quando os usuários já tiverem trocado a senha
> temporária pela própria.

### Recuperação de senha

`token_reset_senha` gerado com `secrets.token_urlsafe(32)`,
`token_reset_expiracao` de 24 horas por padrão. O e-mail usa o template
`services/templates/reset_password.html`. Falha de SMTP é registrada no log e
**não propagada** ao usuário.

### Primeiro acesso

A tela `/configurar-senha` e o endpoint `POST /configurar-senha` existem e
funcionam, com validação mais rigorosa (`validar_senha_atualizada`). Mas o
bloqueio que forçava passar por ela está **comentado** em
`util/auth_decorator.py` — revertido em 2026-09-02. Hoje nada obriga a troca.

---

## 2. Sessão

| Aspecto | Valor | Observação |
| --- | --- | --- |
| Transporte | Cookie `integracar_session` | `SessionMiddleware` do Starlette |
| Proteção | **Assinado**, não cifrado | `itsdangerous` com a `SECRET_KEY`. O conteúdo é legível por quem tiver o cookie; não é falsificável |
| `max_age` | 2 592 000 s (30 dias) | |
| `same_site` | `lax` | |
| `https_only` | `True` em produção | |
| `HttpOnly` | Sim (padrão do middleware) | |
| Inatividade | 90 minutos | `INACTIVITY_TIMEOUT_MINUTES`, em `util/auth_decorator.py` |

Conteúdo da sessão: `usuario` (sem nenhum campo de senha), `_last_activity`,
`_expira_em`, `_max_age`, `_csrf_token`.

`criar_sessao` remove `senha` e `senha_usuario` antes de gravar; `GET /session`
filtra os mesmos campos na leitura. Isso foi consequência de uma correção: o
`POST /login` devolvia `senha_usuario` (hash, ou texto puro em conta legada) no
corpo da resposta, visível na aba Network do navegador.

A renovação de inatividade acontece por `GET /session/atividade`, chamado pelo
frontend **só em interação real do usuário** — não por polling, que renovaria a
sessão de uma aba esquecida aberta.

> A sessão é *stateless*: não há tabela de sessões, e não há como invalidar uma
> sessão específica. Trocar a `SECRET_KEY` invalida **todas** de uma vez.

---

## 3. Autorização

### Dupla camada, uma delas decorativa

```
Frontend   usePermission / podeProcessarDocumentos
             -> decide o que APARECE na tela.  Conveniência.
Backend    @requer_autenticacao([perfis]) + checagem de propriedade
             -> decide o que ACONTECE.  É a que vale.
```

Esconder um botão não protege um endpoint. A regra do projeto é: toda rota
autenticada tem `@requer_autenticacao`, e toda rota que recebe um id confere a
**propriedade** daquele id antes de usá-lo.

### Checagem de propriedade (anti-IDOR)

Os pontos onde isso é feito:

| Função | Garante |
| --- | --- |
| `_obter_registro_do_usuario(cod_ocr_documento, cod_usuario)` | O vínculo existe **e** é do usuário logado |
| `pasta_service.obter_pasta_do_usuario(cod_pasta, cod_usuario)` | A pasta existe e pertence ao usuário — não confia no id sozinho |
| `pasta_service.validar_processo_para_upload` | A pasta existe, é do usuário e é do tipo `processo` |
| `atividade_service.validar_atividade_do_usuario` | A atividade é do próprio usuário |
| `atividade_service.validar_atividade_do_campus` | A atividade está no campus do orientador ou consultor |
| `solicitacao_desvinculo_service.contar_outros_vinculados` | O vínculo é do usuário antes de contar os demais |

Duas falhas de IDOR foram encontradas e corrigidas na auditoria: leitura de
atividade por id sem checagem de propriedade, e orientador ou consultor
acessando, editando e excluindo processo de **qualquer** campus.

`coordenador` segue sem restrição de escopo **por projeto** — a visão sistêmica
é a função do perfil.

---

## 4. CSRF

Implementação própria, em `util/csrf.py` e no `csrf_middleware` de `main.py`.

| Aspecto | Como é |
| --- | --- |
| Geração | `secrets.token_urlsafe(32)`, guardado na sessão em `_csrf_token` |
| Transporte | Header `X-CSRF-Token` (somente header; não lê de corpo nem de cookie próprio) |
| Comparação | `secrets.compare_digest` |
| Métodos protegidos | Todos menos `GET`, `HEAD`, `OPTIONS`, `TRACE` |
| Obtenção | `GET /csrf-token`, que cria o token se ainda não existir |
| Isenções | `/login`, `/ocr-callback`, `/searchable-callback` |

A proteção cobria apenas duas rotas mutantes antes da auditoria; hoje é o
middleware que cobre todas, com as três isenções justificadas:

- `/login` não tem sessão prévia de onde carregar token. Protegido por rate
  limit de 5 por minuto por IP.
- os dois callbacks são servidor-a-servidor, sem cookie e sem de onde tirar
  token CSRF. Autenticados por `X-Service-Token`.

A comparação de caminho usa `get_route_path(request.scope)`, e isso não é
detalhe: `request.url.path` carrega o `root_path` em produção, e
`/gestao/api/ocr-callback` nunca casaria com `/ocr-callback`. A isenção virava
letra morta, os webhooks levavam `403` e o job ficava preso em `pending`.

Quem envia o token é o **servidor do frontend**, não o navegador — ver
[frontend.md, seção 3](frontend.md#csrf-no-servidor), incluindo a necessidade
de substituir o cookie pelo `Set-Cookie` da resposta de `/csrf-token`.

Há ainda a proteção nativa do React Router 7.12+, que compara o `Host` recebido
pelo Node com o `Origin` do navegador; `allowedActionOrigins` libera o domínio
com e sem `www`.

---

## 5. Headers de resposta

Sempre:

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
```

Somente em produção:

```
Content-Security-Policy: default-src 'self';
                         script-src 'self' https://cdn.jsdelivr.net;
                         style-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net;
                         img-src 'self' data: https://fastapi.tiangolo.com;
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

`'unsafe-inline'` foi removido de `script-src` e **permanece em `style-src`** —
pendência conhecida, a remover depois de validar o build de produção.
`cdn.jsdelivr.net` e `fastapi.tiangolo.com` estão liberados porque o
`/docs` do FastAPI carrega o Swagger UI de lá.

---

## 6. Rate limiting

`util/rate_limit.py`, `slowapi`, chave por IP.

| Alvo | Limite |
| --- | --- |
| Global padrão | 200 por hora |
| `POST /login` | 5 por minuto |
| `POST /esqueci-senha` | 3 por minuto |
| `POST /redefinir-senha` | 3 por minuto |
| `api_geral` (definido, não aplicado) | 100 por minuto |

Resposta: `429` com `Retry-After: 60` e mensagem em português.

> **Duas limitações reais.** (1) `storage_uri="memory://"`: o contador vive no
> processo, zera a cada reinício, e com mais de um worker uvicorn o limite
> efetivo é multiplicado pelo número de workers. Para valer em produção,
> precisa de Redis. (2) O limite é **só por IP**, sem bloqueio por conta —
> aceitável no porte atual, a reavaliar se houver indício de credential
> stuffing.

---

## 7. Injeção e validação de entrada

Auditado e sem achado além do IDOR já citado:

- **SQL**: 100% parametrizado com `%s`. O SQL fica em `data/sql/` como
  constante; nenhuma consulta é montada por concatenação com dado de entrada.
- **Upload**: o PDF nunca é escrito em disco com nome fornecido pelo usuário.
  Validação de tipo (`application/pdf` ou extensão `.pdf`), de conteúdo não
  vazio e de tamanho (300 MB) antes de qualquer coisa.
- **SSRF**: `OCR_API_BASE` e `SEARCHABLE_API_BASE` são fixos por ambiente;
  nenhum destino de requisição vem de entrada do usuário.
- **E-mail**: sem concatenação manual de header; `fastapi-mail` com template.
- **Nome de arquivo em ZIP**: `_nome_arquivo_zip_seguro` e `_nome_unico_no_zip`
  sanitizam e desambiguam antes de montar o `download.zip`.
- **Validação de corpo**: Pydantic em `data/schemas/`, com validadores
  compartilhados em `validators.py`. Erro de validação é reduzido pelo handler
  global a `{"erro": "primeira mensagem legível"}`.

---

## 8. Log e dados pessoais

`util/logging.py`: `RotatingFileHandler`, 10 MB por arquivo, 5 backups (≈50 MB
no total), em `logs/` — fora do versionamento, sem rota HTTP servindo o
diretório.

O que **nunca** vai para o log: senha, hash de senha. Token é sempre truncado
(`token[:8] + "..."`).

O que vai: `request_id`, `user_id`, `role`, `ip`, `user_agent`, caminho, método,
e o e-mail nas tentativas de login (deliberado, para investigar acesso).

Decisões de acesso registradas em `WARNING`, servindo de trilha de auditoria:
usuário não autenticado, permissão insuficiente, sessão expirada por
inatividade, rate limit excedido, token de callback inválido.

> **Observação de LGPD, não corrigida.** O item 9 da auditoria removeu `str(e)`
> das respostas ao cliente, mas o manteve no log interno. A mensagem de um erro
> do MySQL (por exemplo `IntegrityError` de e-mail ou CPF duplicado) pode
> incluir o valor duplicado no texto, indo para o log em texto puro. Não é
> exposição por requisição — exige acesso ao servidor. Reavaliar se surgir
> exigência de retenção ou mascaramento.

Não há política de expurgo por tempo, só rotação por tamanho.

---

## 9. Segredos

| Segredo | Onde fica | Observação |
| --- | --- | --- |
| `SECRET_KEY` | `.env` | Gerar com `python3 -c "import secrets; print(secrets.token_urlsafe(32))"`. **Nunca reaproveitar entre ambientes** |
| Credenciais do MySQL | `.env` | |
| Credenciais SMTP | `.env` | |
| Os quatro tokens de serviço | `.env` | Combinados com a equipe da workstation. Ver [integracoes.md](integracoes.md) |

`.env` está no `.gitignore`, e `check-deploy.py` **verifica isso** com
`git check-ignore -v .env`, falhando se não estiver.

> **Pendência.** No `docker-compose.yml`, `MYSQL_ROOT_PASSWORD` usa a **mesma**
> `DB_PASSWORD` do usuário da aplicação. Em produção, separar.

---

## 10. Dependências

Situação em 2026-10-06, a partir do relatório de auditoria.

### Aplicado

Patches de baixo risco no backend (quatro bumps) e no frontend, com a suíte
rodada depois de cada rodada. `overrides` em `frontend/package.json` força
`qs: ^6.16.0` em dependência transitiva.

### Pendente, por ordem de prioridade

1. **`pdfjs-dist` — prioridade alta.** RCE conhecido ao abrir PDF malicioso, e
   este sistema renderiza PDF de upload de bolsista e de digitalização de
   terceiros. A correção é *major version* (`6.3.289`); a versão em uso é
   `^5.7.284`. Exige validação visual dedicada do `PdfViewer` antes de aplicar,
   dado o histórico de problemas nessa integração.
2. *Major versions* do backend avaliadas e não aplicadas: `fastapi`/`starlette`
   (0.115 para 1.x), `mysql-connector-python` (8 para 9),
   `aiosmtplib`/`fastapi-mail` (2 para 5), `pytest` (8 para 9). Cada uma precisa
   de teste de regressão dedicado.
3. Rodar `pip-audit` (ou `safety`) e `npm audit` periodicamente. Não há isso
   automatizado em CI hoje.

---

## 11. Quadro de pendências

| # | Pendência | Prioridade |
| --- | --- | --- |
| 1 | Atualizar `pdfjs-dist` (RCE ao abrir PDF malicioso) | Alta |
| 2 | Trocar o rate limiting para Redis | Média |
| 3 | Remover `'unsafe-inline'` de `style-src` na CSP | Média |
| 4 | Separar `MYSQL_ROOT_PASSWORD` de `DB_PASSWORD` | Média |
| 5 | Reavaliar os *major upgrades* do backend com teste de regressão | Média |
| 6 | Reativar a exigência de troca de senha no primeiro acesso | Média |
| 7 | `UNIQUE` em `Usuario.email_usuario`; `ENUM` ou `CHECK` em `role_usuario` | Média |
| 8 | Migrar as senhas legadas em texto puro para bcrypt | Decidido não fazer agora; reavaliar após o lançamento |
| 9 | Mascarar CPF e e-mail em log de erro de banco | Baixa |
| 10 | Autenticar ou desativar `/docs` e `/redoc` em produção | Baixa |
| 11 | Bloqueio por conta no login, além do bloqueio por IP | Baixa |

---

## 12. Como relatar uma vulnerabilidade

Não há endereço público de divulgação. Até existir, relate à coordenação do
projeto por canal interno, com:

- descrição do problema e impacto;
- passos de reprodução;
- `arquivo:linha` do código envolvido, quando aplicável.

Não abra issue pública com detalhe explorável.
