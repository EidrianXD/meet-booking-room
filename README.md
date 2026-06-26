# Sistema de Gerenciamento de Salas

Monorepo contendo o backend (Node.js + Express + Prisma + PostgreSQL) e o frontend (Quasar/Vue) do sistema de agendamento de salas de reunião, com containerização Docker e pipeline CI/CD no Jenkins.

Fluxo do pipeline: push no `main` → checkout com submodules → testes paralelos → build de imagens → scan Trivy → push GHCR → deploy local com 3 containers (`postgres`, `backend`, `frontend`) ordenados por healthcheck.

---

## Sumário

1. [Demo em 5 minutos](#demo-em-5-minutos)
2. [Contexto da aplicação](#1-contexto-da-aplicação)
3. [Objetivo](#2-objetivo)
4. [Arquitetura](#3-arquitetura)
5. [Decisões de design e tradeoffs](#4-decisões-de-design-e-seus-tradeoffs)
6. [Plano em fases (executado)](#5-plano-em-fases)
7. [Estrutura do repositório](#6-estrutura-do-repositório)
8. [Riscos conhecidos e mitigações](#7-riscos-conhecidos-e-mitigações)
9. [Status atual](#8-status-atual)

---

## Demo em 5 minutos

**Pré-requisitos:** Docker Desktop instalado e rodando.

Este guia leva qualquer pessoa do zero — sem nenhuma configuração prévia do GitHub ou Jenkins — até ver o pipeline rodando no próprio computador. Cada operador usa **o seu próprio GitHub** (não o do autor do projeto).

### Passo 1 — Clonar o repositório

```bash
git clone --recurse-submodules https://github.com/EidrianXD/meet-booking-room.git
cd meet-booking-room
```

### Passo 2 — Configurar variáveis de ambiente

```bash
cp .env.example .env
```

Edite `.env` e ajuste:

- `GHCR_OWNER` → seu login do GitHub em **lowercase** (ex: `joaosilva` se você é `JoaoSilva` no GitHub)
- `JWT_SECRET` → qualquer string aleatória (`openssl rand -hex 32`)
- `DB_PASSWORD` → senha do Postgres local (`openssl rand -hex 24`)

### Passo 3 — Subir só a aplicação

```bash
docker compose up --build
```

Quando os 3 containers ficarem `healthy`, abra **http://localhost:8080** e faça login com `john` / `123456`.

### Passo 4 — Subir o pipeline CI/CD completo

Para ver o ciclo completo (testes + build + scan + push + deploy), suba o Jenkins local:

```bash
# em outro terminal
cd docker/jenkins
cp .env.example .env             # edite JENKINS_ADMIN_PASSWORD
docker compose up -d --build
```

Acesse **http://localhost:8081** e siga o passo-a-passo de [`docker/jenkins/README.md`](docker/jenkins/README.md) — o guia leva você por:

1. Como gerar um Personal Access Token (PAT) no GitHub
2. Cadastrar as 3 credenciais (`ghcr-pat`, `jwt-secret-prod`, `db-password-prod`)
3. Criar o job `distrimed`
4. Disparar a primeira build

Ao final você terá:
- **Imagens publicadas no seu GHCR** em `https://github.com/SEU_USUARIO?tab=packages`
- **Aplicação prod rodando** em http://localhost:8090 (independente da app dev em :8080)

---

Este documento descreve a containerização e CI/CD implementados em fases incrementais.

---

## 1. Contexto da aplicação

| Camada | Stack | Observações |
|---|---|---|
| Backend | Node 20 + TypeScript + Express + Prisma | Clean Architecture (`domain`/`application`/`infrastructure`/`presentation`), `/health` exposto, JWT, Vitest |
| Banco | **PostgreSQL 16-alpine** | Schema em `backend/prisma/schema.prisma`, migrations versionadas em `backend/prisma/migrations/` |
| Frontend | Quasar (Vue 3) + Vite + TypeScript | SPA, build estático servido por `nginx:1.27-alpine` em produção |
| Testes | Vitest nos dois projetos (20 + 18 testes) | Rodam em containers Node no pipeline |
| Registry | GitHub Container Registry (GHCR) | Tags `<sha-7>` (imutável) e `latest` (mutável) |
| CI/CD | Jenkins LTS (declarative pipeline) | DooD via socket do host; SCM polling 1 min |
| Scanner | Trivy (`HIGH`/`CRITICAL` + `--ignore-unfixed`) | Gate real: bloqueia push se base image tiver CVE corrigível |

Pontos relevantes para a containerização:

- Backend já lê configuração de variáveis de ambiente (`JWT_SECRET`, `PORT`, `CORS_ORIGINS`, `DATABASE_URL`) — nenhum hardcode.
- `/health` já existe — pronto para `healthcheck` do Compose.
- Frontend é SPA pura — basta servir `dist/spa` com fallback para `index.html`.

---

## 2. Objetivo

Entregar um pipeline DevOps funcional end-to-end, contendo:

1. Imagens Docker multi-stage, executando como usuário não-root, para backend e frontend.
2. Orquestração local via Docker Compose para desenvolvimento e deploy.
3. Pipeline declarativo no Jenkins cobrindo: lint, testes, build, scan de vulnerabilidades, push de imagens e deploy.
4. Imagens versionadas e publicadas no GitHub Container Registry (GHCR).
5. Scan de segurança com Trivy como quality gate de build.

Restrições do escopo:

- **Incremental**: ao final de cada fase o sistema continua funcional.
- **Local**: tudo roda em uma única máquina. Kubernetes/Helm/service mesh ficam fora do escopo.
- **Reprodutível**: o setup do Jenkins é definido em código (JCasC + plugins.txt), não em UI.

---

## 3. Arquitetura

```
┌────────────┐  git push   ┌─────────────┐
│  GitHub    │ ──poll────▶ │   Jenkins   │  (LTS, JCasC, DooD via socket)
└────────────┘  SCM 1 min  │  (Docker)   │
                           └──────┬──────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
  Backend tests             Build images                Trivy scan
  (node:20-slim)            (paralelo, multi-          (HIGH+CRITICAL
  Frontend tests            stage, sha+latest)         --ignore-unfixed)
  (node:22-slim)                  │                         │
                                  └─────────────┬───────────┘
                                                ▼
                                     docker push GHCR
                                  (4 tags por build)
                                                │
                                                ▼
                            docker compose -p distrimed-prod
                            pull && up -d --no-build
                                                │
                  ┌─────────────────────────────┼─────────────────────────────┐
                  ▼                             ▼                             ▼
            ┌──────────┐                ┌──────────┐                  ┌──────────┐
            │ frontend │ ◀── proxy ──▶  │ backend  │  ◀── network ──▶ │ postgres │
            │ nginx    │   /api/* →     │ express  │   conn pool      │ 16-alpine│
            │ :80→8090 │                │ :3000    │                  │ pgdata   │
            │ healthz  │                │ /health  │                  │pg_isready│
            └──────────┘                └──────────┘                  └──────────┘
              ▲ exposto                  ▲ interno                     ▲ interno
              ao host                    à network                     à network
```

**Ordem de boot** (`depends_on: condition: service_healthy`):
`postgres healthy → backend starts → backend healthy → frontend starts`. Sem race conditions.

**Isolamento dev/prod:** mesmo `docker-compose.yml`, separação por `-p <project>` (`distrimed` vs `distrimed-prod`). Volumes e networks ganham prefixo automático — `distrimed_postgres-data` ≠ `distrimed-prod_postgres-data`.

---

## 4. Decisões de design (e seus tradeoffs)

| Decisão | Motivo | Alternativa descartada |
|---|---|---|
| **SQLite na Fase 1 → Postgres na Fase 6** | Reduz superfície de falha do pipeline inicial; Postgres entra depois que a CI/CD está estável. Em cada estágio o sistema continua funcional. | Já entrar com Postgres adicionaria healthchecks, secret do DB e ordering — caminho crítico maior. |
| **Um único `docker-compose.yml` + `-p <project>`** | Dev (`distrimed`) e prod (`distrimed-prod`) saem do mesmo arquivo, isolados pelo prefixo do projeto. Volumes e networks sem `name:` explícito recebem prefixo automático. | Dois composes (`dev.yml`/`prod.yml`) duplica manutenção. |
| **Jenkins local com `docker.sock` montado (DooD)** | Permite que o pipeline rode `docker build/push/run` no daemon do host sem aninhar um segundo daemon. | Docker-in-Docker (DinD) tem custo de setup; kaniko adiciona atrito sem ganho proporcional. Em produção remota, agente remoto isolado seria o caminho. |
| **Deploy via `compose up` no mesmo host do Jenkins** | Evita setup de SSH/firewall/DNS no escopo local. Em produção remota: SSH para `$DEPLOY_HOST` antes do `compose pull && up -d`. | Servidor remoto exige infraestrutura adicional fora do escopo local. |
| **GHCR como registry** | Gratuito, integrado ao GitHub, autenticação via PAT. Tags imutáveis `<sha-7>` + ponteiro `latest` permitem rollback trocando uma variável. | Docker Hub tem limites de pull; Harbor/Nexus self-hosted adiciona mais um container. |
| **Trivy com `--severity HIGH,CRITICAL --ignore-unfixed --skip-dirs <npm bundled>`** | Gate ativo: bloqueia o build quando há CVE com fix disponível. Ignora CVEs sem fix (sem ação possível) e o npm bundled do Node base (não executado em runtime). | Falhar em qualquer CVE deixa pipeline vermelho sem ação. Ignorar todos torna o gate inerte. |
| **JCasC (Configuration as Code) no Jenkins** | Admin user definido em YAML + env vars; sem setup wizard. Reprodutível: derrubar e subir o controller produz a mesma configuração. | Wizard manual + plugins instalados via UI produz setup não-reprodutível. |
| **`agent docker { image '...'; reuseNode true }`** | Cada stage de teste roda num container com a versão exata de Node necessária (`node:20-slim` backend, `node:22-slim` frontend). Não exige Node instalado no controller. | Instalar Node como tool no Jenkins prende o pipeline a uma versão. |
| **SCM polling 1 min em vez de webhook** | Funciona com Jenkins local sem expor porta ao GitHub. Latência de até 1 min entre push e build. | Webhook exigiria VPS pública ou túnel ngrok (descrito em `docker/jenkins/README.md`). |
| **`SEED_ON_BOOT` como env var no entrypoint** | Mesma imagem roda em dev (seed=true, login imediato) e prod (seed=false, dados controlados). Seed é idempotente (`upsert`). | Duas imagens ou entrypoint custom em dev duplica complexidade. |

---

## 5. Plano em fases

A implementação é dividida em fases independentes. Ao final de cada fase, o estado do repositório é funcional.

### Fase 0 — Preparação

Trabalho de base que destrava as fases seguintes.

- [x] Adicionar `.gitignore` raiz cobrindo `node_modules`, `dist`, `.env`, `*.db`.
- [x] Adicionar `.env.example` em `backend/` documentando `JWT_SECRET`, `PORT`, `CORS_ORIGINS`, `DATABASE_URL` (já existia, completo).
- [x] Adicionar script `lint` aos `package.json` de backend e frontend (frontend já tinha; backend agora usa `tsc --noEmit`).
- [x] Registrar `backend/` e `frontend/` como **git submodules** do repo raiz (`EidrianXD/meeting-room-booking-{api,frontend}`).

**Critério de aceite:** `git status` limpo; `npm test` passa nos dois projetos a partir de uma instalação limpa. ✅

---

### Fase 1 — Containerização do backend

- [x] `backend/Dockerfile` multi-stage:
  - `deps`: instala todas as dependências para o build.
  - `build`: roda `prisma generate` e `tsc`, depois `npm prune --omit=dev`.
  - `runtime`: imagem `node:20-slim` (Debian, evita complicação de `binaryTargets` musl do Alpine para o Prisma), usuário não-root (`node`), copia `node_modules` já pruned, `dist/` e `prisma/`. Expõe `3000`. Entrypoint que roda `prisma migrate deploy` antes de `node dist/src/main.js`. Inclui `HEALTHCHECK` apontando para `/health`.
- [x] `backend/.dockerignore` (exclui `node_modules`, `dist`, `prisma/dev.db`, `.env`, `tests`, etc.).
- [x] Ajustes paralelos:
  - `prisma` CLI movido de `devDependencies` para `dependencies` (necessário para `migrate deploy` em runtime).
  - `package-lock.json` removido do `.gitignore` (necessário para `npm ci` reprodutível no Jenkins).
- [x] `schema.prisma` com `binaryTargets = ["native", "debian-openssl-3.0.x"]` (descoberto na validação: imagem `node:20-slim` usa OpenSSL 3 e o Prisma Client precisava do engine correspondente).
- [x] Validar build local: `docker build -t distrimed-backend:dev backend/` (~493 MB).
- [x] Validar `/health` respondendo dentro do container: `GET http://localhost:3000/health` → `{"status":"ok"}`; `HEALTHCHECK` reportando `healthy`.

**Critério de aceite:** container do backend sobe isoladamente e responde em `GET /health`. ✅

---

### Fase 2 — Containerização do frontend

- [x] `frontend/Dockerfile` multi-stage:
  - `build`: `node:22-slim` (Quasar 2.6 exige Node 22+), roda `quasar build` gerando `dist/spa`.
  - `runtime`: `nginx:1.27-alpine` servindo `dist/spa` com config customizada (SPA fallback, gzip, cache imutável para assets versionados, `HEALTHCHECK` em `/healthz`).
- [x] `frontend/.dockerignore`.
- [x] `frontend/docker/nginx.conf` mínimo e otimizado para SPA, com **reverse proxy `/api/* → http://backend:3000/`** (resolve CORS, evita rebuild por ambiente).
- [x] URL da API resolvida via `ARG VITE_API_BASE_URL=/api` (default casa com o proxy do Nginx; pode ser sobrescrito por `--build-arg` se precisar de URL absoluta).
- [x] `package-lock.json` removido do `.gitignore` do frontend (necessário para `npm ci` reprodutível no Jenkins).
- [x] Validação integrada: `distrimed-frontend:dev` (75 MB) e `distrimed-backend:dev` numa network compartilhada — `GET /` serve a SPA, `GET /api/health` proxia para o backend e retorna `{"status":"ok"}`.

**Critério de aceite:** SPA carrega no navegador e consegue chamar o backend rodando em outro container. ✅

---

### Fase 3 — Orquestração local com Docker Compose

- [x] `docker-compose.yml` na raiz com serviços `backend` e `frontend`:
  - Volume nomeado `distrimed-backend-data` montado em `/app/data` (persistência do `dev.db` do SQLite).
  - `HEALTHCHECK` herdado das imagens (definido nos Dockerfiles).
  - `frontend` com `depends_on: backend.condition: service_healthy`.
  - Network bridge nomeada (`distrimed-net`); backend **não exposto** ao host, toda comunicação SPA↔API passa pelo proxy do Nginx.
  - Variáveis lidas de `.env` na raiz com defaults sensatos via `${VAR:-default}`; `JWT_SECRET` obrigatório (`${JWT_SECRET:?...}`).
- [x] `.env.example` na raiz documentando `JWT_SECRET`, `SEED_ON_BOOT`, `FRONTEND_PORT`, `VITE_API_BASE_URL`.
- [x] Entrypoint do backend ganhou suporte a `SEED_ON_BOOT=true` — roda `dist/prisma/seed.js` (idempotente, upsert) após `migrate deploy`. Permite que o primeiro `docker compose up` já entregue usuários e salas prontos para o login.
- [x] Validação ponta a ponta (curl simulando o browser):
  - `GET /api/health` → `{"status":"ok"}`
  - `POST /api/auth/login` (`john`/`123456`) → JWT retornado
  - `GET /api/rooms` autenticado → 3 salas (`Alfa`, `Beta`, `Gama`)
  - `POST /api/bookings` autenticado → booking criado
  - **Persistência:** `docker compose restart backend` → booking sobreviveu (volume OK).

**Critério de aceite:** `docker compose up` levanta a aplicação completa e o fluxo principal funciona pelo navegador. ✅

---

### Fase 4 — Pipeline Jenkins

- [x] `docker/jenkins/` — stack do Jenkins local (Docker outside of Docker via socket do host):
  - `Dockerfile` com plugins pré-instalados (workflow, docker-workflow, github, configuration-as-code, etc.), Docker CLI + Compose plugin, `usermod -aG root jenkins` para acesso ao socket.
  - `jenkins.yaml` (JCasC) bootstrapa o admin user via env (sem setup wizard).
  - `docker-compose.yml` + `.env.example` + `README.md` com instruções de setup (credenciais + criação do job).
- [x] `Jenkinsfile` declarativo na raiz com os stages:
  1. **Checkout** — com `git submodule update --init --recursive`.
  2. **Backend / Frontend: install + lint + test** — sequencial entre os dois, cada um num container `node:20-slim` / `node:22-slim` via `agent docker { reuseNode true }`.
  3. **Build images** — paralelo, taggeando com `${SHA_SHORT}` e `latest`.
  4. **Trivy scan** — paralelo, `--severity HIGH,CRITICAL --ignore-unfixed --skip-dirs usr/local/lib/node_modules/npm`, caches separados por scan para evitar lock contention do BoltDB.
  5. **Push GHCR** — `docker login` com credencial `ghcr-pat` (Username/Password) + push das 4 tags.
  6. **Deploy local prod** — re-autentica no GHCR, `docker compose -p distrimed-prod pull && up -d --no-build` com `BACKEND_TAG`/`FRONTEND_TAG` = SHA-7 da build. Roda na porta 8090.
- [x] Polling SCM (`H/1 * * * *`) — substituto local do webhook do GitHub.
- [x] Quality gate Trivy:
  - Backend: `apt-get upgrade` no runtime stage zera os CVEs de glibc/libcap2/libsystemd0.
  - Frontend: `apk upgrade` no runtime stage zera os CVEs de openssl/libpng/libxml2/musl/zlib.
  - Bundled `npm` do Node base image (`/usr/local/lib/node_modules/npm`) explicitamente skipado (não roda em runtime).

**Critério de aceite:** push no `main` dispara o pipeline, todos os stages ficam verdes e a versão nova fica acessível no `compose up -d` final. ✅

**Validação na build #3 (commit `0e63d77`):**
- 4 tags publicadas no GHCR (`distrimed-backend:0e63d77`, `:latest`, `distrimed-frontend:0e63d77`, `:latest`).
- `distrimed-prod` rodando em `http://localhost:8090`, com fluxo completo (SPA → login → rooms) validado via curl.

---

### Fase 5 — Quality gate de segurança refinado (parcial)

- [x] Configuração do Trivy já entregue na Fase 4:
  - Falha em `HIGH`/`CRITICAL`.
  - Ignora CVEs sem fix disponível (`--ignore-unfixed`).
  - Skipa `usr/local/lib/node_modules/npm` do Node base image (não executado em runtime).
  - Caches separados por scan paralelo (`trivy-cache-backend`, `trivy-cache-frontend`) — evita lock contention do BoltDB.
- [x] Timeout global de 30 min no pipeline (`options { timeout(...) }`).
- [ ] Relatório SARIF arquivado como artefato do build (`archiveArtifacts`). _Polimento opcional; sem demanda real de SOC/análise externa no momento._

---

### Fase 6 — Graduação para PostgreSQL ✅

- [x] `schema.prisma`: `provider = "postgresql"`.
- [x] Serviço `postgres:16-alpine` no `docker-compose.yml`:
  - Volume `postgres-data` (sem `name:` → prefixo automático por projeto).
  - Healthcheck `pg_isready -U $POSTGRES_USER -d $POSTGRES_DB`.
  - Senha obrigatória via `${DB_PASSWORD:?...}` (falha cedo se ausente).
  - **Porta não exposta no host** — só acessível dentro da network do Compose.
- [x] Backend `depends_on: postgres: condition: service_healthy`.
- [x] Entrypoint do backend roda `prisma migrate deploy` antes do `node` (com Postgres já healthy).
- [x] Migration inicial regenerada com `prisma migrate dev --name init` contra Postgres 16.
- [x] `DATABASE_URL` removida do Dockerfile — agora **obrigatória em runtime** (injetada pelo compose como `postgresql://${USER}:${DB_PASSWORD}@postgres:5432/${DB}`).
- [x] Jenkinsfile: nova credencial `db-password-prod` (Secret text) injetada no stage de deploy via `withCredentials`.
- [x] `.env.example` documenta `DB_PASSWORD`, `POSTGRES_DB`, `POSTGRES_USER`.

**Critério de aceite:** mesmo fluxo da Fase 3 funcionando, agora com Postgres no lugar do SQLite, com persistência atravessando restart do container. ✅

**Validação na build #4 (commit `f10e0c2`):**
- 4 tags publicadas no GHCR.
- Deploy `distrimed-prod` rodando 3 containers em `http://localhost:8090` com ordering `postgres healthy → backend healthy → frontend started`.
- Login + booking + restart do backend → dados persistidos no volume `distrimed-prod_postgres-data`.

---

### Fase 7 — Polimentos (parcial)

- [x] **Isolamento real entre dev e prod** — `name:` removido dos volumes e networks; prefixo automático por project name (`distrimed_postgres-data` vs `distrimed-prod_postgres-data`). Era o "ponto conhecido" da Fase 4.
- [x] **Tagging imutável** — toda imagem ganha `<sha-7>` (commit) + `latest`. Rollback é uma linha (`BACKEND_TAG=<sha-anterior> docker compose up -d`).
- [x] **README com TOC, seção de demo, sumário visual.**
- [ ] Badge de status do build no README (Jenkins local não tem URL pública; viável só com VPS).
- [ ] Versionamento semântico das imagens (`v1.0.0`) via tag git — exige fluxo de release; fora do escopo atual.
- [ ] BuildKit/buildx no Jenkins (acelera build incremental). Hoje usa o builder legacy.
- [ ] Makefile com atalhos (`make up`, `make test`, `make pipeline-logs`).
- [ ] Documentar fluxo de migração para VPS pública (texto, sem implementar).

---

## 6. Estrutura do repositório

```
distrimed/                                ← repo raiz (este)
├── README.md                             ← este arquivo
├── .env.example                          ← JWT_SECRET, DB_PASSWORD, FRONTEND_PORT, ...
├── .gitignore                            ← cobre node_modules, .env, .quasar, etc.
├── .gitmodules                           ← registra backend e frontend como submodules
├── docker-compose.yml                    ← postgres + backend + frontend
├── Jenkinsfile                           ← pipeline declarativo
├── docker/
│   └── jenkins/                          ← stack do Jenkins controller
│       ├── Dockerfile                    ← plugins + Docker CLI + DooD
│       ├── docker-compose.yml
│       ├── jenkins.yaml                  ← JCasC: admin bootstrap
│       ├── plugins.txt
│       ├── .env.example
│       └── README.md                     ← setup (credenciais + criação do job)
├── backend/  ← submodule: github.com/EidrianXD/meeting-room-booking-api
│   ├── Dockerfile                        ← multi-stage: deps → build → runtime
│   ├── .dockerignore
│   ├── .env.example
│   ├── docker/
│   │   └── entrypoint.sh                 ← prisma migrate deploy + (seed) + node
│   ├── prisma/
│   │   ├── schema.prisma                 ← provider: postgresql
│   │   ├── migrations/<timestamp>_init/  ← gerada com prisma migrate dev
│   │   └── seed.ts
│   ├── src/                              ← Clean Architecture (domain/app/infra/presentation)
│   └── tests/                            ← Vitest (20 testes)
└── frontend/ ← submodule: github.com/EidrianXD/meeting-room-booking-frontend
    ├── Dockerfile                        ← multi-stage: build (node:22-slim) → runtime (nginx:alpine)
    ├── .dockerignore
    ├── .env.example
    ├── docker/
    │   └── nginx.conf                    ← SPA fallback + gzip + proxy /api/* → backend
    ├── src/                              ← Vue 3 / Quasar / Pinia / FullCalendar
    └── tests/                            ← Vitest (18 testes)
```

---

## 7. Riscos conhecidos e mitigações

| Risco | Mitigação |
|---|---|
| Jenkins em container precisa buildar Docker → exige `docker.sock` | Trade-off documentado. Em produção remota, usar agente remoto ou BuildKit em VM isolada. |
| Usar SQLite em container | Resolvido na Fase 6 — projeto roda em PostgreSQL 16 com healthcheck `pg_isready` e senha em Jenkins Credentials. |
| Trivy falhar com CVEs da base do Node | Ocorreu nas builds #1 e #2; resposta foi atualizar as bases (`apt-get upgrade` + `apk upgrade`) em vez de afrouxar o gate. Build #3 passou com 0 CVEs. |
| Frontend precisa saber URL do backend em build-time (SPA estática) | Resolvido com `ARG VITE_API_BASE_URL=/api` + reverse proxy Nginx no mesmo container. Uma imagem serve qualquer ambiente sem rebuild. |
| Race condition entre `migrate deploy` e múltiplas instâncias do backend | Não relevante no escopo atual (1 instância). Em produção, migration roda como job separado antes do rollout. |
| Credenciais (`GHCR_PAT`, `JWT_SECRET`, `DB_PASSWORD`) tocando disco/repo | Todas vivem em Jenkins Credentials, injetadas via `withCredentials` apenas dentro do bloco `sh`. `docker logout` em `post { always }` limpa o registry config. |
| Lock contention do Trivy quando 2 scans rodam em paralelo no mesmo cache | Resolvido na build #2: caches separados por scan (`trivy-cache-backend`, `trivy-cache-frontend`). |
| Schema do Prisma + base image incompatíveis em runtime (`debian-openssl-1.1.x` vs `3.0.x`) | Resolvido na Fase 1: `binaryTargets = ["native", "debian-openssl-3.0.x"]`. |

---

## 8. Status atual

| Fase | Status | Builds que validaram |
|---|---|---|
| 0 — Preparação | concluída | — |
| 1 — Dockerfile backend | concluída | validação local |
| 2 — Dockerfile frontend | concluída | validação local |
| 3 — Docker Compose | concluída | validação local |
| 4 — Jenkins pipeline | concluída | builds #1 (falha: CVEs), #2 (falha: lock), #3 (sucesso) |
| 5 — Trivy refinements | concluída (SARIF arquivado fica pendente, opcional) | build #3 |
| 6 — Postgres | concluída | build #4 (sucesso) |
| 7 — Polimentos | parcial (isolamento dev/prod, tagging e README concluídos; demais opcionais) | — |

**Resultado do pipeline na build #4:**
- 4 tags publicadas no GHCR por build (backend SHA + latest, frontend SHA + latest).
- 0 CVEs HIGH/CRITICAL com fix disponível nas duas imagens (40+ CVEs corrigidos via `apt-get upgrade` / `apk upgrade` nas bases).
- 3 containers no deploy: `postgres healthy → backend healthy → frontend started`.

**Reinicializar do zero:**

```bash
docker compose -p distrimed-prod down -v
docker compose -p distrimed down -v
cd docker/jenkins && docker compose down -v && cd ../..
# subir na ordem: jenkins → aplicação dev → disparar pipeline
```

**Commits seguem padrão semântico** (`tipo(escopo): título`), sem co-author, em três repositórios:
- raiz: `github.com/EidrianXD/meet-booking-room`
- backend: `github.com/EidrianXD/meeting-room-booking-api`
- frontend: `github.com/EidrianXD/meeting-room-booking-frontend`
