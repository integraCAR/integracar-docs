# Frontend — React Router 7 em modo framework

Interface web do sistema de gestão. Fica em `integracar-gestao/frontend/`.

Pré-requisitos: [README.md](README.md) e [arquitetura.md](arquitetura.md).

> **Aviso de documentação antiga.** `integracar-gestao/README.md` e
> `DEVELOPMENT.md` descreveram por muito tempo um frontend Next.js com
> `NEXT_PUBLIC_API_URL`. Esse frontend não existe mais: foi substituído por
> React Router 7 em modo framework, e o repositório `integracar-frontend`
> (outro projeto, da API de extração) está descontinuado. Se encontrar
> `NEXT_PUBLIC_*` em algum lugar, é resquício morto.

---

## 1. O modelo mental

Não é uma SPA que consome uma API. É uma aplicação **full-stack**: cada rota
tem um `loader` (busca os dados) e, quando escreve, um `action` — e os dois
**rodam no servidor Node**, nunca no navegador.

```
 Navegador                     Node (React Router)              API FastAPI
    │                                 │                              │
    │── GET /processos/42 ───────────►│                              │
    │                                 │── GET /bolsista/ocr/42 ─────►│
    │                                 │◄───── JSON ──────────────────│
    │◄──── HTML já renderizado ───────│                              │
    │                                 │                              │
    │── POST (action: editar campo) ─►│                              │
    │                                 │── GET /csrf-token ──────────►│
    │                                 │── PUT .../campo (+token) ───►│
    │◄──── redirect / revalidate ─────│                              │
```

Três regras que decorrem disso:

1. **Nenhum componente faz `fetch` para a API de gestão.** Toda chamada passa
   por `app/services/*.server.ts`. O sufixo `.server.ts` garante que o Vite
   nem inclua o arquivo no bundle do cliente.
2. **Não existe variável de ambiente pública com a URL da API.** `API_URL` é
   lida de `process.env`, só no servidor.
3. **O cookie de sessão é repassado pelo servidor**, nunca manipulado por
   JavaScript de navegador.

---

## 2. Rotas

Declaradas explicitamente em `app/routes.ts` (não por convenção de nome de
arquivo). O prefixo `_app.` marca as rotas que ficam sob o layout autenticado.

### Fora do layout — públicas e de dados

| Caminho | Arquivo | O que é |
| --- | --- | --- |
| `/` | `routes/login.tsx` | Tela de login (rota índice) |
| `/logout` | `routes/logout.tsx` | Action de saída |
| `/esqueci-senha` | `routes/esqueci-senha.tsx` | Solicitação de recuperação |
| `/recuperar-senha` | `routes/recuperar-senha.tsx` | Nova senha a partir do token |
| `/configurar-senha` | `routes/configurar-senha.tsx` | Primeiro acesso |
| `/sessao/atividade` | `routes/sessao.atividade.tsx` | Renova inatividade. Chamada só em interação real do usuário |
| `/processos/:id/pdf` | `routes/processos.$id.pdf.tsx` | Proxy do PDF, repassa `Range` |
| `/processos/:id/pdf-info` | `routes/processos.$id.pdf-info.tsx` | Como exibir o documento |
| `/processos/:id/paginas/:n/imagem` | `routes/processos.$id.paginas.$n.imagem.tsx` | Imagem da página `n` |
| `/documentos/export.csv` | `routes/documentos.export-csv.tsx` | Download |
| `/documentos/export.xlsx` | `routes/documentos.export-xlsx.tsx` | Download |
| `/pastas/:id/download.zip` | `routes/pastas.$id.download-zip.tsx` | Download do processo |
| `/pastas/todas` | `routes/pastas.todas.tsx` | Lista achatada, usada por seletores |
| `/ocr/:id/vinculos` | `routes/ocr.$id.vinculos.tsx` | Quem mais tem o documento |

As rotas de PDF, imagem, exportação e ZIP são **endpoints de dados**, não
telas: existem para que o navegador baixe o binário sem nunca ver o token de
serviço da API de extração.

### Dentro do layout `_app.tsx` — telas autenticadas

| Caminho | Tela |
| --- | --- |
| `/dashboard` | Lista de documentos, estado do processamento, atalhos |
| `/visao-geral` | Números do projeto (coordenador) |
| `/monitoramento-extracao` | Fila e histórico da extração (coordenador) |
| `/processos/enviar` | Envio de PDFs |
| `/processos/:id` | **Revisão de campos lado a lado com o PDF** |
| `/pastas` | Explorador de pastas |
| `/pastas/:id` | Conteúdo de uma pasta; em processo, a visão "ver junto" |
| `/notificacoes` | Solicitações de desvínculo |
| `/atividades`, `/atividades/:id`, `/atividades/novo` | Atividades |
| `/bolsistas`, `/bolsistas/:id`, `/bolsistas/novo` | Gestão de bolsistas (coordenador) |
| `*` | `routes/_app.$.tsx` — 404 dentro do layout |

O `loader` de `_app.tsx` é quem carrega o usuário da sessão; as telas filhas o
leem por `useRouteLoaderData("routes/_app")`, que é o que `usePermission`
consome.

---

## 3. Camada de serviços

| Arquivo | Responsabilidade |
| --- | --- |
| `config.server.ts` | Resolve `API_URL`, padrão `http://backend:8000`. Em SSR, **sempre** o endereço interno: sair para a internet para falar com o próprio backend perde o cookie e dá `401` |
| `api.server.ts` | `apiFetch` e `apiFetchBinary`. Monta a URL com o prefixo `/gestao/api` em produção, repassa cookie, obtém e injeta o token CSRF, acumula `Set-Cookie` do backend para devolver ao navegador |
| `session.server.ts` | Login, leitura de sessão, logout, troca de senha de primeiro acesso, tipo `User` e helpers `hasRole`/`hasAnyRole` |
| `permissions.server.ts` | Permissão de recurso no servidor (`getProcessResourcePermissions`) |
| `documentos.server.ts` | Documentos, campos, feedback, anotação |
| `pastas.server.ts` e `pastaActions.server.ts` | Leitura e mutação de pastas |
| `notificacoes.server.ts` | Solicitações de desvínculo |
| `coordenador.server.ts` | Visão geral, gráficos, monitoramento, bolsistas |

### CSRF no servidor

```
action dispara mutação
  └─ obterCsrfInfo(request)        memoizado por Request
       ├─ GET /csrf-token com o cookie repassado
       └─ devolve { token, cookie }
  └─ apiFetch injeta X-CSRF-Token
       └─ e SUBSTITUI o Cookie pelo da resposta do /csrf-token, se houver
```

A substituição do cookie é obrigatória: a sessão do backend é *stateless*, o
cookie carrega o próprio token assinado. Se `/csrf-token` criou um token novo,
o `Set-Cookie` daquela resposta é a única versão do cookie que combina com o
token recebido.

### Propagação de `Set-Cookie`

`apiFetch` com `propagarCookies: true` acumula os `Set-Cookie` do backend em um
`WeakMap` indexado pela `Request`, e `comSessao(request, payload)` os devolve
na resposta ao navegador. Sem isso, a renovação de sessão feita pelo backend se
perdia no caminho — foi o conserto do commit `c3fc23e`.

Um `Set-Cookie` completo (`nome=valor; Path=/; HttpOnly; ...`) não pode ser
reenviado como header `Cookie` de requisição: só o par `nome=valor` importa. É
o que `paraCookieDeRequisicao` faz.

---

## 4. Organização da interface

Atomic design, em `app/ui/`:

```
atoms/       Button, Input, Label, Select, Checkbox
molecules/   Marca, Paginacao, ThemeToggle
organisms/   PdfViewer (1713 linhas), CamposEditor (1361), Sidebar,
             BarraSuperior, UploadDropzone, AnotacoesCard, HistoricoTab,
             PerformanceTab, JobProgressoCard, ConfirmModal, LogoutModal,
             DevModal, VisaoGeralOcr, PaginaProcessandoView,
             MonitoramentoFilaChart, FilaProfundidadeChart, HoverBarChart
templates/   Uma por tela; recebe dados prontos do loader e não busca nada
```

A fronteira é deliberada: `routes/` conhece o servidor, `templates/` conhece
só props. Isso é o que torna as telas testáveis sem rede.

### `hooks/` e `lib/`

| Arquivo | O que faz |
| --- | --- |
| `hooks/usePermission.ts` | Mapa de capacidades por perfil (`canCreate`, `canEdit`, `canDelete`, `viewScope`, `canManageBolsistas`, `viewBolsistasScope`), com fallback seguro (tudo negado) |
| `hooks/usePdfAoLado.ts` | Estado do painel do PDF ao lado dos campos |
| `hooks/useAtividadeDaSessao.ts` | Dispara `/sessao/atividade` em interação real |
| `hooks/useMenuLateral.ts`, `hooks/useTheme.tsx` | Menu e tema claro/escuro |
| `lib/roles.ts` | `podeProcessarDocumentos(role)` — hoje `bolsista` e `coordenador` |
| `lib/camposProcesso.ts` | Separa um valor em termos de busca e descobre de qual PDF veio cada valor na visão "ver junto" |
| `lib/consolidado.ts` | Normalização da listagem consolidada |
| `lib/erros.ts` | Mensagem legível a partir do payload de erro da API |
| `lib/statusColors.ts`, `lib/utils.ts`, `lib/validators.ts` | Cores de status, `cn`, validações |
| `lib/tabela_modulos.json` | **Cópia manual** de `util/tabela_modulos.json` |

> O cálculo de módulo fiscal é derivado no cliente, e por isso a tabela do
> INCRA precisa estar nos dois lados. A fonte é a do backend:
> `cp ../../util/tabela_modulos.json app/lib/tabela_modulos.json`.

A permissão do frontend governa **o que aparece na tela**. A decisão que vale é
sempre a do backend: esconder um botão não protege um endpoint.

---

## 5. Visualizador de PDF

`app/ui/organisms/PdfViewer.tsx` é o componente mais complexo do repositório.
Três decisões explicam o tamanho dele.

### Importação sempre dinâmica

`pdfjs-dist` e o pacote de viewer só funcionam no navegador: usam recursos de
JS que o Node do SSR não tem. Um `import` estático no topo do arquivo seria
avaliado também no servidor e derrubaria o processo no boot. Por isso o import
acontece dentro de `useEffect` e handlers, e é cacheado em uma promise para não
recarregar o módulo a cada chamada.

### Framework de viewer do pdf.js, não canvas na mão

`pdfjs-dist/web/pdf_viewer.mjs` é o visualizador oficial do pdf.js — o mesmo
código que exibe PDF dentro do Firefox. Rolagem contínua, dimensionamento de
página, camada de texto posicionada e busca com destaque vêm prontos e
testados.

Esse módulo não importa o núcleo do pdf.js: ele lê tudo de
`globalThis.pdfjsLib`, convenção antiga de quando era carregado por `<script>`.
Sem setar essa global antes, o módulo quebra ao carregar (desestrutura de
`undefined`) e nada mais roda.

### Dois modos de exibição

`GET /processos/:id/pdf-info` responde, entre outros campos, `render_servidor`:

- `false` (padrão): pdf.js renderiza o arquivo.
- `true`: o visualizador monta o documento com as **imagens renderizadas no
  servidor**, página por página, pedindo DPI alto. É a saída para PDFs em
  formato bitonal de scanner que o pdf.js decodifica mal — causa real da
  corrupção de páginas observada em agosto de 2026.

Falha ao consultar `/pdf-info` não derruba a abertura: o backend devolve o
padrão (`render_servidor: false`) em vez de erro.

### Busca ancorada no campo

Clicar na lupa ao lado de um campo:

1. Converte o valor em **termos** (`termosDoValor`): lista vira um termo por
   item; texto com vários identificadores separados por vírgula ("624868,
   52695") vira um termo por valor. Endereço ("Rua X, 123"), nome ("SILVA,
   JOÃO") e decimal brasileiro ("37,8206", sem espaço após a vírgula)
   continuam sendo um termo só.
2. Usa a página de origem registrada em `campos._fontes` pela extração como
   âncora, e o identificador da instância quando o campo pertence a uma seção
   repetida (requerimento, CCIR).
3. Alinha o destaque encontrado à altura (`posicaoAlvoY`) em que o campo foi
   clicado, em vez de centralizar no painel. Em painel bem mais alto que a
   tela, centralizar nascia o destaque fora da área visível.
4. Quando o alinhamento não cabe (painel já no limite de rolagem), recolhe as
   seções acima e realinha o último trecho encontrado sem rebuscar.

### `Range` no proxy do PDF

`/processos/:id/pdf` repassa o header `Range` e o status de range da resposta.
Sem isso o pdf.js não consegue pedir só um pedaço do arquivo: além de esperar
o PDF inteiro para mostrar a primeira página, ele baixa o arquivo **duas
vezes** (tenta range, descobre que não há suporte, refaz o pedido inteiro) —
comprovado em captura de rede em produção, 867 kB baixados duas vezes
seguidas.

---

## 6. Editor de campos

`app/ui/organisms/CamposEditor.tsx`.

Ordem das seções fixada explicitamente em `ORDEM_SECOES`:
`capa`, `requerimento`, `ccir`, `quadro_areas`. Isso é necessário porque o
`jsonb` do Postgres do lado da extração não preserva a ordem das chaves — sem
fixar, `quadro_areas` (12 letras) aparecia antes de `requerimento_digital`
(20), por ordenação interna por tamanho de chave.

Duas chaves são **metadado** e não aparecem como campo editável:

- `_fontes` — página de origem de cada campo, usada pela busca.
- `quadro_areas_conferencia` — conferência derivada.

`ccir` e `ccirs` são tratadas como lista de objetos
(`CHAVES_LISTA_DE_OBJETOS`); qualquer seção que comece com `requerimento` é
normalizada para `requerimento`.

Edição de campo booleano: o `PUT .../campo` recebe sempre **string** (o modal é
texto livre). `services/campos_merge.normalizar_valor_editado` converte de
volta para booleano nos campos conhecidamente booleanos — hoje só
`capa.is_car`. Sem isso, abrir o lápis do `is_car` e confirmar gravava a string
`"true"` por cima do booleano; seis documentos do acervo ficaram assim antes do
conserto.

---

## 7. Configuração de build

### `react-router.config.ts`

| Opção | Valor | Motivo |
| --- | --- | --- |
| `basename` | `/gestao/` em produção, `/` fora | O sistema fica sob um caminho no domínio |
| `ssr` | `true` | Renderização no servidor |
| `routeDiscovery` | `{ mode: "initial" }` | No modo lazy, rota alcançada só por redirecionamento do servidor (como `/configurar-senha`) nunca é descoberta a tempo e a transição no cliente falha |
| `allowedActionOrigins` | `["www.integracar.agr.br", "integracar.agr.br", "null"]` em produção | A proteção nativa do React Router 7.12+ compara `Host` recebido pelo Node com o `Origin` do navegador. Atrás de proxy que não repasse `Host`, toda action era rejeitada. O nginx atende o domínio com e sem `www` |

`isProduction` considera `ENVIRONMENT === "production"` **ou**
`NODE_ENV === "production"`. Durante `npm run build`, `NODE_ENV` já é
`production`, então o `basename` fica `/gestao/` mesmo sem `ENVIRONMENT` —
enquanto o prefixo `/gestao/api` em `api.server.ts` depende **só** de
`ENVIRONMENT`. É a origem da armadilha de configuração descrita em
[deploy.md](deploy.md).

### `vite.config.ts`

`base` segue a mesma regra do `basename`. Plugins: `tailwindcss`,
`reactRouter`, `tsconfigPaths` (é o que faz o atalho `~/` apontar para `app/`).

### `Dockerfile`

Build multiestágio em quatro camadas sobre `node:20-alpine`: dependências de
desenvolvimento, dependências de produção, build, e a imagem final com apenas
`node_modules` de produção e `build/`. `CMD ["npm", "run", "start"]` sobe o
`react-router-serve` na porta 3000.

### `package.json`

```
npm run dev         react-router dev        porta 5173
npm run build       react-router build      gera build/client e build/server
npm run start       react-router-serve      serve o build, porta 3000
npm run typecheck   react-router typegen && tsc
```

Não há script de lint nem de teste no frontend. A verificação disponível é
`npm run typecheck`, e ela precisa rodar antes de abrir PR: o `typegen` gera os
tipos das rotas, e sem ele o `tsc` reclama de coisas que não são erro real.

### `overrides`

`"qs": "^6.16.0"` está fixado em `overrides` para forçar a versão corrigida em
dependência transitiva.
