# Arquitetura — integracar-gestao

Como o sistema é montado, por que foi montado assim, e o que acontece entre o
clique do usuário e a resposta na tela.

Pré-requisito: [README.md](README.md).

---

## 1. Duas máquinas, dois mundos

O sistema de gestão roda em uma **VPS** modesta: dois núcleos, compartilhados
com o próprio site que os bolsistas usam. O trabalho pesado — rasterizar um PDF
de 60 páginas, rodar um modelo de visão em cada página, rodar Tesseract e
Ghostscript — roda em uma **workstation** com GPU, em outro lugar da rede.

```
   VPS                                  WORKSTATION
   leve, sempre de pé                   pesada, com GPU
   +-------------------------+          +--------------------------------+
   | frontend SSR (Node 20)  |          | gateway nginx                  |
   | API FastAPI             |  HTTPS   |   /api/          integracar-   |
   | MySQL 8.0               | <------> |                  backend + ocr |
   +-------------------------+          |   /pesquisavel/  PDF-Pesquisavel|
                                        | Postgres (fila + extração)     |
                                        +--------------------------------+
```

Três consequências diretas dessa separação, e são elas que explicam a maior
parte do código de integração:

1. **Tudo é assíncrono.** Nenhuma requisição de usuário espera o OCR terminar.
   O sistema registra "está na fila" e descobre depois que terminou.
2. **Descobrir que terminou tem dois caminhos.** O caminho rápido é o
   *callback*: o serviço avisa. O caminho de segurança é a *reconciliação*: um
   laço periódico pergunta. Os dois existem porque o callback se perde — basta
   o processo reiniciar na hora errada.
3. **A autenticação entre máquinas não é a do usuário.** Não há cookie nem
   sessão entre VPS e workstation: cada chamada leva um token de serviço no
   header `X-Service-Token`, e cada direção tem o seu.

---

## 2. Camadas do backend

```
  HTTP
   │
   ▼
 routes/        Declara método, caminho e perfil autorizado. Nada mais.
   │            Uma pasta por perfil de usuário.
   ▼
 controllers/   Fronteira HTTP: lê o corpo, valida forma, escolhe o status,
   │            monta o JSON, registra o log com contexto da requisição.
   ▼
 services/      Regra de negócio. Não conhece Request nem Response.
   │            É aqui que mora a decisão: o que é um processo, quando
   │            reaproveitar um PDF pesquisável, como combinar campos.
   ▼
 data/repo/     Acesso ao banco. Uma função por operação, SQL vindo de
   │            data/sql/, retorno em data/model/ (dataclasses).
   ▼
 MySQL
```

`util/` atravessa todas as camadas: autenticação, CSRF, rate limiting, log,
fuso horário e a leitura e validação das variáveis de ambiente.

### Por que `data/sql/` separado de `data/repo/`

Todo SQL é literal, em constantes de módulo, com parâmetros `%s` — não há ORM.
O ganho é poder ler a consulta exata que vai ao banco, incluindo os comentários
que explicam a escolha (por exemplo o `ROW_NUMBER() OVER (PARTITION BY campo)`
que define o veredito atual de um campo em
`data/sql/feedback_campo_sql.py`). O custo é não ter migração automática; ver
[banco-de-dados.md](banco-de-dados.md).

### Por que rotas por perfil, e não por recurso

A autorização do projeto é por **perfil**, não por objeto: um bolsista não vê
os processos de outro, um coordenador vê os de todos. Agrupar as rotas por
perfil deixa a decisão de acesso visível no arquivo — `/bolsista/*` tem
`@requer_autenticacao(["bolsista", "coordenador"])` na linha de cima de cada
endpoint — em vez de escondida em uma regra dinâmica.

O preço é duplicação: `listar_atividades` existe em quatro arquivos. Ela é
mitigada por `controllers/shared/atividades_controller.py`, que concentra o
miolo, e por `services/atividade_service.py`, que concentra a regra; os
controllers por perfil só passam a função de escopo (`own`, `campus`, `all`).

---

## 3. Ciclo de vida de uma requisição

Os middlewares de `main.py` executam na ordem inversa de registro. Para uma
requisição que chega:

```
1. catch_exceptions_middleware     Rede de segurança: exceção não tratada
                                   vira 500 com JSON {"erro": "..."}
2. security_headers_middleware     X-Content-Type-Options, X-Frame-Options,
                                   X-XSS-Protection; em produção, CSP + HSTS
3. csrf_middleware                 Métodos não seguros exigem X-CSRF-Token
                                   válido. Isenções: /login e os 2 callbacks
4. request_context_middleware      Gera request_id (UUID), extrai usuário, IP
                                   e user-agent, deixa em request.state
5. SessionMiddleware (Starlette)   Decodifica o cookie assinado em
                                   request.session
6. CORSMiddleware                  Origens permitidas por ambiente
   │
   ▼
   rota  ->  @requer_autenticacao  ->  controller  ->  service  ->  repo
```

### Detalhes que já causaram incidente

**`get_route_path`, não `request.url.path`, na isenção de CSRF.** Em produção o
app é montado sob `root_path="/gestao/api"`, e `url.path` carrega esse prefixo
quando o nginx repassa o caminho inteiro: `/gestao/api/ocr-callback` nunca
casaria com `/ocr-callback` e a isenção virava letra morta. Resultado: os dois
webhooks levavam `403` em produção e o job ficava eternamente `pending`. A
comparação usa a mesma função que o roteador do Starlette usa para casar rota.

**Handler global de `RequestValidationError`.** Sem ele, o FastAPI devolve
`{"detail": [...]}` — uma lista de objetos. O frontend só sabe exibir `erro` ou
`detail` como texto; ao receber o array tentava renderizar objeto em JSX e
quebrava a página inteira (tela branca) em vez de mostrar a mensagem de
validação. O handler reduz à primeira mensagem legível, no formato
`{"erro": "..."}` usado pelo resto da API.

**Assinatura reescrita no decorador de autenticação.** `usuario_logado` é
injetado pelo decorador a partir da sessão, não vem do cliente. Sem esconder
esse parâmetro da assinatura que o FastAPI inspeciona, rotas que também
recebem `File`/`Form` (multipart) fazem o FastAPI tentar ler `usuario_logado`
da requisição, e um campo homônimo enviado pelo cliente quebra com
`422 Input should be a valid dictionary`.

---

## 4. O frontend não é um cliente da API

Em modo framework do React Router 7 com SSR, `loader` e `action` rodam **no
servidor Node**. O navegador nunca fala com a API de gestão: ele fala com o
servidor do frontend, que fala com a API.

```
Navegador  --(HTML + fetch de dados da própria rota)-->  Node (React Router)
                                                              │
                                        repassa Cookie, busca │ CSRF, injeta
                                        X-CSRF-Token          │ Host em prod
                                                              ▼
                                                        API FastAPI
```

Isso resolve três coisas de uma vez:

- **O cookie de sessão nunca é visto por JavaScript de navegador**, porque
  quem o carrega é o servidor.
- **Não existe variável pública com a URL da API.** `API_URL` é lida só em
  `process.env`, no servidor; por padrão aponta para `http://backend:8000`, o
  nome do serviço na rede Docker.
- **O CSRF funciona sem o navegador participar.** Como o proxy roda no
  servidor, ele busca o token em `GET /csrf-token` usando o cookie repassado e
  o envia no header na chamada mutante. A função é memoizada por `Request`
  para não repetir a busca quando um mesmo loader dispara mais de uma mutação.

E cria uma sutileza própria: a sessão do backend é *stateless*, o cookie
carrega o próprio token assinado. Se `GET /csrf-token` precisou criar um token
novo, o `Set-Cookie` da resposta já reflete isso e **tem que substituir** o
cookie original na chamada seguinte — senão o token enviado não bate com o que
está no cookie repassado. É o que o campo `cookie` de `CsrfInfo` faz
(`frontend/app/services/api.server.ts`).

---

## 5. Decisões de projeto e seus motivos

| Decisão | Motivo |
| --- | --- |
| Integração com PDF pesquisável é opcional por configuração | Tornar obrigatório derrubaria no boot toda instalação que ainda não subiu o serviço. É melhoria de visualização, não requisito de funcionamento |
| Integração com a API de extração é obrigatória, falha no boot | Sem ela o sistema não tem função. Falhar alto é melhor que aceitar upload e perder o documento |
| Callback **e** reconciliação periódica | Callback é rápido mas se perde. Reconciliação é lenta mas não se perde. Ver `services/conclusao_ocr_service.py` |
| Callbacks idempotentes, respondendo 2xx para job desconhecido | A API de extração retenta até receber 2xx. Se o vínculo local não existe mais, o problema não é dela; responder 2xx evita retentativa infinita |
| Feedback de campo append-only, sem `UNIQUE` | Sobrescrever apagava o sinal de erro. Marcar errado, corrigir e votar certo apagava o "errado" para sempre — e é esse o dado que a métrica de qualidade precisa |
| Correção de campo gera feedback automático | A correção já confessa que o valor anterior estava errado, sem depender de alguém lembrar de votar antes de editar |
| Reaproveitar PDF pesquisável de "irmão" | A API de extração deduplica por hash; dois vínculos para o mesmo `documento_id` não precisam de dois OCRs. O segundo aponta para o pesquisável do primeiro |
| Upload sem pasta cria um processo por PDF | "Processos Avulsos" como processo único virava um saco com dezenas de PDFs sem relação. Virou categoria com uma subpasta por PDF |
| Tipo de pasta fixado na criação | Validar passa a ser conferir que a operação bate com o tipo, nunca inferir pelo conteúdo |
| `SET time_zone = '-03:00'` a cada checkout do pool | O pool usa `pool_reset_session=True`, que roda `RESET_CONNECTION` a cada `close()` e desfaz o fuso aplicado só no connect inicial. Sem reaplicar, apenas o primeiro uso de cada conexão ficava correto e o resto voltava ao fuso do servidor (UTC) silenciosamente — horários 3 h adiantados |
| Offset fixo, não `America/Sao_Paulo` | Não depende da tabela de fusos do MySQL estar carregada, e o Brasil não tem mais horário de verão |
| Framework de viewer do pdf.js, não canvas na mão | Rolagem contínua, camada de texto posicionada e busca com destaque já vêm prontos e testados. Reimplementar teve baixa recompensa quando foi tentado |
| `import` dinâmico de `pdfjs-dist` | O pacote só funciona no navegador; `import` estático no topo do arquivo seria avaliado também no SSR e derrubaria o processo no boot |
| `routeDiscovery: { mode: "initial" }` | No modo lazy, rota alcançada só por redirecionamento do servidor (como `/configurar-senha`) nunca é descoberta a tempo e a transição no cliente falha |
| `allowedActionOrigins` explícito em produção | A proteção nativa do React Router 7.12+ compara `Host` recebido pelo Node com o `Origin` do navegador. Atrás de proxy que não repasse `Host` corretamente, toda action era rejeitada. O nginx aceita o domínio com e sem `www` no mesmo bloco, então as duas formas estão liberadas |

---

## 6. Observabilidade

Log estruturado em `util/logging.py`, com `RotatingFileHandler` em
`logs/app.log` e saída no stdout. Todo registro carrega o contexto da
requisição, injetado por `request_context_middleware`:

```
2026-10-06 14:31:02 | INFO | controllers.bolsista.ocr_controller |
  req_id=3f2b... | user_id=42 | PDF enviado para processamento OCR
```

Campos disponíveis em `extra`: `request_id`, `user_id`, `role`, `ip`,
`user_agent`, mais o que cada ponto de log acrescenta (`job_id`,
`documento_id`, `cod_pasta`, `error`). `ContextFilter` preenche `-` quando o
campo não existe, para que log fora de requisição (laço de reconciliação,
boot) não quebre a formatação.

Há decisões de acesso registradas em `WARNING`: usuário não autenticado,
permissão insuficiente, sessão expirada por inatividade, rate limit excedido,
token de callback inválido. São a trilha de auditoria do sistema.

---

## 7. Onde seguir

- Endpoints e middlewares em detalhe: [backend.md](backend.md)
- Telas, rotas e visualizador: [frontend.md](frontend.md)
- Tabelas e migrações: [banco-de-dados.md](banco-de-dados.md)
- Contratos com os serviços externos: [integracoes.md](integracoes.md)
