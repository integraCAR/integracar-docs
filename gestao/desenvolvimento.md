# Desenvolvimento — integracar-gestao

Como subir o ambiente local, onde mexer, o que verificar antes de abrir PR, e o
que fazer quando algo não sobe.

Pré-requisito: [README.md](README.md).

---

## 1. Pré-requisitos

| Ferramenta | Versão | Para que |
| --- | --- | --- |
| Python | 3.12+ | Backend. `check-deploy.py` aceita 3.10+, mas a imagem de produção é 3.12 |
| Node | 20+ | Frontend. A imagem de produção é `node:20-alpine` |
| Docker e Docker Compose | 20.10+ / 2.0+ | MySQL local |
| Git | — | |

Para desenvolver o backend inteiro você também precisa de **acesso à API de
extração** (`OCR_API_BASE` e os tokens). Sem ela o sistema sobe, mas upload,
listagem de documentos e visualização de PDF falham com `502`. Combine o acesso
com a equipe da workstation antes de começar.

---

## 2. Setup, passo a passo

```bash
git clone https://github.com/integraCAR/integracar-gestao.git
cd integracar-gestao
```

### 2.1 Backend

```bash
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2.2 Configuração

```bash
cp .env.example .env
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

Edite o `.env`. O mínimo para o processo subir:

```env
ENVIRONMENT=development
FRONTEND_URL=http://localhost:5173
SECRET_KEY=<a chave gerada acima>

DB_USER=root
DB_PASSWORD=<sua senha local>
DB_HOST=localhost
DB_PORT=3306
DB_NAME=integracar_local

MAIL_USERNAME=...
MAIL_PASSWORD=...
MAIL_FROM=...
MAIL_SERVER=smtp-relay.brevo.com
MAIL_PORT=587

OCR_API_BASE=https://...
OCR_SERVICE_TOKEN=...
OCR_CALLBACK_TOKEN=...
OCR_PUBLIC_BASE=http://localhost:8000
```

As quatro variáveis `OCR_*` são **obrigatórias**: `util/ocr_config.py` falha no
boot se faltar qualquer uma. As três `SEARCHABLE_*` são opcionais e podem ficar
comentadas — o PDF pesquisável simplesmente fica desligado.

> `FRONTEND_URL` precisa ser `http://localhost:5173` em desenvolvimento: é a
> porta do `react-router dev`, e é para lá que o backend redireciona quem não
> está autenticado. O padrão do código já é essa porta; o `.env.example` traz
> `:3000`, que é a porta do build servido, não do dev.

### 2.3 MySQL

```bash
docker compose up -d mysql
docker compose ps                 # confirme "healthy"
```

O compose publica a porta em `127.0.0.1:3306` — só localhost.

### 2.4 Schema e dados iniciais

```bash
python initialize_database.py
```

> **Este script apaga tudo.** Derruba as tabelas, recria e roda os seeders. Em
> banco com dado que importe, use o script de migração específico em
> `scripts/`. Ver [banco-de-dados.md](banco-de-dados.md).

### 2.5 Subir

```bash
uvicorn main:app --reload                    # terminal 1, porta 8000
```

```bash
cd frontend && npm install && npm run dev    # terminal 2, porta 5173
```

| Endereço | O que é |
| --- | --- |
| <http://localhost:5173> | Interface |
| <http://localhost:8000/docs> | Swagger UI |
| <http://localhost:8000/redoc> | ReDoc |

---

## 3. Fluxo de trabalho

```bash
git checkout -b feat/descricao-curta      # ou fix/, docs/, refactor/
# ... alterações + testes ...
pytest                                     # backend
cd frontend && npm run typecheck           # frontend
git commit -am "feat: descrição no imperativo, em português"
git push origin feat/descricao-curta
# abrir PR
```

### Antes de abrir PR

| Verificação | Comando |
| --- | --- |
| Suíte passando | `pytest` |
| Tipos do frontend | `cd frontend && npm run typecheck` |
| Build do frontend | `cd frontend && npm run build` |
| Nada de segredo no diff | `git diff --cached` |
| Documentação atualizada | Mexeu em endpoint, tabela ou integração? Atualize em `integracar-docs` |

Não há CI configurada neste repositório. As verificações acima são manuais, e é
por isso que elas precisam entrar no hábito.

### Convenções de commit

Português, no imperativo, com prefixo de tipo:

```
feat: adiciona dropdown para seleção do orientador
fix: valida orientador e escopo global do coordenador
docs: atualiza contrato do callback de PDF pesquisável
refactor: extrai regra de pasta para pasta_service
test: cobre reconciliação com API indisponível
```

Parte do histórico tem um emoji antes do tipo. A convenção atual do workspace
é **sem emoji**, em mensagem de commit e em qualquer texto.

### Revisão de código

Convenção do workspace, válida para todos os repositórios:

- **Comece por uma frase dizendo qual é o problema.** A notificação por e-mail
  mostra só as primeiras linhas.
- **Cite `arquivo:linha`**, nunca URL completa do GitHub no meio do texto —
  vira uma tira de link que quebra a leitura no e-mail.
- Depois do resumo, explique com rigor **como reproduzir** e **o que acontece
  de errado**. Não encurte às custas do argumento.
- Tudo em português do Brasil. Revisão em inglês obriga a equipe a traduzir
  antes de agir.

---

## 4. Onde mexer

### Acrescentar um endpoint

1. **Rota** em `routes/<perfil>/<perfil>.py`: decorador de rota, depois
   `@requer_autenticacao([...])`, e a função delegando ao controller.
   - Rota com caminho literal vai **antes** da rota com parâmetro no mesmo
     nível (`/bolsista/ocr/jobs` antes de `/bolsista/ocr/{id}`), senão o
     parâmetro captura o literal.
2. **Controller** em `controllers/<perfil>/`: valida, chama o service, monta o
   `JSONResponse`, registra log com `**get_log_extra(request)`.
3. **Service** em `services/`, se houver regra de negócio nova.
4. **Repo e SQL** em `data/repo/` e `data/sql/`, se tocar o banco.
5. **Schema** em `data/schemas/`, se receber corpo.
6. **Teste** em `tests/test_routes/`, cobrindo o caminho feliz, o `401`/`403` e
   pelo menos um caminho de erro.
7. **Frontend**: função em `app/services/*.server.ts`, e o `loader`/`action`
   que a usa.
8. **Documentação**: a tabela de endpoints em [backend.md](backend.md).

### Acrescentar uma coluna

Ver [banco-de-dados.md, seção 5](banco-de-dados.md#5-migrações). Resumo: ajuste
o `CRIAR_TABELA`, acrescente a constante `ALTERAR_...` ao lado, escreva
`scripts/migrar_<assunto>.py` seguindo o padrão de `migrar_searchable.py`, e
atualize a tabela de histórico.

### Acrescentar uma tela

1. Arquivo em `app/routes/` e entrada em `app/routes.ts` (as rotas são
   **declaradas**, não inferidas do nome).
   - Prefixo `_app.` para ficar sob o layout autenticado.
2. `loader` buscando por `app/services/*.server.ts`; `action` se escrever.
3. Template em `app/ui/templates/`, recebendo dados prontos por props — sem
   buscar nada.
4. Entrada no `Sidebar`, se for navegável, com o corte de perfil correto
   (`usePermission` ou `podeProcessarDocumentos`).
5. `npm run typecheck`.

---

## 5. Padrões de código

### Python

- Nomes de domínio em **português** (`criar_pasta`, `cod_usuario`,
  `listar_documentos`). Nomes técnicos em inglês quando é o termo consagrado
  (`request`, `response`, `timeout`).
- Comentários e docstrings em português, e **explicando o porquê**, não o quê.
  O padrão do repositório é registrar o incidente que motivou a linha — veja
  `conexao_db.py:DB_TIME_ZONE` e o `csrf_middleware` de `main.py`. É isso que
  evita que a próxima pessoa "simplifique" a linha e reintroduza o bug.
- Type hints nas assinaturas públicas.
- Log sempre com `extra={**get_log_extra(request), ...}`.
- Exceção de integração vira exceção de domínio (`OcrApiError`,
  `SearchableApiError`), traduzida para HTTP só no controller.

### TypeScript e React

- Componentes funcionais, props tipadas.
- Nada de `fetch` direto para a API fora de `*.server.ts`.
- `cn()` de `~/lib/utils` para compor classes do Tailwind.
- Atomic design: `atoms` sem estado, `templates` sem busca de dados.
- O atalho `~/` aponta para `app/` (via `vite-tsconfig-paths`).

---

## 6. Depuração

### Backend

```bash
# log em tempo real
tail -f logs/app.log

# só os erros
grep -E "ERROR|WARNING" logs/app.log

# seguir uma requisição inteira pelo request_id
grep "req_id=3f2b8c1a" logs/app.log
```

O `request_id` é a ferramenta principal: todo log de uma mesma requisição
carrega o mesmo, inclusive os do service e do repo.

Breakpoint: `breakpoint()` na linha, e rode com `uvicorn main:app` **sem**
`--reload` (o reloader derruba a sessão de depuração).

### Frontend

- `console.log` em `loader` e `action` aparece **no terminal do `npm run dev`**,
  não no navegador — eles rodam no servidor.
- `console.log` em componente aparece no navegador.
- Erro de API é registrado por `api.server.ts` como
  `[API Error] Status: <n> na URL: <url>`, e falha de conexão como
  `[Fetch Failed] Falha ao conectar em: <url>`. Os dois no terminal.

---

## 7. Solução de problemas

| Sintoma | Causa provável e saída |
| --- | --- |
| `ValueError: SECRET_KEY não configurada no arquivo .env` | `.env` ausente ou sem `SECRET_KEY`. Gere e preencha |
| `ValueError: Variáveis de ambiente obrigatórias não configuradas` | Falta variável de banco ou `OCR_*`. A mensagem diz quais |
| `ModuleNotFoundError: No module named 'mysql'` | `venv` não ativado, ou `pip install -r requirements.txt` não rodado |
| `Erro ao criar connection pool` | MySQL não está de pé (`docker compose ps`), ou `DB_HOST`/`DB_PORT` errados. Dentro de container, `DB_HOST` é `mysql`, não `localhost` |
| Porta 8000 ocupada | `lsof -i :8000` e encerre o processo, ou `uvicorn main:app --reload --port 8001` |
| `Address already in use` no container | `docker compose down` e suba de novo |
| Erro de CORS no navegador | `FRONTEND_URL` diferente da origem real. Em dev, as portas comuns de localhost já estão liberadas |
| `403 CSRF token ausente` em ação do frontend | O `action` não passou por `apiFetch`, ou chamou a API direto. Toda mutação precisa ir por `app/services/*.server.ts` |
| `401` em toda chamada do SSR | `API_URL` apontando para a URL pública em vez do endereço interno: o servidor sai para a internet e perde o cookie |
| `502` em upload, listagem e PDF | API de extração inalcançável ou token rejeitado. Ver [integracoes.md](integracoes.md#6-diagnóstico) |
| Documento preso em `na_fila` | Callback não chegou. `OCR_PUBLIC_BASE` precisa ser alcançável de fora; em dev, use um túnel |
| E-mail não sai | SMTP não configurado. O erro fica no log e não aparece para o usuário; confira `logs/app.log` |
| Lupa não acha nada no PDF | PDF pesquisável desligado ou ainda em `pending`. Confira `searchable_status` do documento |
| Tela branca ao salvar formulário | Era o sintoma clássico de `{"detail": [...]}` em erro de validação. Se voltar, confira o handler de `RequestValidationError` em `main.py` |
| `npm run typecheck` reclamando de rota que existe | Rode `npx react-router typegen` antes; o script já faz isso, rodar `tsc` sozinho não |
| `/configurar-senha` não carrega após redirecionamento | `routeDiscovery: { mode: "initial" }` em `react-router.config.ts` |

---

## 8. Documentação do repositório que não é confiável

| Arquivo | Situação |
| --- | --- |
| `frontend/README.md` | Template padrão do React Router, em inglês, sem nada deste projeto |
| `tests/README.md` | Descreve arquivos de teste que não existem mais (`test_processo_repo.py`) e contagens antigas ("78+ testes") |
| `README.md`, `DEVELOPMENT.md`, `DEPLOYMENT.md` | Foram atualizados para apontar para `integracar-docs`. Versões anteriores descreviam frontend Next.js, `NEXT_PUBLIC_API_URL`, `/api/` como prefixo de produção, 161 testes e 94% de cobertura — tudo desatualizado |

A fonte atual é este repositório de documentação. Ao corrigir algo no código que
contradiga um documento, corrija o documento no mesmo PR.
