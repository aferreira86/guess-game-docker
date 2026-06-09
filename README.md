# Guess Game — Docker Compose

Implementação de infraestrutura Docker para o [guess_game](https://github.com/fams/guess_game), um jogo de adivinhação com backend Flask, banco de dados PostgreSQL e frontend React servido via NGINX.

---

## Estrutura do Repositório

```
.
├── docker-compose.yml        # Orquestração de todos os serviços
├── Dockerfile.backend        # Imagem do backend Flask
├── Dockerfile.frontend       # Imagem do frontend React + NGINX
├── nginx/
│   └── nginx.conf            # Configuração do proxy reverso e balanceamento de carga
├── README.md
│
│   (subdiretórios do projeto original, sem modificações)
├── guess/                    # Código Flask (blueprints, rotas)
├── repository/               # Acesso ao banco de dados (postgres, sqlite, etc.)
├── frontend/                 # Código-fonte React
├── run.py                    # Entry point do Flask
└── requirements.txt          # Dependências Python
```

---

## Arquitetura

```
                        ┌───────────────────────────────────┐
                        │        Rede Docker: backend_net    │
                        │                                    │
  Usuário               │  ┌──────────────────────────────┐  │
  ────────► :80 ───────►│  │   NGINX (frontend container) │  │
                        │  │                              │  │
                        │  │  • Serve arquivos React      │  │
                        │  │  • /api/* → proxy reverso    │  │
                        │  └──────────┬───────────────────┘  │
                        │             │ least_conn             │
                        │      ┌──────┴──────┐                │
                        │      ▼             ▼                │
                        │  ┌────────┐  ┌────────┐            │
                        │  │backend1│  │backend2│            │
                        │  │ :5000  │  │ :5000  │            │
                        │  └───┬────┘  └───┬────┘            │
                        │      └─────┬─────┘                  │
                        │            ▼                        │
                        │      ┌──────────┐                   │
                        │      │PostgreSQL│                   │
                        │      │  db:5432 │                   │
                        │      └──────────┘                   │
                        └───────────────────────────────────┘
```

---

## Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/) >= 24
- [Docker Compose](https://docs.docker.com/compose/install/) >= 2.20

---

## Como Instalar e Rodar

### 1. Clone o repositório original e adicione os arquivos Docker

```bash
git clone https://github.com/fams/guess_game.git
cd guess_game
```

Copie os arquivos deste repositório para dentro da pasta `guess_game`:
- `docker-compose.yml`
- `Dockerfile.backend`
- `Dockerfile.frontend`
- `nginx/nginx.conf`

### 2. Suba todos os serviços

```bash
docker compose up --build -d
```

### 3. Acesse a aplicação

Abra o navegador em:

```
http://localhost
```

> O frontend React estará disponível na raiz. As chamadas de API são automaticamente roteadas via NGINX para o backend Flask.

### 4. Verificar se está tudo funcionando

```bash
# Ver status dos containers
docker compose ps

# Verificar logs
docker compose logs -f

# Testar o health check do backend
curl http://localhost/api/health
```

---

## Como Jogar

1. Acesse `http://localhost`
2. Clique em **Maker** → crie um jogo digitando uma palavra secreta → anote o **Game ID** gerado
3. Clique em **Breaker** → insira o Game ID e tente adivinhar a palavra

---

## Como Atualizar Componentes

A estrutura foi projetada para permitir atualização de qualquer componente apenas trocando a tag da imagem no arquivo correspondente:

### Atualizar o PostgreSQL

Em `docker-compose.yml`, altere a linha:
```yaml
image: postgres:16-alpine
```
Por exemplo, para usar a versão 17:
```yaml
image: postgres:17-alpine
```
Em seguida: `docker compose up -d db`

### Atualizar o Backend (Python/Flask)

Em `Dockerfile.backend`, altere:
```dockerfile
FROM python:3.12-slim
```
Para a versão desejada (ex: `3.13-slim`), e então:
```bash
docker compose build backend1 backend2
docker compose up -d backend1 backend2
```

### Atualizar o Frontend (Node/NGINX)

Em `Dockerfile.frontend`, altere as linhas:
```dockerfile
FROM node:20-alpine AS builder
FROM nginx:1.27-alpine
```
E então:
```bash
docker compose build frontend
docker compose up -d frontend
```

---

## Decisões de Design

### Proxy Reverso e Balanceamento de Carga (NGINX)

O NGINX é o único ponto de entrada da aplicação (porta 80). Ele serve dois propósitos:

1. **Servir o frontend React**: os arquivos estáticos gerados pelo `npm run build` são copiados para `/usr/share/nginx/html` via multi-stage build.
2. **Proxy reverso para a API**: todas as requisições com prefixo `/api/` são redirecionadas para o upstream `backend_pool`, que contém as duas instâncias do backend (`backend1` e `backend2`).

O algoritmo de balanceamento escolhido foi `least_conn` (menor número de conexões ativas), que distribui a carga de forma mais inteligente que o round-robin simples, especialmente quando requisições têm durações variadas.

### Variável de Ambiente `REACT_APP_BACKEND_URL`

O frontend usa `process.env.REACT_APP_BACKEND_URL` para saber onde está a API. Em vez de hardcodar o endereço do backend, definimos essa variável como `/api` em build-time. Assim, o React sempre chama `/api/create`, `/api/guess/:id` etc., e o NGINX se encarrega de rotear para o backend correto — sem expor portas do backend ao host.

### Multi-stage Build do Frontend

O Dockerfile do frontend usa dois estágios:
- **Stage `builder`**: instala Node.js, dependências e compila o React (`npm run build`)
- **Stage final**: apenas NGINX Alpine + os arquivos estáticos compilados

Isso resulta em uma imagem muito menor (sem Node.js na imagem final) e mais segura.

### Duas Instâncias Fixas de Backend

Em vez de usar `docker compose up --scale`, optamos por declarar `backend1` e `backend2` explicitamente no `docker-compose.yml`. Isso porque o NGINX precisa referenciar os servidores do upstream pelo nome, e nomes gerados por `--scale` (ex: `guess_game_backend_1`) são menos previsíveis entre versões do Compose.

### Volume Persistente para o PostgreSQL

O volume `postgres_data` é declarado como named volume. Isso garante que os dados do banco sobrevivam a `docker compose down` e só sejam removidos explicitamente com `docker compose down -v`.

### Reinício Automático

Todos os serviços têm `restart: always`, o que faz o Docker reiniciar automaticamente o container em caso de falha ou reinicialização do host.

### Health Check no Banco

O banco de dados declara um `healthcheck` com `pg_isready`. Os backends (`depends_on: db: condition: service_healthy`) só sobem após o PostgreSQL estar pronto para aceitar conexões, evitando falhas de inicialização por race condition.

---

## Parar a Aplicação

```bash
# Para os containers (dados do banco são preservados)
docker compose down

# Para os containers E remove o volume do banco (apaga todos os dados)
docker compose down -v
```
