# Banco de dados — MySQL 8.0

Dicionário de dados do `integracar-gestao`: nove tabelas, o que cada coluna
guarda, como as tabelas se relacionam e como o schema evolui.

Pré-requisito: [README.md](README.md).

---

## 1. Antes de tudo: dois bancos independentes

O IntegraCAR tem **dois bancos que não se falam**:

| Banco | Dono | Guarda |
| --- | --- | --- |
| MySQL | `integracar-gestao` (este) | Usuários, pastas, vínculos com documentos, atividades, feedback |
| Postgres | `integracar-backend` | Fila de jobs e os **campos extraídos** dos PDFs |

Não há migração nem schema comum. O elo entre os dois é a coluna
`Ocr_Documento.documento_id`, que é um id **do outro banco**. Nenhuma `FOREIGN
KEY` protege esse elo: é referência lógica entre sistemas.

Consequência prática: o conteúdo extraído de um PDF **não está no MySQL**.
Toda consulta de campo passa pela API de extração, via `services/ocr_client.py`.
O MySQL guarda quem é dono do quê, onde está guardado, e o que a pessoa achou
do resultado.

---

## 2. Configuração da conexão

| Aspecto | Valor | Onde |
| --- | --- | --- |
| Charset e collation | `utf8mb4` / `utf8mb4_unicode_ci` | `conexao_db.py`, e em cada `CREATE TABLE` |
| Pool | `MySQLConnectionPool`, `pool_size=20`, `pool_reset_session=True` | `conexao_db.get_pool()` |
| Fuso | `SET time_zone = '-03:00'` **a cada checkout** | `conexao_db.criar_conexao()` |
| Cursores | `dictionary=True` — linhas vêm como `dict` | `conexao_db.executar_query()` |
| Engine | InnoDB em todas as tabelas | `data/sql/*` |

### Por que o fuso é reaplicado a cada conexão

O pool roda `RESET_CONNECTION` no servidor a cada `close()` por causa de
`pool_reset_session=True`, e isso **desfaz** qualquer `time_zone` aplicado só
no connect inicial. Sem reaplicar no checkout, apenas o primeiro uso de cada
uma das 20 conexões ficaria correto; todas as reutilizações — a maior parte do
tráfego — voltariam ao fuso padrão do servidor, normalmente UTC, e as colunas
`criado_em` / `atualizado_em` sairiam três horas adiantadas.

O valor é um **offset fixo**, não `America/Sao_Paulo`: não depende da tabela de
fusos do MySQL estar carregada, e o Brasil não tem mais horário de verão.

---

## 3. Visão geral dos relacionamentos

```
   Campus
     │ 1
     │
     │ N
   Usuario ──┐ cod_orientador (auto-referência)
     │ 1     └──────┐
     │              │
     ├──N── Atividade ── cod_campus ──> Campus
     │              └── orientador_bolsista ──> Usuario
     │
     ├──N── Pasta ── cod_pasta_pai (auto-referência, ON DELETE CASCADE)
     │        │ 1
     │        │ N
     ├──N── Ocr_Documento                    cod_pasta ON DELETE SET NULL
     │        │ 1                            documento_id ──> Postgres
     │        ├──N── Feedback_Campo           ON DELETE CASCADE
     │        └──1── Anotacao_Processo        ON DELETE CASCADE, PK = FK
     │
     ├──N── Feedback_Campo_Pasta ── cod_pasta ──> Pasta (ON DELETE CASCADE)
     │
     └──N── Solicitacao_Desvinculo  (origem e destino, dois FKs para Usuario)
```

---

## 4. Tabelas

### 4.1 `Campus`

Campi do IFES participantes. Tabela de apoio, carregada pelos seeders.

| Coluna | Tipo | Notas |
| --- | --- | --- |
| `cod_campus` | `INT AUTO_INCREMENT` | PK |
| `nome_campus` | `VARCHAR(255) NOT NULL` | |

### 4.2 `Usuario`

Todas as pessoas do sistema, de todos os perfis.

| Coluna | Tipo | Notas |
| --- | --- | --- |
| `cod_usuario` | `INT AUTO_INCREMENT` | PK |
| `cod_campus` | `INTEGER` | FK para `Campus`. Coordenador pode não ter campus |
| `cod_orientador` | `INTEGER NULL` | FK para `Usuario` — auto-referência: o orientador de um bolsista |
| `nome_usuario` | `VARCHAR(255) NOT NULL` | |
| `email_usuario` | `VARCHAR(255) NOT NULL` | Identificador de login. **Sem `UNIQUE`** |
| `senha_usuario` | `VARCHAR(255) NOT NULL` | Hash bcrypt (12 rounds). Contas importadas por planilha ficam em texto puro, por decisão de negócio |
| `cpf_usuario` | `VARCHAR(11) NOT NULL` | Só dígitos |
| `role_usuario` | `VARCHAR(100) NULL` | `bolsista`, `coordenador`, `orientador`, `consultor`. **Sem constraint**: é texto livre |
| `organizacao_usuario` | `VARCHAR(255) NULL` | |
| `token_reset_senha` | `VARCHAR(255) NULL` | Token de recuperação de senha |
| `token_reset_expiracao` | `DATETIME NULL` | Validade padrão de 24 h |
| `primeiro_acesso` | `BOOLEAN DEFAULT TRUE` | Lido pelo frontend; o bloqueio no backend está desativado |

> **Duas ausências a considerar.** `email_usuario` sem `UNIQUE` permite duas
> contas com o mesmo e-mail, e o login resolveria pela primeira encontrada.
> `role_usuario` sem `ENUM` nem `CHECK` permite gravar um perfil que nenhuma
> rota reconhece — o usuário entra e não vê nada.

### 4.3 `Pasta`

Árvore de organização do trabalho do bolsista.

| Coluna | Tipo | Notas |
| --- | --- | --- |
| `cod_pasta` | `INT AUTO_INCREMENT` | PK |
| `nome` | `VARCHAR(255) NOT NULL` | |
| `tipo` | `VARCHAR(20) NOT NULL` | `categoria` ou `processo`. **Fixado na criação**, nunca inferido do conteúdo |
| `cod_pasta_pai` | `INT NULL` | FK para `Pasta`, `ON DELETE CASCADE`. `NULL` = raiz |
| `cod_usuario` | `INT NOT NULL` | FK para `Usuario`. Dono |
| `padrao` | `BOOLEAN NOT NULL DEFAULT FALSE` | Marca a categoria `Processos Avulsos`, uma por usuário, criada automaticamente |
| `criado_em` | `DATETIME` | `CURRENT_TIMESTAMP` |
| `atualizado_em` | `DATETIME` | `ON UPDATE CURRENT_TIMESTAMP` |

Semântica dos dois tipos:

- **categoria** — organiza. Contém outras pastas.
- **processo** — unidade de trabalho. Contém os PDFs de um processo de CAR.

`ON DELETE CASCADE` no auto-relacionamento significa que apagar uma categoria
apaga toda a subárvore. Os PDFs não são apagados: `Ocr_Documento.cod_pasta` é
`ON DELETE SET NULL`.

### 4.4 `Ocr_Documento`

A tabela central. Cada linha é um **vínculo**: "este usuário enviou este PDF, e
o PDF é este documento do outro lado".

| Coluna | Tipo | Notas |
| --- | --- | --- |
| `cod_ocr_documento` | `INT AUTO_INCREMENT` | PK. Id local, o único que o frontend conhece |
| `cod_usuario` | `INT NOT NULL` | FK para `Usuario`. Quem enviou |
| `job_id` | `INT NULL` | Id do job na API de extração. **Chave do callback.** `UNIQUE KEY uq_job` |
| `documento_id` | `INT NOT NULL` | Id do documento na API de extração. **Chave dos campos.** Compartilhado entre vínculos quando a API deduplica por hash |
| `nome_pdf` | `VARCHAR(255)` | Nome do arquivo |
| `status` | `VARCHAR(20) NOT NULL DEFAULT 'na_fila'` | `na_fila`, `processando`, `concluido`, `erro` |
| `error_msg` | `TEXT NULL` | Preenchido só em `erro` |
| `cod_pasta` | `INT NULL` | FK para `Pasta`, `ON DELETE SET NULL`. `NULL` = solto |
| `vezes_reprocessado` | `INT NOT NULL DEFAULT 0` | Quantas vezes pediram reprocessamento |
| `searchable_id` | `VARCHAR(64) NULL` | Id no serviço de PDF pesquisável. `NULL` = nunca enviado (recurso desligado ou upload anterior a ele) |
| `searchable_status` | `VARCHAR(20) NULL` | `pending`, `done`, `erro` |
| `criado_em` | `DATETIME` | |
| `atualizado_em` | `DATETIME` | `ON UPDATE CURRENT_TIMESTAMP` |

Os dois identificadores externos têm papéis diferentes, e trocá-los é um erro
fácil de cometer:

- **`job_id`** identifica *o processamento*. É por ele que chega o callback de
  conclusão. É `UNIQUE`, e é `NULL` quando a API respondeu que o documento já
  existia (não houve job novo).
- **`documento_id`** identifica *o documento*. É por ele que se buscam os
  campos, o PDF e as páginas. **Não é único nesta tabela**: dois bolsistas que
  sobem o mesmo arquivo têm dois vínculos com o mesmo `documento_id`. É o que
  torna necessárias as solicitações de desvínculo e o reaproveitamento de PDF
  pesquisável entre "irmãos".

`searchable_id` e `searchable_status` descrevem **apenas visualização**: não
alteram `documento_id`, não mudam hash, não substituem o arquivo de origem e
não interferem na extração de campos.

### 4.5 `Feedback_Campo`

Histórico de avaliações de campo de **um documento**. Append-only.

| Coluna | Tipo | Notas |
| --- | --- | --- |
| `cod_feedback_campo` | `INT AUTO_INCREMENT` | PK |
| `cod_ocr_documento` | `INT NOT NULL` | FK, `ON DELETE CASCADE` |
| `cod_usuario` | `INT NOT NULL` | FK. Quem avaliou — pode ser o coordenador, não só o dono do PDF |
| `campo` | `VARCHAR(120) NOT NULL` | Caminho exato: `numero`, `capa.numero`, `ccir_2_codigo` |
| `campo_agrupado` | `VARCHAR(120) NOT NULL` | Normalizado para a métrica: `ccir_*_codigo` |
| `correto` | `BOOLEAN NULL` | `NULL` = voto desfeito |
| `valor_avaliado` | `TEXT NULL` | Valor no momento do voto. Mudando o campo depois, o voto fica desatualizado |
| `origem` | `VARCHAR(20) NOT NULL DEFAULT 'voto'` | `voto` (polegar) ou `correcao` (gerado pela edição) |
| `criado_em`, `atualizado_em` | `DATETIME` | |

Índice: `ix_feedback_campo_documento_campo (cod_ocr_documento, campo, criado_em)`.

**Sem `UNIQUE`, de propósito.** Até setembro de 2026 a linha era sobrescrita, e
isso deixava o painel do coordenador enganoso: marcar um campo como errado,
corrigi-lo e votar certo de novo **apagava o "errado" para sempre** — e é
justamente esse sinal que a métrica de qualidade precisa. Hoje as duas
avaliações convivem: o campo continua contando como já tendo errado uma vez,
mesmo depois de corrigido.

O **veredito atual** de um campo é a linha mais recente:

```sql
SELECT ... FROM (
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY campo ORDER BY criado_em DESC, cod_feedback_campo DESC
    ) AS posicao
    FROM Feedback_Campo WHERE cod_ocr_documento = %s
) t WHERE posicao = 1 AND correto IS NOT NULL
```

`origem = 'correcao'` é gravada automaticamente por
`put_editar_campo_controller`: a correção já confessa que o valor anterior
estava errado, sem depender de alguém lembrar de votar "não confere" antes de
editar.

### 4.6 `Feedback_Campo_Pasta`

Mesma mecânica, mas para a visão **combinada** de um processo (pasta tipo
`processo`), onde os campos vêm de vários PDFs mesclados.

| Coluna | Tipo | Notas |
| --- | --- | --- |
| `cod_feedback_campo_pasta` | `INT AUTO_INCREMENT` | PK |
| `cod_pasta` | `INT NOT NULL` | FK para `Pasta`, `ON DELETE CASCADE` |
| `cod_usuario` | `INT NOT NULL` | FK |
| `campo` | `VARCHAR(120) NOT NULL` | Caminho no dado combinado: `capa.interessado`, `ccir.0.numero_ccir` |
| `campo_agrupado` | `VARCHAR(120) NOT NULL` | Normalizado |
| `correto` | `BOOLEAN NULL` | |
| `valor_avaliado` | `TEXT NULL` | Valor **combinado** no momento do voto |
| `origem` | `VARCHAR(20) NOT NULL DEFAULT 'voto'` | |
| `criado_em`, `atualizado_em` | `DATETIME` | |

Índice: `ix_feedback_campo_pasta_campo (cod_pasta, campo, criado_em)`.

### 4.7 `Anotacao_Processo`

Texto livre por documento. **Não é histórico**: é um campo compartilhado que
qualquer bolsista vinculado reescreve.

| Coluna | Tipo | Notas |
| --- | --- | --- |
| `cod_ocr_documento` | `INT` | **PK e FK ao mesmo tempo**, `ON DELETE CASCADE` |
| `cod_usuario` | `INT NOT NULL` | FK. Quem editou por último |
| `texto` | `TEXT NOT NULL` | |
| `atualizado_em` | `DATETIME` | `ON UPDATE CURRENT_TIMESTAMP` |

PK igual à FK é o que faz o upsert resolver sozinho:

```sql
INSERT INTO Anotacao_Processo (cod_ocr_documento, cod_usuario, texto)
VALUES (%s, %s, %s)
ON DUPLICATE KEY UPDATE cod_usuario = VALUES(cod_usuario), texto = VALUES(texto);
```

Apagar a anotação (texto em branco) **remove a linha**, em vez de guardar linha
vazia — senão o rodapé "editado por Fulano em ..." continuaria aparecendo sob
um campo sem texto nenhum.

### 4.8 `Solicitacao_Desvinculo`

Aviso enviado aos outros bolsistas vinculados ao mesmo documento quando alguém
exclui o próprio vínculo.

| Coluna | Tipo | Notas |
| --- | --- | --- |
| `cod_solicitacao` | `INT AUTO_INCREMENT` | PK |
| `documento_id` | `INT NOT NULL` | Documento na API de extração. **Sem FK** (é id de outro banco) |
| `nome_pdf` | `VARCHAR(255)` | Guardado aqui porque o vínculo de origem já foi apagado |
| `cod_usuario_origem` | `INT NOT NULL` | FK. Quem excluiu |
| `nome_usuario_origem` | `VARCHAR(255)` | Nome no momento do pedido |
| `cod_usuario_destino` | `INT NOT NULL` | FK. Quem está sendo avisado |
| `status` | `VARCHAR(20) NOT NULL DEFAULT 'pendente'` | `pendente`, `aceita`, `recusada` |
| `criado_em` | `DATETIME` | |
| `respondido_em` | `DATETIME NULL` | |

Regra de negócio: no máximo **uma pendente por par (`documento_id`,
`cod_usuario_destino`)**, conferida em código antes do insert
(`OBTER_PENDENTE_POR_DOCUMENTO_E_DESTINO`). Sem isso, exclusões repetidas do
mesmo documento empilhariam avisos idênticos.

### 4.9 `Atividade`

Registro de atividades para acompanhamento.

| Coluna | Tipo | Notas |
| --- | --- | --- |
| `cod_atividade` | `INT AUTO_INCREMENT` | PK |
| `cod_usuario` | `INTEGER NOT NULL` | FK. Autor |
| `cod_campus` | `INTEGER NULL` | FK para `Campus` |
| `orientador_bolsista` | `INTEGER NULL` | FK para `Usuario` — o orientador vinculado |
| `data_hora_inicio_atividade` | `TIMESTAMP NOT NULL` | |
| `nome_atividade` | `VARCHAR(255) NOT NULL` | |
| `descricao_atividade` | `TEXT NOT NULL` | |

> **Atenção ao nome.** A tabela é `Atividade`, singular. O arquivo é
> `data/sql/atividades_sql.py` e o repo `data/repo/atividades_repo.py`, no
> plural, e `initialize_database.py` faz `DROP TABLE IF EXISTS Atividades` —
> que não derruba nada. Ver [seção 6](#6-recriação-do-schema).

---

## 5. Migrações

**Não há Alembic nem qualquer sistema de migração.** O schema nasce de
`CREATE TABLE IF NOT EXISTS` dentro de cada `data/repo/*.criar_tabela()`, e
evolui por scripts pontuais em `scripts/`, rodados à mão uma vez por banco.

Como `IF NOT EXISTS` **não altera tabela existente**, cada mudança de coluna
precisou de um script. Os `ALTER` correspondentes estão em `data/sql/`, ao lado
do `CREATE`, para que o schema completo se leia em um lugar:

```python
# data/sql/ocr_documento_sql.py
ALTERAR_ADICIONAR_PASTA_SCRIPT = """
ALTER TABLE Ocr_Documento ADD COLUMN cod_pasta INT NULL;
ALTER TABLE Ocr_Documento ADD FOREIGN KEY (cod_pasta) REFERENCES Pasta(cod_pasta) ON DELETE SET NULL;
"""
```

### Histórico, em ordem de aplicação

| Script | O que fez |
| --- | --- |
| `migrar_pasta.py` | Criou `Pasta`, adicionou `Ocr_Documento.cod_pasta` |
| `migrar_processos_avulsos.py` | Adicionou `Pasta.padrao` |
| `migrar_avulsos_para_categoria.py` | Converteu `Processos Avulsos` de processo em categoria, criando uma subpasta por PDF |
| `migrar_searchable.py` | Adicionou `searchable_id` e `searchable_status` |
| `backfill_searchable.py` | Enfileirou documentos antigos no serviço de PDF pesquisável |
| `migrar_vezes_reprocessado.py` | Adicionou `vezes_reprocessado` |
| `migrar_feedback_campo.py` | Criou `Feedback_Campo` |
| `migrar_feedback_campo_append_only.py` | Removeu o `UNIQUE`, transformando em histórico |
| `migrar_feedback_campo_origem.py` | Adicionou `origem` |
| `migrar_feedback_campo_pasta.py` | Criou `Feedback_Campo_Pasta` |
| `migrar_anotacao_processo.py` | Criou `Anotacao_Processo` |
| `migrar_solicitacao_desvinculo.py` | Criou `Solicitacao_Desvinculo` |
| `corrigir_fuso_horario.py` | Corrigiu datas gravadas em UTC antes do ajuste de fuso no pool |

### Como aplicar uma migração em produção

```bash
docker compose exec -T backend python scripts/migrar_<nome>.py
```

Os scripts conferem o estado antes de alterar, então rodar duas vezes não
quebra. Mesmo assim: **backup antes**, sempre. Ver [deploy.md](deploy.md).

### Como escrever a próxima

1. Acrescente a coluna ao `CRIAR_TABELA` em `data/sql/`, para bancos novos.
2. Acrescente a constante `ALTERAR_...` ao lado, com o `ALTER` correspondente.
3. Crie `scripts/migrar_<assunto>.py` seguindo o padrão de
   `scripts/migrar_searchable.py`: conferir se já existe, aplicar, relatar.
4. Atualize este arquivo e a tabela de histórico acima.

---

## 6. Recriação do schema

`initialize_database.py` é o caminho de **ambiente novo**, não de produção:

```
1. SET FOREIGN_KEY_CHECKS = 0
2. DROP das tabelas
3. criar_tabela() de cada repo, na ordem de dependência:
     Campus, Usuario, Atividade, Pasta, Ocr_Documento,
     Solicitacao_Desvinculo, Feedback_Campo, Feedback_Campo_Pasta,
     Anotacao_Processo
4. seeders: importar_campus(), importar_usuarios()
```

Dois problemas conhecidos:

- **O `DROP` da tabela de atividades é um no-op.** O script derruba
  `Atividades` (plural) e a tabela é `Atividade` (singular). Reinicializando,
  as atividades antigas sobrevivem apontando para `cod_usuario` de usuários
  recriados com outros ids.
- **Há um `except` com sintaxe PostgreSQL** como alternativa ao
  `SET FOREIGN_KEY_CHECKS`, resquício de uma migração de banco considerada e
  não concluída. Ver o branch `origin/db/postgresql`.

---

## 7. Backup e restauração

```bash
# Backup
docker exec integracar-mysql mysqldump -uroot -p"$DB_PASSWORD" "$DB_NAME" \
  | gzip > backup_$(date +%F_%H-%M-%S).sql.gz

# Restauração
gunzip -c backup_2026-10-06_02-00-00.sql.gz \
  | docker exec -i integracar-mysql mysql -uroot -p"$DB_PASSWORD" "$DB_NAME"
```

O `docker-compose.yml` monta `./backups` em `/backups` dentro do container do
MySQL, para que o dump possa ser escrito de dentro. A rotina automática está
descrita em [deploy.md](deploy.md).

> **O backup do MySQL não é backup do sistema.** Os PDFs e os campos extraídos
> estão na workstation, no Postgres e no disco de `integracar-backend`.
> Restaurar só este banco devolve a estrutura de pastas e os vínculos, com os
> documentos apontando para ids que podem não existir mais do outro lado.
