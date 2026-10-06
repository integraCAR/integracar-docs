# Deploy — integracar-gestao em produção

Como o sistema roda em produção, como atualizar, e o que confirmar antes de
cada subida.

Pré-requisitos: [README.md](README.md) e [arquitetura.md](arquitetura.md).

---

## 1. Topologia de produção

```
   Internet
      │  HTTPS 443
      ▼
   nginx na VPS
      ├── /gestao/      ──► frontend  (container, Node 20, porta 3000)
      └── /gestao/api/  ──► backend   (container, uvicorn, porta 8000)
                                │
                                ├──► mysql (container, 127.0.0.1:3306)
                                │
                                └──► workstation, por HTTPS
                                       /api/         extração de campos
                                       /pesquisavel/ PDF pesquisável
```

| Item | Valor |
| --- | --- |
| Domínio | `www.integracar.agr.br` (também atende sem `www`) |
| Caminho da interface | `/gestao/` |
| Caminho da API | `/gestao/api/` |
| Containers | `integracar-mysql`, `integracar-backend`, `integracar-frontend` |
| Rede Docker | `integracar-network`, bridge |

O prefixo `/gestao/` existe porque o domínio hospeda mais de um sistema do
projeto. Ele aparece em **quatro lugares** que precisam concordar, e é a
principal fonte de erro de configuração:

| Onde | O que define |
| --- | --- |
| nginx | O roteamento dos dois caminhos |
| `main.py`, `root_path="/gestao/api"` | O que o FastAPI anuncia no OpenAPI (os caminhos reais continuam na raiz do app) |
| `react-router.config.ts`, `basename` | O prefixo das URLs que o frontend gera |
| `api.server.ts`, `apiPrefix` | O prefixo que o SSR usa ao chamar a API |

---

## 2. Pré-requisitos do servidor

- Linux (Ubuntu 20.04 LTS ou mais novo)
- Docker 20.10+ e Docker Compose 2.0+
- nginx
- Certificado TLS (Let's Encrypt)
- DNS apontando para a VPS
- 2 GB de RAM e 10 GB de disco, no mínimo
- Acesso SSH

---

## 3. Antes de subir: a checagem obrigatória

```bash
python3 check-deploy.py
```

Nove verificações, e o script sai com código diferente de zero se qualquer uma
falhar:

| # | Verifica |
| --- | --- |
| 1 | `.env` existe |
| 2 | Todas as 17 variáveis obrigatórias estão definidas (mascara as sensíveis na saída) |
| 3 | Coerência do ambiente: em produção, `FRONTEND_URL` sem `localhost` e com HTTPS |
| 4 | Python 3.10+ |
| 5 | `fastapi`, `uvicorn`, `mysql-connector-python`, `bcrypt`, `python-dotenv` importáveis |
| 6 | `SECRET_KEY` gerada (não o placeholder) e com 32+ caracteres |
| 7 | `.env` ignorado pelo git (`git check-ignore -v .env`) |
| 8 | Conectividade real com o MySQL (`SELECT VERSION()`) |
| 9 | Diretório `tests/` existe |

Além disso, manualmente:

```bash
pytest                                  # suíte do backend
cd frontend && npm run typecheck && npm run build
```

---

## 4. Variáveis de ambiente em produção

### Backend — `.env` na raiz

```env
ENVIRONMENT=production
FRONTEND_URL=https://www.integracar.agr.br
SECRET_KEY=<gerada só para produção, nunca a de dev>

DB_USER=integracar_prod
DB_PASSWORD=<senha longa e única>
DB_HOST=mysql
DB_PORT=3306
DB_NAME=integracar_prod

MAIL_USERNAME=...
MAIL_PASSWORD=...
MAIL_FROM=noreply@integracar.agr.br
MAIL_SERVER=smtp-relay.brevo.com
MAIL_PORT=587

OCR_API_BASE=https://<host-da-workstation>/api
OCR_SERVICE_TOKEN=<combinado com a equipe da workstation>
OCR_CALLBACK_TOKEN=<combinado com a equipe da workstation>
OCR_PUBLIC_BASE=https://www.integracar.agr.br/gestao/api
OCR_RECONCILIACAO_INTERVALO_S=300

SEARCHABLE_API_BASE=https://<host-da-workstation>/pesquisavel
SEARCHABLE_SERVICE_TOKEN=<combinado>
SEARCHABLE_CALLBACK_TOKEN=<combinado>
```

Três pontos que erram com frequência:

1. **`DB_HOST=mysql`**, não `localhost`: dentro da rede Docker o host é o nome
   do serviço.
2. **`OCR_PUBLIC_BASE` precisa do `/gestao/api`.** É nela que a API de extração
   faz o `POST` do callback. Sem o prefixo, o callback bate em caminho
   inexistente e o documento fica preso em `na_fila` para sempre.
3. **`OCR_SERVICE_TOKEN` e `OCR_CALLBACK_TOKEN` são tokens diferentes**, um por
   direção. Ver [integracoes.md](integracoes.md#2-autenticação-quatro-tokens-duas-direções).

### Frontend — armadilha conhecida

O serviço `frontend` no `docker-compose.yml` recebe hoje apenas:

```yaml
environment:
  NEXT_PUBLIC_API_URL: https://www.integracar.agr.br/gestao/api
```

**`NEXT_PUBLIC_API_URL` não é lida por nenhum código atual** — é resquício do
frontend Next.js antigo. As variáveis que o React Router realmente usa são:

| Variável | Efeito se ausente |
| --- | --- |
| `API_URL` | Cai no padrão `http://backend:8000`, que por acaso é o nome do serviço no compose. Funciona, mas por coincidência |
| `ENVIRONMENT` | `api.server.ts` **não** aplica o prefixo `/gestao/api` nas chamadas ao backend |

E há uma assimetria a entender antes de mexer: `react-router.config.ts`
considera produção quando `ENVIRONMENT === "production"` **ou**
`NODE_ENV === "production"`. Durante `npm run build`, `NODE_ENV` já é
`production`, então o `basename` fica `/gestao/` de qualquer forma. Já o
prefixo em `api.server.ts` depende **só** de `ENVIRONMENT`.

Resultado prático: sem `ENVIRONMENT`, o frontend gera URLs sob `/gestao/` (certo)
e chama o backend sem prefixo em `http://backend:8000` (também funciona, porque
os caminhos reais do FastAPI estão na raiz do app). A configuração "funciona",
mas por dois acidentes que se cancelam. **Antes de mexer no nginx ou no
compose, torne isso explícito:**

```yaml
environment:
  ENVIRONMENT: production
  API_URL: http://backend:8000
```

E remova `NEXT_PUBLIC_API_URL`, de `frontend/.env` também.

---

## 5. nginx

> **Este bloco é um modelo de referência, não uma cópia do arquivo em
> produção.** A configuração do nginx da VPS não está versionada em nenhum
> repositório deste workspace (o `nginx.conf` de `integracar-infra` é o gateway
> da workstation, outra máquina). O modelo abaixo foi derivado do que o código
> exige: o prefixo `/gestao/`, o `root_path` do FastAPI, o `basename` do React
> Router, o teto de 300 MB do upload e a verificação de `Host` das actions.
> Antes de aplicar, compare com o arquivo que já está no servidor
> (`/etc/nginx/sites-available/`), e considere versionar esse arquivo.

```nginx
upstream integracar_backend  { server 127.0.0.1:8000; }
upstream integracar_frontend { server 127.0.0.1:3000; }

server {
    listen 80;
    server_name integracar.agr.br www.integracar.agr.br;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name integracar.agr.br www.integracar.agr.br;

    ssl_certificate     /etc/letsencrypt/live/integracar.agr.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/integracar.agr.br/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options           "DENY"              always;
    add_header X-Content-Type-Options    "nosniff"           always;

    # PDF de processo de CAR pode passar de 100 MB
    client_max_body_size 320m;

    # API do gestão
    location /gestao/api/ {
        proxy_pass http://integracar_backend/;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout  300s;
        proxy_send_timeout  300s;
        proxy_request_buffering off;    # upload grande sem bufferizar em disco
    }

    # Interface do gestão
    location /gestao/ {
        proxy_pass http://integracar_frontend/;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Quatro detalhes que não são enfeite:

- **`proxy_set_header Host $host` é obrigatório.** A proteção nativa do React
  Router compara o `Host` recebido pelo Node com o `Origin` do navegador; sem
  repassar o `Host` correto, **toda** action é rejeitada — login incluído. O
  `allowedActionOrigins` em `react-router.config.ts` é o cinto de segurança
  disso.
- **`server_name` com e sem `www`.** O `Origin` chega em qualquer uma das duas
  formas, dependendo de como o usuário acessou; é por isso que as duas estão em
  `allowedActionOrigins`.
- **`client_max_body_size`** tem que acomodar o teto de 300 MB do upload. O
  padrão do nginx é 1 MB.
- **Timeouts generosos** no caminho da API: o upload para a workstation tem
  `UPLOAD_TIMEOUT` de 120 s do lado do cliente HTTP.

Ativar:

```bash
sudo ln -s /etc/nginx/sites-available/integracar /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### TLS

```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot certonly --nginx -d integracar.agr.br -d www.integracar.agr.br
```

Renovação automática pelo timer do certbot. Para conferir:
`sudo certbot renew --dry-run`.

### Firewall

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

O MySQL **não** entra: o compose já o publica apenas em `127.0.0.1:3306`.

---

## 6. Subida inicial

```bash
cd /app/integracar-gestao

docker compose up -d --build
docker compose ps                 # confirme mysql "healthy"

# APENAS em banco novo e vazio — este script apaga tudo
docker compose exec -T backend python initialize_database.py
```

### Ajuste recomendado no compose para produção

O `docker-compose.yml` versionado publica `backend` em `0.0.0.0:8000` e
`frontend` em `0.0.0.0:3000`. Em produção, com nginx na frente, prenda os dois
em localhost:

```yaml
backend:
  ports: ["127.0.0.1:8000:8000"]
frontend:
  ports: ["127.0.0.1:3000:3000"]
```

Sem isso, as duas portas ficam acessíveis pela internet, contornando o nginx —
e com elas o `/docs` do FastAPI, que não tem autenticação.

---

## 7. Atualização

```bash
cd /app/integracar-gestao

# 1. Backup, sempre
/usr/local/bin/backup-integracar.sh

# 2. Código
git pull origin main

# 3. Migração, se o PR trouxe uma
docker compose exec -T backend python scripts/migrar_<nome>.py

# 4. Rebuild e subida
docker compose up -d --build

# 5. Conferência
docker compose ps
docker compose logs -f --tail=100 backend
curl -I https://www.integracar.agr.br/gestao/
```

Depois de subir, confirme pelo navegador: login, abrir um processo existente,
e o PDF renderizando. São os três caminhos que dependem de configuração
(sessão, API de extração, proxy de PDF).

### Rollback

```bash
cd /app/integracar-gestao
git log --oneline -10
git checkout <commit-anterior>
docker compose up -d --build

# só se a migração precisar ser desfeita
gunzip -c backups/integracar_<data>.sql.gz \
  | docker exec -i integracar-mysql mysql -uroot -p"$DB_PASSWORD" integracar_prod
```

Rollback de código é rápido; rollback de schema não é. É por isso que o backup
vem antes da migração, e não depois.

---

## 8. Backup

`/usr/local/bin/backup-integracar.sh`:

```bash
#!/bin/bash
set -euo pipefail

# cron não herda o ambiente da sessão: lê DB_PASSWORD do próprio .env
set -a; . /app/integracar-gestao/.env; set +a

BACKUP_DIR="/app/integracar-gestao/backups"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
mkdir -p "$BACKUP_DIR"

docker exec integracar-mysql mysqldump \
    -uroot -p"$DB_PASSWORD" --single-transaction --quick \
    "$DB_NAME" > "$BACKUP_DIR/integracar_$DATE.sql"

gzip "$BACKUP_DIR/integracar_$DATE.sql"
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +30 -delete
echo "Backup concluído: integracar_$DATE.sql.gz"
```

```bash
sudo chmod +x /usr/local/bin/backup-integracar.sh
sudo crontab -e
#   0 2 * * * /usr/local/bin/backup-integracar.sh
```

`--single-transaction` evita travar as tabelas InnoDB durante o dump. O
`docker-compose.yml` já monta `./backups` em `/backups` dentro do container.

> **Backup do MySQL não é backup do sistema.** Os PDFs e os campos extraídos
> estão na workstation, no Postgres e no disco de `integracar-backend`.
> Restaurar só este banco devolve usuários, pastas e vínculos, com documentos
> apontando para ids que podem não existir mais do outro lado. O backup da
> workstation é responsabilidade do `integracar-infra`.

Teste a restauração em ambiente separado pelo menos uma vez por trimestre.
Backup não verificado não é backup.

---

## 9. Operação

```bash
docker compose ps                        # estado dos containers
docker compose logs -f backend
docker compose logs --tail=200 frontend
docker compose logs mysql

docker compose restart backend           # só a API
docker compose down                      # parar tudo
```

O log da aplicação também fica em `logs/app.log` dentro do container do
backend (10 MB por arquivo, 5 backups). Para seguir uma requisição inteira,
use o `request_id`:

```bash
docker compose exec backend grep "req_id=<uuid>" logs/app.log
```

### O que observar

| Sinal | Onde | O que significa |
| --- | --- | --- |
| Documentos acumulando em `na_fila` | `/monitoramento-extracao` ou o banco | Callback não chegando, ou workstation parada |
| `WARNING` "API de extração indisponível, reconciliação adiada" | `logs/app.log` | A workstation está fora. A reconciliação tenta no próximo ciclo |
| `ERROR` "Token de serviço OCR rejeitado" | `logs/app.log` | `OCR_SERVICE_TOKEN` divergente dos dois lados |
| `WARNING` "Callback OCR rejeitado - token inválido" | `logs/app.log` | `OCR_CALLBACK_TOKEN` divergente |
| `429` frequente | `logs/app.log` | Rate limit. Lembre que ele é por processo e zera no restart |
| Disco cheio | `df -h`, `docker system df` | Imagens antigas. `docker system prune -a` com cuidado |

---

## 10. Checklists

### Em cada deploy

- [ ] `python3 check-deploy.py` passando
- [ ] `pytest` passando
- [ ] `npm run typecheck` e `npm run build` sem erro
- [ ] Backup do banco feito **antes** da migração
- [ ] Migração do PR identificada e aplicada
- [ ] Login, abrir processo e renderizar PDF confirmados pelo navegador
- [ ] `docker compose logs backend` sem `ERROR` novo

### Semanal

- [ ] Containers de pé e sem restart em laço
- [ ] Backup do dia presente e com tamanho plausível
- [ ] `logs/app.log` sem `ERROR` recorrente
- [ ] Documentos presos em `na_fila` há mais de um dia

### Mensal

- [ ] Espaço em disco
- [ ] Validade do certificado (`sudo certbot certificates`)
- [ ] `pip-audit` e `npm audit`
- [ ] Pendências de [seguranca.md](seguranca.md#11-quadro-de-pendências)

### Trimestral

- [ ] Restauração de backup testada em ambiente separado
- [ ] Revisão de usuários e perfis ativos
- [ ] Rotação da `SECRET_KEY` avaliada (invalida todas as sessões)

---

## 11. Solução de problemas em produção

| Sintoma | O que checar |
| --- | --- |
| Container não sobe | `docker compose logs backend`. Quase sempre é variável de ambiente faltando — o boot falha alto de propósito |
| `502 Bad Gateway` no nginx | Container caído, ou a porta do `proxy_pass` não corresponde à publicada |
| Login rejeitado com erro de origem | `proxy_set_header Host $host` ausente no nginx, ou domínio fora de `allowedActionOrigins` |
| Interface carrega sem estilo nem JS | `basename` e `base` divergindo do caminho no nginx. Os ativos saem sob `/gestao/` |
| Sessão caindo a cada requisição | `API_URL` apontando para a URL pública em vez de `http://backend:8000`: o SSR sai para a internet e perde o cookie |
| Upload falhando em arquivo grande | `client_max_body_size` do nginx; e `proxy_request_buffering off` |
| Callback levando `403` | Isenção de CSRF não casou o caminho (tem que usar `get_route_path`, não `request.url.path`) |
| Callback levando `401` | Token de callback divergente |
| Documento preso em `na_fila` | `OCR_PUBLIC_BASE` sem o prefixo `/gestao/api`, ou inalcançável de fora |
| Horários 3 horas adiantados | `SET time_zone` não aplicado no checkout do pool. Ver [banco-de-dados.md](banco-de-dados.md#por-que-o-fuso-é-reaplicado-a-cada-conexão) |
| Certificado expirado | `sudo certbot renew && sudo systemctl reload nginx` |
| Lupa sem funcionar em nenhum PDF | As três `SEARCHABLE_*` configuradas? Faltando uma, o recurso fica desligado silenciosamente — por projeto |
