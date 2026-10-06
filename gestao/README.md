# Sistema de Gestão IntegraCAR (`integracar-gestao`)

Sistema web onde bolsistas, orientadores, consultores e coordenadores do
projeto IntegraCAR trabalham os processos de CAR: enviam os PDFs, acompanham o
processamento, revisam campo a campo o que foi extraído do documento e
registram atividades.

É o único sistema do projeto que o usuário final abre no navegador. Todo o
resto (leitura dos PDFs por modelo de visão, geração de camada de texto) roda
em serviços de retaguarda que este sistema aciona e aguarda.

> **Leitura em dois níveis.** Cada seção começa com a explicação em linguagem
> comum e termina com o detalhe técnico. Quem quer entender o sistema pode
> parar no primeiro parágrafo; quem vai mexer no código precisa do segundo.

---

## Sumário

1. [O problema que o sistema resolve](#1-o-problema-que-o-sistema-resolve)
2. [Como funciona, por cima](#2-como-funciona-por-cima)
3. [Perfis de usuário](#3-perfis-de-usuário)
4. [Funcionalidades](#4-funcionalidades)
5. [Stack e versões](#5-stack-e-versões)
6. [Mapa do repositório](#6-mapa-do-repositório)
7. [Começando](#7-começando)
8. [Variáveis de ambiente](#8-variáveis-de-ambiente)
9. [Comandos do dia a dia](#9-comandos-do-dia-a-dia)
10. [Estado atual e limitações conhecidas](#10-estado-atual-e-limitações-conhecidas)
11. [Documentação detalhada](#11-documentação-detalhada)

---

## 1. O problema que o sistema resolve

Um processo de CAR chega como um PDF escaneado, com dezenas de páginas:
capa, requerimento, CCIR, quadro de áreas. Os dados que interessam
(interessado, CPF, município, matrícula, áreas em hectares) estão ali como
imagem, não como texto. Antes do IntegraCAR, alguém abria o PDF e digitava
tudo em planilha.

O sistema troca "digitar" por "conferir": a máquina lê o documento e propõe os
valores, o bolsista confirma ou corrige. Corrigir um campo é mais rápido que
digitá-lo, e cada correção fica registrada — o que permite medir onde a
extração automática erra e melhorá-la.

**Em termos técnicos**, o sistema é a camada de orquestração e revisão humana
de um pipeline de extração de informação de documentos: recebe o documento,
despacha jobs assíncronos para dois serviços independentes (extração de campos
e OCR de camada de texto), reconcilia o estado desses jobs, e expõe o
resultado em uma interface de revisão com rastreabilidade de origem e coleta
de sinal de qualidade (feedback por campo).

---

## 2. Como funciona, por cima

```
                       VPS (este repositório)
  +--------------------------------------------------------------+
  |  Navegador  --->  Frontend React Router 7 (SSR, Node)        |
  |                        |                                     |
  |                        | loaders/actions, servidor-a-servidor |
  |                        v                                     |
  |                   API FastAPI  <--->  MySQL 8.0              |
  +------------------------|-------------------------------------+
                           | HTTPS, header X-Service-Token
                           v
                 WORKSTATION (outros repositórios)
  +--------------------------------------------------------------+
  |  gateway nginx                                               |
  |    /api/         -->  integracar-backend (extração de campos)|
  |                         + integracar-ocr (worker, fila)      |
  |    /pesquisavel/ -->  PDF-Pesquisavel (OCRmyPDF)             |
  +--------------------------------------------------------------+
```

O caminho de um PDF, do envio até a tela de revisão:

1. **Envio.** O bolsista arrasta um ou mais PDFs na tela de envio. O frontend
   repassa ao backend, que valida (é PDF? não está vazio? cabe em 300 MB?) e
   manda tudo para a API de extração em um único `POST /uploads`. Validação é
   tudo-ou-nada: se um arquivo é inválido, nenhum é enviado.
2. **Dois processamentos em paralelo.** A API de extração enfileira o job de
   leitura dos campos. Em paralelo, o mesmo PDF vai para o serviço de PDF
   pesquisável, que devolve o documento com camada de texto — é isso que faz
   a lupa funcionar dentro de um documento escaneado.
3. **Registro local.** O backend grava um *vínculo* na tabela `Ocr_Documento`:
   quem enviou, qual o `job_id` e o `documento_id` do lado da API de extração,
   em que pasta o PDF foi guardado, e em que pé está cada um dos dois
   processamentos.
4. **Conclusão.** Quando a extração termina, a API de extração chama de volta
   `POST /ocr-callback`; o serviço de PDF pesquisável chama
   `POST /searchable-callback`. Os dois callbacks são autenticados por token de
   serviço, são idempotentes, e um laço de reconciliação periódico cobre o caso
   de callback perdido (reinício do processo, queda de rede).
5. **Revisão.** O bolsista abre a tela do processo: à esquerda os campos
   extraídos, à direita o PDF. Clicar na lupa ao lado de um campo procura
   aquele valor dentro do PDF e destaca a ocorrência, usando a página de
   origem que a extração registrou em `campos._fontes`.
6. **Correção e sinal de qualidade.** Editar um campo grava a correção e, de
   tabela, um registro de feedback de origem `correcao`. Polegar para cima ou
   para baixo grava feedback de origem `voto`. O histórico é *append-only*: uma
   correção nunca apaga o erro anterior, porque é justamente o erro que a
   métrica de qualidade precisa ver.

O detalhamento de cada etapa está em [arquitetura.md](arquitetura.md) e
[integracoes.md](integracoes.md).

### Degradação deliberada

As integrações são **best effort por projeto**: faltando configuração ou
estando o serviço fora do ar, o sistema continua inteiro, só sem o recurso
associado.

| Serviço fora do ar | O que acontece |
| --- | --- |
| PDF pesquisável | O PDF original continua sendo servido. Perde-se a busca dentro do documento; a extração de campos não é afetada |
| API de extração | Envio de novos PDFs falha com `502` e mensagem clara. Documentos já processados continuam visíveis e editáveis |
| E-mail (SMTP) | Recuperação de senha registra o erro no log e não propaga exceção para o usuário |

---

## 3. Perfis de usuário

O perfil fica na coluna `Usuario.role_usuario` e é a base de toda autorização.
No backend, cada perfil tem seu próprio grupo de rotas, protegido pelo
decorador `@requer_autenticacao([...])` (`util/auth_decorator.py`).

| Perfil | O que faz | Rotas próprias |
| --- | --- | --- |
| `bolsista` | Envia PDFs, organiza em pastas, revisa e corrige campos, registra atividades | 46 endpoints `/bolsista/*` |
| `coordenador` | Tudo que o bolsista faz, mais: cadastra e gerencia bolsistas, visão geral do projeto, painel de monitoramento da extração | 15 endpoints `/coordenador/*` (e acesso a `/bolsista/*`) |
| `orientador` | Registra e consulta atividades da sua equipe | 6 endpoints `/orientador/*` |
| `consultor` | Registra e consulta atividades no escopo do campus | 6 endpoints `/consultor/*` |
| `gestor_tecnico` | **Previsto, ainda sem rotas próprias.** O frontend já reserva as permissões (edita e exclui, não cria, enxerga tudo) | — |
| `gestor_administrativo` | **Previsto, ainda sem rotas próprias.** O frontend já reserva as permissões (somente leitura) | — |

Dois cortes transversais convivem com os perfis:

- **Quem pode processar documentos** — hoje `bolsista` e `coordenador`, corte
  centralizado em `frontend/app/lib/roles.ts` porque é checado no menu lateral
  e em cinco rotas.
- **Escopo de visão** (`own` / `campus` / `all`) e capacidades de escrita, em
  `frontend/app/hooks/usePermission.ts`. Isso governa a interface; a decisão
  que vale é sempre a do backend.

> **Cuidado ao ler o frontend.** O tipo `UserRole`
> (`frontend/app/services/session.server.ts`) lista seis perfis, incluindo os
> dois ainda não implementados no backend. Um usuário gravado com
> `gestor_tecnico` consegue entrar e navegar, mas não tem endpoint próprio
> nenhum: as telas que dependem de rota por perfil vêm vazias.

---

## 4. Funcionalidades

### Processos e pastas

O bolsista organiza o trabalho em uma árvore de pastas de dois tipos, fixados
na criação e nunca inferidos do conteúdo:

- **categoria** — pasta de organização, contém outras pastas.
- **processo** — a unidade de trabalho. Contém os PDFs de *um* processo de CAR.

Upload sem pasta escolhida cai em `Processos Avulsos`, uma categoria criada
automaticamente (uma por usuário), e **cada PDF ganha o próprio processo
dentro dela**. Isso é resposta a um problema real: antes, "Avulsos" era um
processo único e virava um saco com dezenas de PDFs sem relação entre si.

Um processo com vários PDFs tem a visão **"ver junto"**: os campos de todos os
PDFs do processo combinados em uma leitura só
(`services/campos_merge.py`). Quando dois PDFs divergem em uma seção, o
resultado guarda uma lista de versões daquela seção — uma por PDF — em vez de
amontoar os valores divergentes dentro de um container único, onde não se
sabia de qual documento vinha cada um.

### Revisão de campos lado a lado com o PDF

A tela de detalhes do processo
(`frontend/app/ui/templates/ProcessoTemplate.tsx`) é o coração do sistema:

- Painel esquerdo: campos extraídos agrupados por seção (`capa`,
  `requerimento_digital`, `ccir`, `quadro_areas`), editáveis um a um.
- Painel direito: o PDF, renderizado com `pdfjs-dist` usando o framework de
  viewer oficial do pdf.js (o mesmo código que exibe PDF dentro do Firefox) —
  rolagem contínua, camada de texto posicionada e busca com destaque vêm
  prontos em vez de reimplementados.
- A lupa ao lado de um campo procura o valor no PDF e alinha o destaque
  encontrado à altura em que o campo foi clicado. Campo com vários valores
  ("624868, 52695") gera um termo de busca por valor, não uma busca pelo texto
  inteiro.
- Módulos fiscais são calculados no cliente a partir de
  `util/tabela_modulos.json` (tabela do INCRA por município do ES). O arquivo
  do backend é a fonte; `frontend/app/lib/tabela_modulos.json` é cópia manual.

### Qualidade da extração

Cada campo aceita avaliação (polegar para cima/baixo) e cada edição gera
automaticamente um registro de correção. As tabelas `Feedback_Campo` (por
documento) e `Feedback_Campo_Pasta` (por processo, na visão combinada) são
**append-only, sem `UNIQUE`**: reavaliar não sobrescreve. O veredito atual de
um campo é a linha mais recente (`ROW_NUMBER() OVER (PARTITION BY campo ...)`),
e o histórico inteiro alimenta a métrica do coordenador.

Campos numerados por ocorrência (`ccir_1_codigo`, `ccir_2_codigo`) e índices de
versão (`capa.0.interessado`, `capa.1.interessado`) são normalizados em
`campo_agrupado` por `services/feedback_campo.py` — sem isso, o painel do
coordenador mostraria dezenas de linhas quase idênticas em vez de "código do
CCIR: 20% de acerto", que é a pergunta que ele quer responder.

### Monitoramento (coordenador)

- **Visão geral** (`/visao-geral`): números do projeto.
- **Monitoramento da extração** (`/monitoramento-extracao`): fila atual,
  histórico e profundidade da fila ao longo do tempo, com dados vindos da API
  de extração. As médias são ponderadas pelo total de cada período, não médias
  simples — um dia com 1 documento não pode pesar igual a um dia com 50.

### Atividades

Registro de atividades por usuário, com orientador vinculado, usado para
acompanhamento do bolsista. Existe para os quatro perfis implementados, cada
um com seu escopo de visibilidade (próprias, do campus, todas).

### Notificações de desvínculo

A API de extração deduplica documentos por hash: dois bolsistas que sobem o
mesmo arquivo compartilham o mesmo `documento_id`. Quando um deles exclui seu
vínculo, os outros recebem uma solicitação de desvínculo pendente
(`Solicitacao_Desvinculo`) em vez de perder o documento sem aviso. Uma
pendência por par (documento, destinatário), para não empilhar avisos
repetidos.

---

## 5. Stack e versões

Versões conforme `requirements.txt` e `frontend/package.json` em 2026-10-06.

### Backend

| Componente | Versão | Papel |
| --- | --- | --- |
| Python | 3.12 | Imagem `python:3.12-slim` |
| FastAPI | 0.115.5 | Framework HTTP |
| uvicorn | 0.32.1 | Servidor ASGI |
| MySQL | 8.0 | Banco, `utf8mb4` / `utf8mb4_unicode_ci` |
| mysql-connector-python | 8.2.0 | Driver, pool de 20 conexões |
| bcrypt | 4.1.2 | Hash de senha |
| itsdangerous | 2.2.0 | Assinatura do cookie de sessão |
| slowapi | 0.1.9 | Rate limiting (armazenamento em memória) |
| fastapi-mail | 1.4.1 | E-mail transacional via SMTP (Brevo) |
| httpx | 0.28.1 | Cliente HTTP das integrações |
| pandas / numpy | 2.3.3 / 2.3.4 | Exportação CSV e XLSX |
| pytest | 8.3.4 | Testes, com `pytest-cov`, `pytest-mock`, `pytest-asyncio` |

### Frontend

| Componente | Versão | Papel |
| --- | --- | --- |
| React Router | 7.18.3 | **Modo framework**, com SSR — loaders e actions rodam no servidor |
| React | 19.2.4 | Biblioteca de UI |
| Vite | 7.1.7 | Bundler por baixo do React Router |
| Tailwind CSS | 4.1.13 | Estilo, via `@tailwindcss/vite` |
| TypeScript | 5.9.2 | Tipos, `npm run typecheck` |
| pdfjs-dist | 5.7.284 | Visualizador de PDF e camada de texto |
| recharts | 3.10.1 | Gráficos do monitoramento |
| lucide-react, sonner, Radix UI | — | Ícones, notificações, primitivos acessíveis |
| Node | 20 | Imagem `node:20-alpine` |

> **Não existe chamada do navegador para a API de gestão.** O frontend está em
> modo framework: `loader` e `action` rodam no servidor Node e falam com a API
> por `~/services/*.server.ts`, repassando o cookie de sessão. Não há, e não
> deve haver, variável de ambiente pública com a URL da API.

---

## 6. Mapa do repositório

```
integracar-gestao/
├── main.py                  Montagem do app FastAPI: middlewares, handlers, routers
├── conexao_db.py            Pool MySQL (20 conexões) e fuso -03:00 por checkout
├── initialize_database.py   Recria o schema do zero. DESTRUTIVO
├── check-deploy.py          Validação pré-deploy (9 verificações)
├── requirements.txt
├── Dockerfile               Backend
├── docker-compose.yml       mysql + backend + frontend
│
├── routes/                  Uma pasta por perfil; só declara rota e delega
│   ├── publico/             login, sessão, senha, os dois callbacks
│   ├── bolsista/            46 endpoints
│   ├── coordenador/         15 endpoints
│   ├── consultor/           6 endpoints
│   └── orientador/          6 endpoints
│
├── controllers/             HTTP: valida entrada, monta resposta, registra log
│   ├── bolsista/            ocr_controller.py (885 linhas), pasta_controller.py (475)
│   ├── coordenador/  consultor/  orientador/  publico/
│   └── shared/              base reaproveitada entre perfis
│
├── services/                Regra de negócio, sem HTTP
│   ├── ocr_client.py        Cliente da API de extração (28 funções)
│   ├── searchable_client.py Cliente do PDF pesquisável
│   ├── searchable_service.py Quando enfileirar e quando reaproveitar
│   ├── conclusao_ocr_service.py Reconciliação periódica de jobs
│   ├── campos_merge.py      Visão "ver junto" de um processo
│   ├── pasta_service.py     Regras da árvore de pastas
│   ├── feedback_campo.py    Normalização de campo para a métrica
│   ├── atividade_service.py, solicitacao_desvinculo_service.py, email_service.py
│   └── templates/           reset_password.html
│
├── data/
│   ├── model/               dataclasses do domínio
│   ├── repo/                acesso ao banco, uma função por operação
│   ├── sql/                 SQL literal, incluindo CREATE TABLE e ALTERs
│   └── schemas/             modelos Pydantic de entrada e validadores
│
├── util/
│   ├── auth_decorator.py    @requer_autenticacao, sessão, inatividade (90 min)
│   ├── csrf.py  security.py  rate_limit.py  logging.py  timezone.py
│   ├── ocr_config.py        obrigatório: falha no boot se faltar variável
│   ├── searchable_config.py opcional: ausente desliga o recurso
│   └── tabela_modulos.json  módulo fiscal por município do ES (fonte)
│
├── scripts/                 13 migrações pontuais, idempotentes, rodadas à mão
├── seeders/                 importação inicial de campus e usuários a partir de CSV
├── tests/                   485 testes
│
└── frontend/
    ├── app/
    │   ├── routes.ts        Declaração explícita das rotas
    │   ├── routes/          Um arquivo por rota; `_app.*` fica sob o layout logado
    │   ├── services/        *.server.ts — única porta de entrada para a API
    │   ├── ui/              atoms / molecules / organisms / templates
    │   ├── hooks/  lib/  types/
    │   └── app.css
    ├── react-router.config.ts  basename, SSR, origens de action permitidas
    ├── vite.config.ts
    └── Dockerfile           Build multiestágio
```

A arquitetura em camadas e o porquê de cada uma estão em
[backend.md](backend.md) e [frontend.md](frontend.md).

---

## 7. Começando

Pré-requisitos: Python 3.12+, Node 20+, Docker (para o MySQL) e Git.

```bash
git clone https://github.com/integraCAR/integracar-gestao.git
cd integracar-gestao

# 1. Backend: ambiente e dependências
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 2. Configuração
cp .env.example .env
python3 -c "import secrets; print(secrets.token_urlsafe(32))"   # cole em SECRET_KEY
# edite o .env: banco, e-mail e as quatro variáveis OCR_* (obrigatórias)

# 3. MySQL local
docker compose up -d mysql

# 4. Schema e dados iniciais  (ATENCAO: apaga tudo que existir)
python initialize_database.py

# 5. Backend em modo dev, porta 8000
uvicorn main:app --reload
```

Em outro terminal:

```bash
cd frontend
npm install
npm run dev        # http://localhost:5173
```

- Interface: <http://localhost:5173>
- API, documentação interativa: <http://localhost:8000/docs> e `/redoc`

O passo a passo completo, incluindo o que fazer quando algo não sobe, está em
[desenvolvimento.md](desenvolvimento.md).

---

## 8. Variáveis de ambiente

### Backend (`.env` na raiz)

**Obrigatórias — o processo não sobe sem elas.** `conexao_db.py` valida as de
banco, `main.py:205` a `SECRET_KEY` e `util/ocr_config.py` as de OCR; todas
falham alto e claro no boot.

| Variável | Para que serve |
| --- | --- |
| `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`, `DB_NAME` | Conexão MySQL |
| `SECRET_KEY` | Assina o cookie de sessão. Nunca reaproveitar entre ambientes |
| `ENVIRONMENT` | `development` ou `production`. Governa `root_path`, CSP, HSTS, cookie `secure` e CORS |
| `FRONTEND_URL` | Origem liberada no CORS e base dos redirecionamentos de login |
| `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_FROM`, `MAIL_SERVER`, `MAIL_PORT` | SMTP da recuperação de senha |
| `OCR_API_BASE` | URL da API de extração |
| `OCR_SERVICE_TOKEN` | Token que **este** sistema envia em `X-Service-Token` |
| `OCR_CALLBACK_TOKEN` | Token que este sistema **exige** no callback recebido |
| `OCR_PUBLIC_BASE` | Base pública própria, usada para montar a `callback_url`. Precisa ser alcançável pela internet |

**Opcionais.**

| Variável | Padrão | Para que serve |
| --- | --- | --- |
| `MAIL_FROM_NAME` | — | Nome do remetente |
| `OCR_RECONCILIACAO_INTERVALO_S` | `300` | Intervalo da reconciliação de jobs pendentes. `0` desliga |
| `SEARCHABLE_API_BASE` | — | URL do serviço de PDF pesquisável |
| `SEARCHABLE_SERVICE_TOKEN` | — | Token enviado ao serviço |
| `SEARCHABLE_CALLBACK_TOKEN` | — | Token exigido no callback do serviço |

As três `SEARCHABLE_*` funcionam **em conjunto**: faltando qualquer uma,
`SEARCHABLE_ATIVO` é falso e o recurso fica desligado, servindo o PDF original.
Isso é deliberado — tornar obrigatório derrubaria no boot toda instalação que
ainda não subiu o serviço.

`OCR_SERVICE_TOKEN` e `OCR_CALLBACK_TOKEN` são **tokens diferentes**, cada um
com uma direção. Trocá-los um pelo outro produz `401` silencioso em um dos dois
sentidos.

### Frontend (`frontend/.env`)

| Variável | Padrão | Para que serve |
| --- | --- | --- |
| `API_URL` | `http://backend:8000` | Endereço da API visto **pelo servidor Node**. Em Docker, nome do serviço; nunca a URL pública, que faria o SSR sair para a internet e perder o cookie de sessão |
| `ENVIRONMENT` | — | `production` liga o prefixo `/gestao/api` nas chamadas e o `basename` `/gestao/` |

`NEXT_PUBLIC_API_URL`, presente em `frontend/.env` e no `docker-compose.yml`,
é resquício do frontend Next.js antigo e **não é lida por nenhum código atual**.
Ver [limitações conhecidas](#10-estado-atual-e-limitações-conhecidas).

---

## 9. Comandos do dia a dia

```bash
# ---------- Backend ----------
uvicorn main:app --reload               # dev, porta 8000
pytest                                  # suíte inteira, com cobertura (pytest.ini)
pytest tests/test_routes -v             # só as rotas
pytest -k nome_do_teste                 # por palavra-chave
python3 check-deploy.py                 # 9 verificações pré-deploy

# ---------- Frontend ----------
cd frontend
npm run dev                             # dev com HMR, porta 5173
npm run typecheck                       # react-router typegen && tsc
npm run build                           # build de produção
npm run start                           # serve o build (react-router-serve)

# ---------- Docker ----------
docker compose up -d mysql              # só o banco, para dev local
docker compose up -d --build            # stack inteira
docker compose logs -f backend
```

`pytest` sozinho já roda com cobertura: as flags estão em `pytest.ini`
(`--cov`, `--cov-branch`, relatórios em terminal, `htmlcov/` e
`coverage.xml`). Para rodar sem cobertura, use `pytest --no-cov`.

---

## 10. Estado atual e limitações conhecidas

Medido em 2026-10-06, no `main`.

### Números reais

| Medida | Valor | Como reproduzir |
| --- | --- | --- |
| Testes | 485 coletados: 480 passando, 5 pulados | `pytest` |
| Cobertura (linha e ramo) | 71,52% | `pytest`, linha `TOTAL` |
| Endpoints HTTP | 83 | `grep -rh '^@router\.' routes \| wc -l` |
| Tabelas MySQL | 9 | ver [banco-de-dados.md](banco-de-dados.md) |

### Pendências e armadilhas

- **`docker-compose.yml` não configura o frontend.** O serviço `frontend`
  recebe `NEXT_PUBLIC_API_URL`, que o React Router não lê, e **não** recebe
  `API_URL` nem `ENVIRONMENT`. Na prática o padrão `http://backend:8000`
  acerta o endereço por acidente (é o nome do serviço no compose), mas
  `ENVIRONMENT` ausente desliga o prefixo `/gestao/api` em tempo de execução
  enquanto o `basename` `/gestao/` é fixado em tempo de build por `NODE_ENV`.
  Conferir contra o nginx antes de confiar. Ver [deploy.md](deploy.md).
- **Rate limiting em memória.** `util/rate_limit.py` usa `storage_uri="memory://"`:
  o limite é por processo e zera a cada reinício. Com mais de um worker uvicorn,
  o limite efetivo é multiplicado pelo número de workers. Para valer em
  produção, precisa de Redis.
- **Senhas de contas importadas ficam em texto puro.** Decisão de negócio
  registrada em `docs/security/relatorio-seguranca.md`: a equipe precisa
  consultar a senha temporária para comunicar ao usuário no lançamento. As
  senhas definidas pelo próprio usuário usam bcrypt normalmente.
- **Bloqueio de primeiro acesso desativado.** O trecho que forçava a troca de
  senha no primeiro login está comentado em `util/auth_decorator.py`. A tela
  existe (`/configurar-senha`) e o endpoint funciona, mas nada obriga a passar
  por ela.
- **`gestor_tecnico` e `gestor_administrativo` sem backend.** Ver
  [perfis](#3-perfis-de-usuário).
- **`initialize_database.py` apaga o banco.** Não há sistema de migração: o
  schema nasce de `CREATE TABLE IF NOT EXISTS` nos repositórios e evolui por
  scripts pontuais em `scripts/`, rodados à mão uma vez cada. Em produção,
  rodar apenas o script da migração, nunca o `initialize_database.py`.
- **`initialize_database.py` não derruba a tabela de atividades.** O script faz
  `DROP TABLE IF EXISTS Atividades` (plural), mas a tabela criada e usada pelo
  sistema é `Atividade` (singular, `data/sql/atividades_sql.py`). O `DROP` é um
  no-op: ao reinicializar, as atividades antigas sobrevivem apontando para
  `cod_usuario` de usuários que acabaram de ser recriados com outros ids.
- **`frontend/app/lib/tabela_modulos.json` é cópia manual** de
  `util/tabela_modulos.json`. Mudou lá, copie aqui.
- **`.coveragerc` inclui `source = .`**, o que mede também arquivos de
  script e migração raramente executados; a cobertura real das camadas de
  rota, serviço e repositório é mais alta que os 71,52% do total.

---

## 11. Documentação detalhada

| Documento | Conteúdo |
| --- | --- |
| [arquitetura.md](arquitetura.md) | Camadas, ciclo de vida de uma requisição, decisões de projeto e seus motivos |
| [backend.md](backend.md) | Catálogo dos 83 endpoints, middlewares, autenticação, tratamento de erro |
| [frontend.md](frontend.md) | Rotas, telas, visualizador de PDF, proxy servidor-a-servidor |
| [banco-de-dados.md](banco-de-dados.md) | Dicionário de dados das 9 tabelas e as migrações |
| [integracoes.md](integracoes.md) | Contratos com a API de extração e com o PDF pesquisável |
| [seguranca.md](seguranca.md) | Sessão, CSRF, rate limiting, headers, autorização |
| [desenvolvimento.md](desenvolvimento.md) | Ambiente local, padrões de código, solução de problemas |
| [deploy.md](deploy.md) | Produção: Docker, nginx, HTTPS, backup, rollback |
| [testes.md](testes.md) | Estrutura da suíte, fixtures, cobertura |
