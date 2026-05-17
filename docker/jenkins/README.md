# Jenkins local — Stack CI/CD do Distrimed

Stack autocontido do Jenkins controller para rodar o pipeline definido no
[`Jenkinsfile`](../../Jenkinsfile) da raiz do projeto.

## Primeira execução

```bash
cd docker/jenkins
cp .env.example .env
# editar .env: trocar JENKINS_ADMIN_PASSWORD por algo forte
docker compose up -d --build
```

A primeira execução é mais lenta porque o `jenkins-plugin-cli` baixa os plugins listados em `plugins.txt`. Depois, abra http://localhost:8081 e faça login com `JENKINS_ADMIN_ID` / `JENKINS_ADMIN_PASSWORD` definidos no `.env`.

> O setup wizard está desabilitado: o usuário admin é criado via
> [Configuration as Code](jenkins.yaml) (JCasC) no boot. Não há senha
> inicial nos logs — use a do `.env`.

## Configurar as credenciais (uma vez)

Em **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**, cadastrar:

| ID | Tipo | Valor |
|---|---|---|
| `ghcr-pat` | Username/Password | username = login GitHub (`EidrianXD`), password = PAT com escopos `write:packages`, `read:packages`, `delete:packages` |
| `jwt-secret-prod` | Secret text | JWT secret do deploy — gere com `openssl rand -hex 32` |
| `db-password-prod` | Secret text | Senha do Postgres do deploy — gere com `openssl rand -hex 24` |

## Criar o job do pipeline

**New Item → Pipeline → nome: `distrimed`**, depois:

- **Build Triggers** → **Poll SCM** → `H/1 * * * *` (já vem do `Jenkinsfile`, redundante mas seguro).
- **Pipeline → Definition: Pipeline script from SCM**
- **SCM: Git**
- **Repository URL:** `https://github.com/EidrianXD/meet-booking-room.git`
- **Branches to build:** `*/main`
- **Additional Behaviours → Advanced sub-modules behaviours**:
  - ☑ Recursively update submodules
  - ☑ Use credentials from default remote of parent repository (se algum dia os repos virarem privados)
- **Script Path:** `Jenkinsfile`

Salvar. Clicar em **Build Now** uma vez para validar.

## O pipeline

```
Checkout (com submodules)
  → Backend  : install + lint + test  (container node:20-slim)
  → Frontend : install + lint + test  (container node:22-slim)
  → Build images          (paralelo: back ∥ front)
  → Trivy scan            (paralelo: back ∥ front, HIGH/CRITICAL --ignore-unfixed)
  → Push GHCR             (apenas main)
  → Deploy local prod     (apenas main, em :8090 para não colidir com dev :8080)
```

Detalhes em [`../../Jenkinsfile`](../../Jenkinsfile).

## Operação

```bash
# logs
docker compose logs -f jenkins

# parar
docker compose down

# reset completo (apaga jobs, credenciais, histórico)
docker compose down -v

# rebuild da imagem (após mudar plugins.txt ou jenkins.yaml)
docker compose build --no-cache
```

## Decisões de design

- **DooD (Docker outside of Docker):** o container do Jenkins fala com o
  daemon do host via `/var/run/docker.sock`. Builds, scans e pushes
  rodam no Docker do host, não em um daemon aninhado. O container ganha
  acesso ao Docker do host (privilégio elevado equivalente a root). Em
  produção remota, usar agentes isolados ou kaniko.
- **Setup wizard desligado + JCasC:** o admin user vem do YAML, sem etapa
  manual de "Unlock Jenkins" / "Customize". Reprodutível e idempotente.
- **Polling SCM (1 min):** o Jenkins consulta o GitHub a cada minuto.
  Trade-off vs. webhook: latência maior, mas dispensa IP público / túnel
  ngrok.
- **Deploy em porta 8090 (não 8080):** evita colisão com `docker compose
  up` de desenvolvimento. O projeto de deploy se chama `distrimed-prod`,
  separado de qualquer `docker-compose` rodando manualmente.
