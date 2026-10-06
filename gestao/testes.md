# Testes — integracar-gestao

Como a suíte é organizada, como rodar, o que ela cobre e o que não cobre.

Pré-requisito: [README.md](README.md).

Todos os números abaixo foram medidos em 2026-10-06, no `main`, e cada um traz
o comando que o reproduz.

---

## 1. Resumo

```
$ pytest
...
485 testes coletados: 480 passando, 5 pulados
Cobertura total (linha e ramo): 71,52%
Tempo: cerca de 6 segundos
```

| Diretório | Testes | O que cobre |
| --- | --- | --- |
| `tests/test_routes/` | 184 | Endpoints HTTP, de ponta a ponta com `TestClient` |
| `tests/test_services/` | 128 | Regra de negócio |
| `tests/test_repo/` | 123 | Camada de acesso a dados, com banco mockado |
| `tests/test_util/` | 40 | Segurança, decorador de autenticação, fuso |
| `tests/test_controllers/` | 7 | Helpers compartilhados entre controllers |
| `tests/test_conexao_db.py` | 3 | Pool de conexão |

Reproduzir a contagem por diretório:

```bash
pytest --collect-only -q --no-cov tests/test_routes | tail -1
```

> **A suíte não precisa de MySQL, nem de Docker, nem da API de extração.** Todo
> acesso a banco é mockado e todo cliente HTTP é mockado. É por isso que ela
> roda em segundos — e é por isso que ela não detecta erro de SQL de verdade.

---

## 2. Rodando

```bash
pytest                              # tudo, já com cobertura
pytest --no-cov                     # mais rápido, sem relatório
pytest tests/test_routes -v         # um diretório
pytest tests/test_routes/test_bolsista_ocr.py       # um arquivo
pytest -k reconciliacao             # por palavra-chave
pytest -x                           # para no primeiro erro
pytest -rs                          # mostra o motivo de cada teste pulado
pytest --lf                         # só os que falharam da última vez
```

As flags de cobertura estão em `pytest.ini` (`addopts`), então `pytest` sozinho
já gera:

| Saída | Onde |
| --- | --- |
| Resumo com linhas não cobertas | Terminal |
| Relatório navegável | `htmlcov/index.html` |
| XML para ferramenta externa | `coverage.xml` |

Configuração em `pytest.ini` e `.coveragerc`: `asyncio_mode = auto` (não é
preciso marcar cada teste assíncrono), `--strict-markers` (marcador não
declarado é erro), `--cov-branch` (cobertura de ramo, não só de linha).

### Marcadores declarados

`unit`, `integration`, `slow`, `auth`, `repo`, `routes`. Estão declarados em
`pytest.ini` e **pouco usados** na suíte atual; a organização efetiva é por
diretório.

---

## 3. Fixtures

Todas em `tests/conftest.py` (236 linhas).

### Clientes HTTP

| Fixture | O que entrega |
| --- | --- |
| `client` | `TestClient(app)` **já com o token CSRF obtido e anexado** como header padrão |
| `authenticated_client` | `client` com a sessão mockada como `bolsista` |
| `authenticated_coordenador_client` | `client` com a sessão mockada como `coordenador` |

A fixture `client` chama `GET /csrf-token` e guarda o token em
`c.headers["X-CSRF-Token"]`. Sem isso, **todo** `POST`, `PUT` e `DELETE` da
suíte levaria `403` do `csrf_middleware`. É o primeiro lugar a olhar quando um
teste novo de escrita falha com 403.

### Usuários de teste

Um por perfil: `mock_usuario_bolsista`, `mock_usuario_coordenador`,
`mock_usuario_consultor`, `mock_usuario_orientador`,
`mock_usuario_gestor_tecnico`, `mock_usuario_gestor_administrativo`.

Os dois últimos existem mesmo sem rotas próprias no backend — são usados para
verificar que o acesso é **negado**.

### Dados e banco

`mock_db_connection` (conexão e cursor `MagicMock`), `mock_processo`,
`mock_analise_processo`, `mock_atividade`.

### Detalhe do `conftest.py` que importa

```python
os.environ['OCR_RECONCILIACAO_INTERVALO_S'] = '0'
from main import app
```

A variável é definida **antes** de importar `main`, desligando o laço de
reconciliação periódica. Sem isso, cada execução da suíte subiria uma tarefa
assíncrona tentando falar com a API de extração.

Pela mesma razão, o `.env` precisa estar presente e preenchido para rodar os
testes: `main` importa `conexao_db` e `util/ocr_config`, que **falham no boot**
se faltar variável obrigatória. A suíte não conecta no banco, mas lê a
configuração.

---

## 4. Testes pulados

Cinco, todos por limitação conhecida:

| Arquivo | Motivo declarado |
| --- | --- |
| `test_routes/test_login.py:13` | "Login requires proper session middleware and cookie handling" |
| `test_routes/test_login.py:157` | idem |
| `test_routes/test_login.py:126` | "Rota /logout não implementada" |
| `test_routes/test_login.py:144` | idem |
| `test_util/test_auth_decorator.py:220` | "Requires SessionMiddleware installed on FastAPI app" |

Para conferir a lista atual: `pytest -rs`.

> **Dois desses motivos estão desatualizados.** `POST /logout` **existe** e
> funciona (`routes/publico/publico_login.py:54`). Os dois testes pulados por
> "rota não implementada" podem ser reescritos.

O caminho de login completo — com cookie de sessão de verdade — é o maior buraco
da suíte, e justamente o mais crítico. Hoje ele é verificado manualmente, pelo
navegador, contra o `docker compose` completo.

---

## 5. Cobertura

Total de **71,52%** (linha e ramo), com `source = .` no `.coveragerc`.

### Como ler esse número

O `source = .` inclui arquivos que a suíte não tem por que executar:

| Arquivo ou pasta | Cobertura | Por que |
| --- | --- | --- |
| `check-deploy.py` | 0% | Script de operação, roda fora da suíte |
| `scripts/*` | baixa | Migrações pontuais, rodadas à mão |
| `util/database.py` | 40% | Caminho alternativo de conexão, pouco usado |
| `util/security.py` | 53% | As funções de geração de senha aleatória e validação rigorosa têm pouca cobertura |

Omitidos pelo `.coveragerc`: `tests/`, `venv/`, `frontend/`, `node_modules/`,
`seeders/`, `conftest.py`, `__init__.py`, `initialize_database.py`.

As camadas de domínio estão bem acima da média:

| Camada | Situação |
| --- | --- |
| `data/model/` | 100% em todos os arquivos |
| `data/repo/` | Alta, com poucas linhas de tratamento de erro descobertas |
| `data/schemas/` | Alta |
| `services/` | Alta nos módulos de regra (`campos_merge`, `pasta_service`, `feedback_campo`, `conclusao_ocr_service`) |
| `controllers/` | Média, concentrada nos caminhos de erro de integração |
| `util/timezone.py`, `util/logging.py` | 100% e 96,6% |

Para ver arquivo por arquivo: `pytest` e depois `htmlcov/index.html`.

Objetivo prático para código novo: **cobrir o caminho feliz, o caminho negado
(`401`/`403`) e pelo menos um caminho de erro de integração.** Perseguir um
número global alto com `source = .` dá pouca informação.

---

## 6. Padrões da suíte

### Teste de rota

```python
def test_listar_documentos_nega_perfil_errado(client, mocker, mock_usuario_orientador):
    mocker.patch("util.auth_decorator.obter_usuario_logado",
                 return_value=mock_usuario_orientador)
    resposta = client.get("/bolsista/ocr")
    assert resposta.status_code == 403
```

A autenticação é simulada mockando `obter_usuario_logado`, não construindo
cookie de sessão. É o que permite rodar sem o `SessionMiddleware` ativo — e
também o que explica os cinco testes pulados.

### Teste de repositório

Mocka `conexao_db` ou `util.database`, e verifica **a chamada**: que SQL foi
executado, com quais parâmetros, e se houve `commit`. Não há banco.

### Teste de serviço

Mocka o repositório e o cliente HTTP, e verifica a decisão. Os melhores
exemplos do repositório estão aqui:

- `test_campos_merge.py` (44 testes) — a mesclagem de campos de vários PDFs,
  caso a caso, incluindo a exceção de `capa.is_car`.
- `test_pasta_service.py` (26) — regras da árvore de pastas, inclusive recusa
  de ciclo.
- `test_searchable_service.py` (7) — reaproveitamento entre "irmãos" e
  comportamento com o recurso desligado.
- `test_conclusao_ocr_service.py` (6) — reconciliação, inclusive com a API
  indisponível (o `break` do ciclo) e com `404`.
- `test_solicitacao_desvinculo_service.py` (14) — uma pendente por par.

### Teste de callback

`test_ocr_callback.py` (7) e `test_searchable_callback.py` (7) cobrem token
inválido, corpo inválido, payload incompleto, `job_id` sem vínculo local e
**idempotência** — recebendo o mesmo aviso duas vezes.

---

## 7. Escrevendo um teste novo

1. Escolha o diretório pela camada que está testando.
2. Reaproveite as fixtures do `conftest.py`. Precisando de uma nova que sirva a
   mais de um arquivo, ela vai para lá.
3. Nome do teste descrevendo o comportamento, em português:
   `test_upload_recusa_arquivo_que_nao_e_pdf`.
4. Cubra o caminho feliz **e** o negado. Teste de rota sem o caso `403` deixa
   passar erro de autorização, que é a classe de bug mais cara deste sistema.
5. `pytest -k <palavra>` enquanto escreve; `pytest` inteiro antes do commit.

### Verificar que o teste testa

Antes de considerar um teste pronto, confirme que ele **falha** sem a correção
ou a funcionalidade. Foi essa prática que validou as correções de IDOR da
auditoria de segurança: os testes foram rodados e verificados como falha real
antes da correção, e passando depois — não apenas "não quebrou nada".

---

## 8. O que a suíte não cobre

| Área | Situação | Como é verificado hoje |
| --- | --- | --- |
| SQL de verdade | Nenhum teste toca MySQL | Manualmente, em ambiente local com `docker compose up -d mysql` |
| Login com cookie real | 4 testes pulados | Manualmente, pelo navegador, contra o `docker compose` completo |
| API de extração real | Sempre mockada | Manualmente, após deploy: upload, abrir processo, renderizar PDF |
| PDF pesquisável real | Sempre mockado | Manualmente |
| E-mail | Nunca enviado | Manualmente, com SMTP configurado |
| Frontend | **Zero testes** | `npm run typecheck` e `npm run build` |
| Migrações de `scripts/` | Sem teste | Manualmente, em cópia do banco |
| Integração ponta a ponta | Nenhuma | `docker compose` completo, manualmente |

Não há **CI configurada** neste repositório: nada roda a suíte
automaticamente em push ou PR. Enquanto for assim, `pytest` e
`npm run typecheck` antes de abrir PR são responsabilidade de quem abre.

### As duas lacunas que mais pesam

1. **Frontend sem teste nenhum.** `PdfViewer.tsx` tem 1 713 linhas e
   `CamposEditor.tsx` 1 361, as duas peças com mais histórico de bug no
   projeto, e nenhuma linha de teste automatizado. `npm run typecheck` pega erro
   de tipo, não de comportamento.
2. **Caminho de login não exercitado automaticamente.** É o caminho pelo qual
   todo usuário passa, e o único verificado só a olho.
