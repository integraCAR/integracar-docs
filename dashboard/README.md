# Dashboard IntegraCAR (`integracar-dashboard`)

Painel em Streamlit que identifica, consolida e mostra os processos do
IntegraCAR que tramitam no **E-Docs**, o sistema de processos eletrônicos do
Governo do Espírito Santo. Serve para acompanhar quantos processos CAR e Simlam
estão com o projeto, em que unidade, com qual campus e com qual bolsista.

---

## Sumário

1. [O problema](#1-o-problema)
2. [Como funciona](#2-como-funciona)
3. [Regra de classificação IntegraCAR](#3-regra-de-classificação-integracar)
4. [Banco de dados](#4-banco-de-dados)
5. [Telas](#5-telas)
6. [Estrutura do repositório](#6-estrutura-do-repositório)
7. [Configuração e credenciais](#7-configuração-e-credenciais)
8. [Rodar e implantar](#8-rodar-e-implantar)
9. [Limitações conhecidas](#9-limitações-conhecidas)

---

## 1. O problema

Os processos do IntegraCAR tramitam no E-Docs junto com milhares de outros
processos do IDAF. O E-Docs não tem:

- filtro nativo por IntegraCAR;
- base consolidada específica do projeto;
- relatórios analíticos automatizados.

O dashboard resolve isso montando uma base SQLite própria, classificando cada
processo e mostrando os números num painel.

---

## 2. Como funciona

```
API do E-Docs (Acesso Cidadão, client_credentials)
   |  paginated-search, local-custodia, autuador, atos
   v
scripts de coleta (cron)  ---->  SQLite (dados_integracar.db)  <----  dashboard.py (Streamlit)
```

1. **Token.** Os scripts pedem um token em
   `https://acessocidadao.es.gov.br/is/connect/token`, com `grant_type=client_credentials`
   e escopo `api-sigades-consultar`.
2. **Busca paginada.** `POST https://api.e-docs.es.gov.br/v2/processos/paginated-search`,
   filtrando pela organização de autuação IDAF e por data de autuação (a partir
   de 2025-01-01).
3. **Custódia.** Para cada processo, `GET /v2/processos/{id}/local-custodia`
   diz com quem o processo está agora.
4. **Classificação.** Aplica a regra da seção 3 e grava em
   `detalhes_processos.eh_integracar`.
5. **Enriquecimento.** Tipo de processo (CAR, Simlam, IUF, outro), autuador,
   número de atos, data do último ato.
6. **Painel.** O Streamlit lê **só o SQLite**. Ele não chama a API do E-Docs.

---

## 3. Regra de classificação IntegraCAR

Um processo é IntegraCAR quando a custódia atual é um **grupo** cujo nome
contém `integracar`:

```python
eh_integracar = 1 if (tipo == "Grupo" and "integracar" in nome.lower()) else 0
```

Os grupos são um por campus do IFES, no formato
`ifes campus <campus> - integracar` (Alegre, Barra de São Francisco, Cachoeiro
de Itapemirim, Colatina, Ibatiba, Itapina, Linhares, Montanha, Nova Venécia,
Piúma, Santa Teresa, Vitória).

O painel também considera processos despachados diretamente para pessoas do
projeto, quando o nome da custódia segue o padrão
`NOME - Estudante bolsista - Projeto IntegraCAR - IFES - <Campus> - ...` ou
`NOME - Orientador ... IntegraCAR`.

Cuidados:

- Nem todo processo CAR é IntegraCAR, e nem todo processo Simlam está fora do
  IntegraCAR. **A classificação depende só da custódia.**
- A atribuição de um processo de um grupo para uma pessoa, dentro do E-Docs,
  **não gera ato formal e não aparece na API**. A API mostra a custódia como do
  grupo, mesmo quando a interface web mostra o processo na pasta de um
  bolsista. O endpoint que mostraria isso (`/v2/usuario/caixas-processo`) exige
  token de usuário (`authorization_code`) e devolve `403` com o token de
  sistema. A investigação completa está em
  `docs/rastreabilidade_custodia_edocs.md`, no repositório.

---

## 4. Banco de dados

SQLite em `src/dashboard/dados_integracar.db`, com `PRAGMA journal_mode=WAL`
para permitir leitura e escrita simultâneas. Tabelas criadas por
`database_manager.init_db()` e pelos scripts de coleta:

| Tabela | Conteúdo |
| --- | --- |
| `detalhes_processos` | Última fotografia de cada processo: `id_processo`, abertura, assunto, status, local atual, interessado, `data_autuacao`, `data_ultimo_ato`, `total_atos`, `local_autuacao_sigla`, `tipo_processo`, `custodia_*`, `eh_integracar` |
| `historico_processos` | Série de quantidade de processos por local de origem, por data de coleta |
| `autuadores` | Quantidade e percentual de processos por autuador, por data de coleta |
| `bolsistas_edocs` | Bolsistas identificados pelos atos no E-Docs: nome, cargo, campus, ativo |
| `integracar_checkpoint` | Página em que a carga completa parou, para retomar |

O arquivo `.db` não é versionado (`*.db` no `.gitignore`).

---

## 5. Telas

`src/dashboard/dashboard.py`, com a paleta da logo do projeto. O menu atual
tem três abas:

| Aba | Conteúdo |
| --- | --- |
| Visão Geral | Evolução mensal dos processos CAR e contribuição da equipe IFES, incluindo processos autuados por quem não está na lista de bolsistas |
| Processos por Unidade | Produção por campus (só processos na custódia dos grupos IFES, para bater com os números do E-Docs) e tabela detalhada |
| Relatório por Autuador | Processos da equipe IFES por bolsista, com filtro por lotação |

O arquivo também tem funções de abas que não estão no menu hoje (`tabConsulta`,
`tabMapaES`, `tabIfes`), incluindo um mapa por município com
`geojs-32-mun.json`.

A lista de bolsistas vem da tabela `bolsistas_edocs`. Se ela estiver vazia, o
painel usa `bolsistas_integracar.csv` como reserva.

---

## 6. Estrutura do repositório

```
src/
  index.html                      Página de apresentação do projeto (estática)
  logo.png
  migrar_dados.py                 Migração antiga de JSON para o SQLite
  dashboard/
    dashboard.py                  Aplicação Streamlit
    database_manager.py           Conexão SQLite, criação das tabelas, salvar/carregar
    db_adapter.py                 Montagem de filtros e consultas para o painel
    gerarToken.py                 Token do Acesso Cidadão (lê st.secrets)
    consultarAPI.py               paginated-search do E-Docs
    consultarAPIquantidade.py     Contagens (em andamento / encerrados)
    consultarAPIdetalhesProcessos.py
    consultarLocalDeOrigem.py
    cron_update_db.py             Atualização periódica: busca, custódia, classificação
    carrega_base_completa.py      Carga completa paginada, com checkpoint
    enriquecer_integracar.py      Enriquecimento em lotes de páginas
    atualizar_bolsistas_edocs.py  Extrai bolsistas dos atos do E-Docs
    dadosAutuadores.py            Estatísticas por autuador
    migrar_jsons_servidor.py      Migração dos JSON antigos do servidor
    locais.json                   Unidades do IDAF
    geojs-32-mun.json             Malha de municípios do ES
    bolsistas_integracar.csv      Lista de bolsistas (reserva)
docs/
  rastreabilidade_custodia_edocs.md
tests/                            Vazio por enquanto
pyproject.toml, poetry.lock       Dependências (Poetry)
requirements.txt                  Exportado do Poetry
```

---

## 7. Configuração e credenciais

As credenciais do Acesso Cidadão ficam num `secrets.toml` do Streamlit, **fora
do git**:

```toml
# .streamlit/secrets.toml
SECRET = "client_id:client_secret"
```

O painel lê com `st.secrets["SECRET"]`. Os scripts de coleta procuram o
arquivo, nesta ordem: `INTEGRACAR_SECRETS` (variável de ambiente; padrão
`/opt/Dashboard-IntegraCAR/.streamlit/secrets.toml`), `.streamlit/` na raiz do
projeto, `.streamlit/` ao lado do script e `~/.streamlit/`.

Outras variáveis:

| Variável | Padrão | Uso |
| --- | --- | --- |
| `INTEGRACAR_SECRETS` | `/opt/Dashboard-IntegraCAR/.streamlit/secrets.toml` | Caminho do `secrets.toml` |
| `INTEGRACAR_MAX_CUSTODIA` | `200` | Máximo de consultas de custódia por execução do cron |

**Nunca escreva o client secret em arquivo versionado**, nem em anotações de
terminal. Se isso acontecer, peça um novo secret ao Acesso Cidadão.

---

## 8. Rodar e implantar

Requer Python 3.13.

```bash
poetry install                      # ou: pip install -r requirements.txt
mkdir -p .streamlit && $EDITOR .streamlit/secrets.toml
streamlit run src/dashboard/dashboard.py
```

Carga inicial e atualização (rodar de dentro de `src/dashboard/`):

```bash
python carrega_base_completa.py --pages-per-run 50 --page-size 200 --from-date 2025-01-01
python cron_update_db.py
python atualizar_bolsistas_edocs.py
```

`carrega_base_completa.py` aceita `--start-page`, `--end-page`,
`--pages-per-run`, `--page-size`, `--from-date`, `--to-date` e `--sleep`. Sem
`--start-page`, retoma do checkpoint.

**No servidor**, o projeto fica em `/opt/Dashboard-IntegraCAR`, com virtualenv
em `/opt/Dashboard-IntegraCAR/venv`, e é servido no mesmo servidor do
`integracar-gestao` e do `integracar-web`, com roteamento pelo nginx. Vários
scripts têm esse caminho fixo no código (`carrega_base_completa.py`,
`enriquecer_integracar.py`). Exemplo de agendamento:

```cron
0 2 * * * cd /opt/Dashboard-IntegraCAR/src/dashboard && \
          /opt/Dashboard-IntegraCAR/venv/bin/python atualizar_bolsistas_edocs.py \
          >> /var/log/bolsistas_edocs.log 2>&1
```

---

## 9. Limitações conhecidas

- **Custódia individual invisível na API** (seção 3).
- **Caminhos fixos.** Alguns scripts só rodam com o projeto em
  `/opt/Dashboard-IntegraCAR`.
- **SQL montado com f-string** em `db_adapter.py`, a partir dos filtros da
  tela. Hoje os valores vêm de controles do próprio painel, mas o certo é
  passar para consultas parametrizadas.
- **Sem testes.** `tests/` só tem `__init__.py`.
- **Dados pessoais no repositório.** Os CSVs de bolsistas têm nomes de pessoas.
  O repositório é privado, mas vale avaliar se esses arquivos precisam estar
  versionados.
- **Arquivos auxiliares do SQLite** (`dados_integracar.db-shm`, `-wal`) estão
  versionados; podem entrar no `.gitignore`.
