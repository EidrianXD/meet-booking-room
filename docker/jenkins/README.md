# Jenkins local — Stack CI/CD do Distrimed

Stack autocontido do Jenkins controller para rodar o pipeline definido no
[`Jenkinsfile`](../../Jenkinsfile) da raiz do projeto.

Este guia leva qualquer pessoa do zero — sem nenhuma config prévia do GitHub
ou Jenkins — até ver o pipeline rodando verde no próprio computador.

---

## Sumário

1. [Pré-requisitos](#1-pré-requisitos)
2. [Criar um Personal Access Token (PAT) do GitHub](#2-criar-um-personal-access-token-pat-do-github)
3. [Subir o Jenkins](#3-subir-o-jenkins)
4. [Cadastrar as 3 credenciais](#4-cadastrar-as-3-credenciais)
5. [Criar o job `distrimed`](#5-criar-o-job-distrimed)
6. [Disparar a primeira build](#6-disparar-a-primeira-build)
7. [Operação](#operação)
8. [Decisões de design](#decisões-de-design)
9. [Troubleshooting](#troubleshooting)

---

## 1. Pré-requisitos

- **Docker Desktop** instalado e rodando.
- **Conta no GitHub** com acesso de leitura ao repo (público, qualquer conta serve).
- O repo já clonado localmente:

  ```bash
  git clone --recurse-submodules https://github.com/EidrianXD/meet-booking-room.git
  cd meet-booking-room
  ```

> O Jenkins **não precisa estar instalado** — vamos subir um Jenkins isolado em container.

---

## 2. Criar um Personal Access Token (PAT) do GitHub

O Jenkins precisa de um PAT para **publicar imagens no GHCR (GitHub Container Registry)** sob seu próprio usuário. Cada operador usa o seu — não há credencial compartilhada.

### 2.1 Abrir a tela de tokens

Acesse: https://github.com/settings/tokens

Ou navegue: avatar (canto superior direito) → **Settings** → **Developer settings** (fim do menu esquerdo) → **Personal access tokens** → **Tokens (classic)**.

> Use **"Tokens (classic)"**, NÃO "Fine-grained tokens" — o GHCR ainda tem suporte limitado a fine-grained.

### 2.2 Gerar token

Clique em **Generate new token → Generate new token (classic)**.

| Campo | Valor recomendado |
|---|---|
| **Note** | `distrimed-jenkins-ghcr` (qualquer nome descritivo) |
| **Expiration** | `90 days` |
| **Scopes** | ☑ `write:packages` &nbsp; ☑ `read:packages` &nbsp; ☑ `delete:packages` |

> Não precisa marcar `repo` — os repositórios do Distrimed são públicos.

Clique em **Generate token** no fim da página.

### 2.3 Copiar o token

GitHub mostra o token (`ghp_...`) **uma única vez**. Copie agora e cole em local temporário seguro (gerenciador de senhas).

⚠️ **NUNCA cole o token em arquivo do projeto, chat, e-mail ou qualquer lugar versionado.** Se vazar, revogue imediatamente na mesma tela.

---

## 3. Subir o Jenkins

```bash
cd docker/jenkins
cp .env.example .env
```

Edite `docker/jenkins/.env`:

```env
JENKINS_ADMIN_ID=admin
JENKINS_ADMIN_PASSWORD=defina-uma-senha-forte-aqui
JENKINS_PORT=8081
```

> Use uma senha forte para `JENKINS_ADMIN_PASSWORD`. Esta é a senha do admin do seu Jenkins local.

Suba o container:

```bash
docker compose up -d --build
```

A primeira execução é mais lenta (baixa os plugins listados em `plugins.txt`). Quando terminar, abra http://localhost:8081 e faça login com `JENKINS_ADMIN_ID` / `JENKINS_ADMIN_PASSWORD` que você definiu.

> O Jenkins **não pede senha pelos logs**. A senha é a que está no seu `.env` — não há setup wizard, o admin é criado via [Configuration as Code](jenkins.yaml).

---

## 4. Cadastrar as 3 credenciais

Vá em **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**.

Crie as 3 credenciais abaixo. **Os IDs precisam ser exatamente os indicados** — o Jenkinsfile referencia esses nomes.

### 4.1 `ghcr-pat` (PAT do GitHub)

| Campo | Valor |
|---|---|
| **Kind** | Username with password |
| **Scope** | Global |
| **Username** | Seu login do GitHub (ex: `JoaoSilva`) |
| **Password** | O PAT que você gerou no passo 2 (`ghp_...`) |
| **ID** | `ghcr-pat` ← exato |
| **Description** | `PAT do GHCR — push/pull de imagens` |

> O **Username desta credencial** determina automaticamente o GHCR owner onde as imagens serão publicadas (`ghcr.io/<username-lowercase>/distrimed-*`). Você não precisa configurar isso em mais lugar nenhum.

### 4.2 `jwt-secret-prod` (JWT do deploy)

| Campo | Valor |
|---|---|
| **Kind** | Secret text |
| **Scope** | Global |
| **Secret** | Gere com: `openssl rand -hex 32` |
| **ID** | `jwt-secret-prod` ← exato |
| **Description** | `JWT_SECRET do deploy local prod` |

Para gerar a string no PowerShell (sem `openssl`):

```powershell
$bytes = New-Object byte[] 32
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
($bytes | ForEach-Object { $_.ToString("x2") }) -join ""
```

### 4.3 `db-password-prod` (senha do Postgres)

| Campo | Valor |
|---|---|
| **Kind** | Secret text |
| **Scope** | Global |
| **Secret** | Gere com: `openssl rand -hex 24` (192 bits) |
| **ID** | `db-password-prod` ← exato |
| **Description** | `Senha do Postgres do deploy local prod` |

---

## 5. Criar o job `distrimed`

**New Item → Pipeline → Item name: `distrimed` → OK**

Na tela de configuração, preencha:

- **Pipeline → Definition:** `Pipeline script from SCM`
- **SCM:** `Git`
- **Repository URL:** `https://github.com/EidrianXD/meet-booking-room.git`
- **Credentials:** deixe em branco (repo é público)
- **Branches to build:** `*/main`
- **Additional Behaviours → Add → Advanced sub-modules behaviours:**
  - ☑ Recursively update submodules
- **Script Path:** `Jenkinsfile`

Os Build Triggers não precisam ser marcados na UI — o próprio Jenkinsfile já declara `pollSCM('H/1 * * * *')` (Jenkins consulta o repo a cada minuto).

Clique em **Save**.

---

## 6. Disparar a primeira build

Na página do job, clique em **Build Now**.

O Stage View vai mostrar cada etapa:

```
Checkout → Backend tests → Frontend tests → Build images → Trivy scan
       → Push GHCR → Deploy local prod
```

Tempo típico da primeira build: alguns minutos (baixar imagens base, instalar plugins, baixar Trivy DB). Builds seguintes são mais rápidas.

### O que esperar ao final

- **Imagens publicadas** no seu GHCR: https://github.com/SEU_USUARIO?tab=packages
- **Aplicação rodando em prod** no seu próprio computador: http://localhost:8090
- Login: `john` / `123456`

---

## Operação

```bash
# logs do Jenkins
docker compose logs -f jenkins

# parar o Jenkins (preserva jobs e credenciais)
docker compose down

# reset completo (APAGA jobs, credenciais e histórico)
docker compose down -v

# rebuild da imagem (após mudar plugins.txt ou jenkins.yaml)
docker compose build --no-cache
```

---

## Decisões de design

- **DooD (Docker outside of Docker):** o container do Jenkins fala com o
  daemon do host via `/var/run/docker.sock`. Builds, scans e pushes
  rodam no Docker do host, não em um daemon aninhado. O container ganha
  acesso ao Docker do host (privilégio elevado equivalente a root). Em
  produção remota, usar agentes isolados ou kaniko.
- **Setup wizard desligado + JCasC:** o admin user vem do YAML, sem etapa
  manual de "Unlock Jenkins" / "Customize". Reprodutível e idempotente.
- **Polling SCM (1 min):** o Jenkins consulta o GitHub a cada minuto.
  Trade-off vs. webhook: latência maior, mas dispensa IP público / túnel.
- **Deploy em porta 8090 (não 8080):** evita colisão com `docker compose
  up` de desenvolvimento. O projeto de deploy se chama `distrimed-prod`,
  separado de qualquer `docker-compose` rodando manualmente.
- **GHCR owner derivado da credencial:** o Jenkinsfile lê o username de
  `ghcr-pat` em runtime (lowercase) e usa como dono do namespace de
  imagens. Permite que qualquer operador rode o pipeline no próprio
  ambiente sem editar código.

---

## Troubleshooting

### "denied: denied" / "denied: permission_denied" no push do GHCR

Verifique se o PAT (`ghcr-pat`) tem os 3 escopos: `write:packages`,
`read:packages`, `delete:packages`. Se faltar `write:packages`, regenere o
token no GitHub e atualize a credencial.

### "Connection refused" no docker.sock dentro do Jenkins

O container do Jenkins precisa pertencer ao grupo `root` (ou ao grupo do
docker.sock do host). Isso já está no Dockerfile (`usermod -aG root
jenkins`). Se mesmo assim houver erro, faça:

```bash
docker exec --user root distrimed-jenkins chmod 666 /var/run/docker.sock
```

### Pipeline não dispara após `git push`

O polling SCM tem jitter de até 1 min. Para forçar manualmente:

```bash
# Via UI: na página do job, clicar "Build Now".
# Via API: ver "Disparar build via API" no Jenkinsfile.
```

### Esqueci a senha do admin

A senha está no seu `docker/jenkins/.env`. Se você apagou:

```bash
docker compose down -v   # APAGA TODOS os dados do Jenkins
# edite .env com nova senha
docker compose up -d --build
```

### Trivy reclama de CVEs novos depois de meses sem rebuildar

Bases Debian/Alpine recebem CVEs novos com fix disponível ao longo do tempo.
Como o Dockerfile faz `apt-get upgrade` / `apk upgrade` apenas no build,
re-rodar a pipeline em uma data futura pega os patches automaticamente.
