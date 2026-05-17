# Distrimed — Sistema de Gerenciamento de Salas

Monorepo contendo o **backend** (Node.js + Express + Prisma) e o **frontend** (Quasar/Vue) do sistema de agendamento de salas de reunião.

Este documento descreve o **planejamento de containerização e CI/CD** que será implementado por cima da base de código existente. A intenção é entregar um pipeline funcional de ponta a ponta, em fases incrementais.

---

## 1. Contexto da aplicação

| Camada | Stack | Observações |
|---|---|---|
| Backend | Node 20 + TypeScript + Express + Prisma | Clean Architecture (`domain`/`application`/`infrastructure`/`presentation`), `/health` já exposto, JWT, Vitest |
| Banco | SQLite (dev) → PostgreSQL (Fase 2) | Schema em `backend/prisma/schema.prisma`, migration inicial já aplicada |
| Frontend | Quasar (Vue 3) + Vite + TypeScript | SPA, build estático servido por Nginx em produção |
| Testes | Vitest nos dois projetos | |

Pontos relevantes para a containerização:

- Backend já lê configuração de variáveis de ambiente (`JWT_SECRET`, `PORT`, `CORS_ORIGINS`, `DATABASE_URL`) — nenhum hardcode.
- `/health` já existe — pronto para `healthcheck` do Compose.
- Frontend é SPA pura — basta servir `dist/spa` com fallback para `index.html`.

---

## 2. Objetivo

Entregar um **pipeline DevOps de qualidade sênior**, demonstrável end-to-end, contendo:

1. Imagens Docker otimizadas (multi-stage, não-root, mínimas) para backend e frontend.
2. Orquestração local via Docker Compose para desenvolvimento e demonstração.
3. Pipeline declarativo no **Jenkins** cobrindo: lint, testes, build, scan de vulnerabilidades, push de imagens e deploy.
4. Imagens versionadas e publicadas no **GitHub Container Registry (GHCR)**.
5. Scan de segurança com **Trivy** como quality gate.

Princípios norteadores:

- **Pragmatismo sobre teatro**: cada peça precisa fazer sentido na demo. Sem Kubernetes, sem service mesh, sem Helm — pode-se mencionar como próximo passo.
- **Incremental**: ao final de cada fase o sistema continua funcional. Sem janela de "tudo quebrado".
- **Demonstrável**: tudo roda em uma única máquina (laptop ou VPS pequeno).

---

## 3. Arquitetura-alvo

```
┌────────────┐    git push    ┌─────────────┐
│  GitHub    │ ──webhook────▶ │   Jenkins   │
└────────────┘                │  (Docker)   │
                              └──────┬──────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                      ▼
        Lint + Test           Build + Scan             Push GHCR
                                     │                      │
                                     │                      ▼
                                     │                ghcr.io/<owner>/
                                     │                  distrimed-{back,front}
                                     ▼
                          docker compose -p prod up -d
                              │           │           │
                              ▼           ▼           ▼
                        ┌─────────┐ ┌─────────┐ ┌─────────┐
                        │frontend │ │ backend │ │ sqlite  │
                        │ (nginx) │ │ (node)  │ │ (vol.)  │
                        └─────────┘ └─────────┘ └─────────┘
```

Na **Fase 2**, o bloco `sqlite (vol.)` é substituído por um serviço `postgres` com volume persistente e healthcheck.

---

## 4. Decisões de design (e seus tradeoffs)

| Decisão | Por quê | Alternativa descartada |
|---|---|---|
| **SQLite na Fase 1, Postgres na Fase 2** | Reduz superfície de falha do pipeline inicial. Permite ter algo funcional rápido. | Já entrar com Postgres adicionaria healthchecks, `migrate deploy` no boot, secret do DB — caminho crítico maior. |
| **Um único `docker-compose.yml`** | Diferenças dev/prod controladas por `.env` e `-p <project>`. Menos arquivos para manter. | Dois composes (`dev`/`prod`) é overhead sem ganho para o escopo. |
| **Jenkins local com `docker.sock` montado** | Padrão de mercado para Jenkins em Docker buildando imagens. Simples e funcional. | Docker-in-Docker (DinD) e kaniko são mais "puros" mas adicionam atrito de configuração. |
| **Deploy via `compose up` no mesmo host do Jenkins** | Zero setup de SSH/firewall/DNS para a demo. README documenta como seria via SSH em produção real. | Servidor remoto exigiria infraestrutura adicional fora do escopo de demonstração. |
| **GHCR como registry** | Gratuito, integrado ao GitHub, autenticação via PAT direto nas credenciais do Jenkins. | Docker Hub tem limites de pull; Harbor/Nexus self-hosted é mais um container para manter. |
| **Trivy com `--severity HIGH,CRITICAL --ignore-unfixed`** | Quality gate real sem falsos positivos de CVEs sem correção disponível na base do Node. | Falhar em qualquer CVE deixa pipeline vermelho sem ação possível. |

---

## 5. Plano em fases

A implementação é dividida em **fases independentes**. Ao final de cada fase, o estado do repositório é funcional e demonstrável.

### Fase 0 — Preparação (≈30 min)

Trabalho de base que destrava todo o resto.

- [x] Adicionar `.gitignore` raiz cobrindo `node_modules`, `dist`, `.env`, `*.db`.
- [x] Adicionar `.env.example` em `backend/` documentando `JWT_SECRET`, `PORT`, `CORS_ORIGINS`, `DATABASE_URL` (já existia, completo).
- [x] Adicionar script `lint` aos `package.json` de backend e frontend (frontend já tinha; backend agora usa `tsc --noEmit`).
- [x] Registrar `backend/` e `frontend/` como **git submodules** do repo raiz (`EidrianXD/meeting-room-booking-{api,frontend}`).

**Critério de aceite:** `git status` limpo; `npm test` passa nos dois projetos a partir de uma instalação limpa. ✅

---

### Fase 1 — Containerização do backend (≈2h)

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

### Fase 2 — Containerização do frontend (≈1h30)

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

### Fase 3 — Orquestração local com Docker Compose (≈1h)

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

### Fase 4 — Pipeline Jenkins (≈3h)

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

**Ponto conhecido (não bloqueante, candidato a polimento na Fase 7):** o `docker-compose.yml` da raiz declara `name: distrimed-backend-data` para o volume, então dev (`-p distrimed`) e prod (`-p distrimed-prod`) compartilham o mesmo volume físico. Compose alerta com `volume already exists but was created for project "distrimed"`. Para isolamento real entre dev/prod, remover o `name:` (deixar derivar do project name) ou marcar `external: true`.

---

### Fase 5 — Quality gate de segurança refinado (≈30 min)

- [ ] Configuração do Trivy:
  - Falha apenas em `HIGH`/`CRITICAL`.
  - Ignora CVEs sem fix disponível (`--ignore-unfixed`).
  - Gera relatório SARIF arquivado como artefato do build (`archiveArtifacts`).
- [ ] Adicionar timeout global no pipeline (15 min) para evitar builds órfãos.

**Critério de aceite:** Trivy roda em ambas as imagens e publica o relatório, falhando apenas em CVEs corrigíveis HIGH/CRITICAL.

---

### Fase 6 (opcional) — Graduação para PostgreSQL

Executar **somente** após Fase 5 estável.

- [ ] Trocar `provider = "sqlite"` para `"postgresql"` em `schema.prisma`.
- [ ] Adicionar serviço `postgres:16-alpine` no `docker-compose.yml` com:
  - Volume nomeado para `pgdata`.
  - Healthcheck `pg_isready`.
  - Senha lida de `.env` (gerada via `openssl rand`).
- [ ] Backend espera healthy do Postgres via `depends_on`.
- [ ] Entrypoint do backend roda `prisma migrate deploy` antes do `node`.
- [ ] Reescrever migration inicial (`prisma migrate dev --name init`) para sintaxe Postgres.
- [ ] Atualizar README e `.env.example`.

**Critério de aceite:** mesmo fluxo da Fase 3 funcionando, agora com Postgres no lugar do SQLite, com persistência atravessando restart do container.

---

### Fase 7 (opcional) — Polimentos

Itens que elevam a percepção do trabalho mas não são bloqueantes.

- [ ] Badge de status do build do Jenkins no README.
- [ ] Versionamento semântico das imagens (`v1.0.0`) via tag git.
- [ ] Documentar como migrar o deploy local para um VPS via SSH (sem implementar).
- [ ] Adicionar Makefile com atalhos (`make up`, `make test`, `make logs`).

---

## 6. Estrutura final do repositório (alvo)

```
distrimed/
├── README.md                          ← este arquivo
├── .env.example
├── .gitignore
├── docker-compose.yml
├── Jenkinsfile
├── docker/
│   └── jenkins/
│       └── docker-compose.yml         ← Jenkins local
├── backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── .env.example
│   ├── src/...
│   └── prisma/...
└── frontend/
    ├── Dockerfile
    ├── .dockerignore
    ├── nginx.conf
    └── src/...
```

---

## 7. Riscos conhecidos e mitigações

| Risco | Mitigação |
|---|---|
| Jenkins em container precisa buildar Docker → exige `docker.sock` | Decisão consciente, documentada. Em produção real, usar agente remoto ou BuildKit. |
| SQLite no container é considerado smell em produção | Documentado como decisão de Fase 1; Fase 6 entrega Postgres. |
| Trivy pode falhar dia 1 com CVEs da base do Node | Configurado desde o início com `--ignore-unfixed` e severidade mínima HIGH. |
| Frontend precisa saber URL do backend em build-time (SPA estática) | Resolvido com `ARG VITE_API_URL` no Dockerfile, injetado pelo Jenkins por ambiente. |
| Race condition entre `migrate deploy` e múltiplas instâncias do backend | Não relevante para escala de demo (1 instância). Em produção, migration roda como job separado antes do deploy. |

---

## 8. Como acompanhar o progresso

Cada fase será implementada em um commit ou conjunto pequeno de commits com a referência `Fase N — <descrição>`. Os critérios de aceite de cada fase definem quando ela pode ser dada como concluída.

Próximo passo: começar pela **Fase 0**.
