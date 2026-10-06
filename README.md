# integracar-docs

Documentação técnica do projeto **IntegraCAR** — apoio à análise e à gestão dos
processos do Cadastro Ambiental Rural (CAR) e do Simlam no Estado do Espírito
Santo.

Este repositório guarda a documentação. O código fica em repositórios
separados, um por sistema. Ele é **público** de propósito: a maior parte dos
repositórios de código é privada, e é aqui que quem não é membro da
organização consegue entender o que cada um faz.

---

## Mapa dos repositórios

A organização [integraCAR](https://github.com/integraCAR) tem 12 repositórios.
Eles formam cinco sistemas, mais dois repositórios de apoio.

| Sistema | Repositório | Visibilidade | Papel | Onde roda | Documentação |
| --- | --- | --- | --- | --- | --- |
| Gestão | `integracar-gestao` | Privado | API, interface e banco do sistema de gestão de processos CAR/Simlam | VPS | [`gestao/`](gestao/README.md) |
| Extração | `integracar-backend-extrator` | Privado | API FastAPI de extração: fila, documentos, histórico de versões, schema do banco | Workstation | [`extrator/api.md`](extrator/api.md) |
| Extração | `integracar-ocr-extrator` | Privado | Worker de OCR (GLM-OCR via Ollama) e extração de campos (regex + LLM) | Workstation | [`extrator/worker-ocr.md`](extrator/worker-ocr.md) |
| Extração | `integracar-infra-extrator` | Privado | `docker compose` da stack de extração, gateway nginx, observabilidade e backup | Workstation | [`extrator/infra.md`](extrator/infra.md) |
| Extração | `PDF-Pesquisavel` | Privado | Serviço que gera a camada de texto dos PDFs escaneados (OCRmyPDF) | Workstation | [`extrator/pdf-pesquisavel.md`](extrator/pdf-pesquisavel.md) |
| Dashboard | `integracar-dashboard` | Privado | Painel Streamlit com os processos IntegraCAR consolidados a partir da API do E-Docs | VPS | [`dashboard/`](dashboard/README.md) |
| LULC | `integracar-lulc-builder` | Público | Pipeline que baixa imagens de satélite e mapas de uso do solo do GeoBases | Local | [`lulc/`](lulc/README.md) |
| LULC | `IntegraCAR-LULC-10K` | Público | Página do dataset IntegraCAR-LULC-10K (EN/PT) | Site estático | [`lulc/`](lulc/README.md#integracar-lulc-10k-landing-page) |
| Web | `integracar-web` | Público | Site institucional do projeto (planejado) | VPS | [`web/`](web/README.md) |
| Web | `LANDING-PAGE-INTEGRACAR-ES` | Privado | Versão anterior da página do dataset | Site estático | [`web/`](web/README.md#landing-page-integracar-es) |
| Apoio | `integracar-docs` | Público | Esta documentação | — | este arquivo |
| Apoio | `.github` | Público | Página de apresentação da organização no GitHub (`profile/README.md`) | — | [`.github`](https://github.com/integraCAR/.github) |

### Como os sistemas se conectam

```
                       VPS
  +-------------------------------------------------------+
  |  integracar-gestao  (API FastAPI + React + MySQL)     |
  |  integracar-dashboard (Streamlit + SQLite)  <-- E-Docs|
  |  integracar-web (site institucional, planejado)       |
  +--------------------------|----------------------------+
                             | HTTPS, X-Service-Token
                             v
                       WORKSTATION (GPU)
  +-------------------------------------------------------+
  |  integracar-infra-extrator: gateway nginx + compose   |
  |    /api/          -> integracar-backend-extrator      |
  |                       <- fila Postgres ->             |
  |                      integracar-ocr-extrator (worker) |
  |    /pesquisavel/  -> PDF-Pesquisavel (OCRmyPDF)       |
  |  Ollama no host (glm-ocr, qwen2.5)                    |
  +-------------------------------------------------------+

  Pesquisa (independente do fluxo acima)
    integracar-lulc-builder -> dataset IntegraCAR-LULC-10K (Hugging Face)
                               └ página: IntegraCAR-LULC-10K
```

---

## Conteúdo desta documentação

### Sistema de Gestão (`integracar-gestao`)

| Documento | Para que serve |
| --- | --- |
| [gestao/README.md](gestao/README.md) | Visão geral do sistema: o que faz, para quem, como as peças se encaixam. **Comece aqui.** |
| [gestao/arquitetura.md](gestao/arquitetura.md) | Arquitetura, fluxo de uma requisição, decisões de projeto e seus motivos |
| [gestao/backend.md](gestao/backend.md) | API FastAPI: camadas, 83 endpoints, autenticação, middlewares |
| [gestao/frontend.md](gestao/frontend.md) | React Router 7 em modo framework (SSR), rotas, telas, visualizador de PDF |
| [gestao/banco-de-dados.md](gestao/banco-de-dados.md) | Dicionário de dados do MySQL: tabelas, colunas, relacionamentos, migrações |
| [gestao/integracoes.md](gestao/integracoes.md) | Contratos com a API de extração e com o serviço de PDF pesquisável |
| [gestao/seguranca.md](gestao/seguranca.md) | Sessão, CSRF, rate limiting, headers, autorização por perfil |
| [gestao/desenvolvimento.md](gestao/desenvolvimento.md) | Como subir o ambiente local e trabalhar no código |
| [gestao/deploy.md](gestao/deploy.md) | Deploy em produção: Docker, nginx, HTTPS, backup, rollback |
| [gestao/testes.md](gestao/testes.md) | Suíte de 485 testes: estrutura, como rodar, cobertura real |

### Sistema de Extração (4 repositórios)

| Documento | Para que serve |
| --- | --- |
| [extrator/README.md](extrator/README.md) | Visão geral: os quatro repositórios, como conversam, caminho de um PDF. **Comece aqui.** |
| [extrator/api.md](extrator/api.md) | `integracar-backend-extrator`: rotas internas e públicas (`/v1`), autenticação, tabelas, migrações, testes |
| [extrator/worker-ocr.md](extrator/worker-ocr.md) | `integracar-ocr-extrator`: pipeline de OCR, extratores, releitura dirigida, campos extraídos |
| [extrator/infra.md](extrator/infra.md) | `integracar-infra-extrator`: serviços do compose, gateway, observabilidade, alertas, backup |
| [extrator/pdf-pesquisavel.md](extrator/pdf-pesquisavel.md) | `PDF-Pesquisavel`: API, estratégia de OCR, configuração, operação |

### Dashboard (`integracar-dashboard`)

| Documento | Para que serve |
| --- | --- |
| [dashboard/README.md](dashboard/README.md) | Coleta no E-Docs, regra de classificação IntegraCAR, banco SQLite, telas, operação no servidor |

### LULC — uso e cobertura do solo (em inglês)

| Documento | Para que serve |
| --- | --- |
| [lulc/README.md](lulc/README.md) | `integracar-lulc-builder`, dataset IntegraCAR-LULC-10K e a página do dataset |

### Web

| Documento | Para que serve |
| --- | --- |
| [web/README.md](web/README.md) | `integracar-web` e `LANDING-PAGE-INTEGRACAR-ES` |

---

## Nomes antigos dos repositórios

Os repositórios do sistema de extração mudaram de nome. Código, comentários e
documentação antiga ainda usam os nomes de antes. A tabela abaixo é a
referência.

| Nome atual (GitHub) | Nomes antigos encontrados no código |
| --- | --- |
| `integracar-backend-extrator` | `integracar-backend`, `integracar-backend-ocr` |
| `integracar-ocr-extrator` | `integracar-ocr`, `integracar-solucao-ocr` |
| `integracar-infra-extrator` | `integracar-infra`, `integracar-infra-ocr` |
| — (fora de uso, não está na organização) | `integracar-frontend`, `integracar-frontend-ocr` |
| `integracar-dashboard` | `Dashboard-IntegraCAR` (nome da pasta no servidor: `/opt/Dashboard-IntegraCAR`) |
| organização `integraCAR` | `integracar-cachoeiro` |

---

## Como manter esta documentação

- **Mexeu em endpoint, tabela ou integração? Atualize este repositório no
  mesmo PR.** A documentação mora aqui porque descreve a integração entre
  sistemas mantidos por equipes diferentes; uma cópia por repositório já
  produziu versões divergentes.
- O README de cada repositório de código é curto e aponta para cá.
- Português do Brasil, sem emojis. A exceção é a pasta `lulc/`, que segue o
  idioma do código e do dataset (inglês).
- **Nunca coloque valor de segredo** (token, senha, client secret) em nenhum
  documento. Cite o nome da variável, nunca o valor.

---

## Projeto

Coordenação: IFES Campus Cachoeiro de Itapemirim
Parceiros: IDAF, FAPES, SEGER, Inova IFES
Vigência: 2024 – 2027
Site: [integracar.agr.br](https://integracar.agr.br)
