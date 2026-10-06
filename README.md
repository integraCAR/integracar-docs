# integracar-docs

Documentação técnica do projeto **IntegraCAR** — apoio à análise e à gestão dos
processos do Cadastro Ambiental Rural (CAR) e do Simlam no Estado do Espírito
Santo.

Este repositório guarda a documentação. O código fica em repositórios
separados, um por sistema.

---

## O projeto em uma frase

===

---

## Mapa dos repositórios

Cada nome abaixo aponta para a **documentação do sistema neste repositório**,
não para o código.

| Repositório | Papel | Onde roda | Documentação |
| --- | --- | --- | --- |
| `integracar-gestao` | Sistema de gestão de processos CAR/Simlam: API, frontend, perfis de usuário, revisão de campos | VPS | [`gestao/`](gestao/README.md) |
| `integracar-backend` | API de extração de campos (dona do schema Postgres e da fila de jobs) | Workstation | No próprio repositório (`CLAUDE.md`) |
| `integracar-ocr` | Worker do pipeline de OCR: PDF em imagens, modelo de visão, extração de campos | Workstation | No próprio repositório (`CLAUDE.md`) |
| `PDF-Pesquisavel` | Serviço que adiciona camada de texto aos PDFs escaneados (OCRmyPDF) | Workstation | Resumida em [`gestao/integracoes.md`](gestao/integracoes.md) |
| `integracar-infra` | Orquestração `docker compose` de toda a stack da workstation | Workstation | No próprio repositório (`CLAUDE.md`) |
| `integracar-frontend` | Frontend antigo da API de extração. **Descontinuado**, não usar | — | — |

As duas metades do sistema de extração (`integracar-backend` e
`integracar-ocr`) são mantidas por outra equipe e têm documentação própria
dentro dos respectivos repositórios.

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

---

## Projeto

Coordenação: IFES Campus Cachoeiro de Itapemirim
Vigência: 2024 – 2027
