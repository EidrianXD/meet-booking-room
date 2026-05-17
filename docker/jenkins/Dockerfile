# syntax=docker/dockerfile:1.7
#
# Imagem do Jenkins controller customizada:
#   - Plugins pré-instalados (plugins.txt)
#   - Docker CLI disponível (para usar o socket do host via DooD)
#   - Setup wizard desativado (configuração reprodutível)
#
# Pattern "Docker outside of Docker" (DooD): o container usa o socket do
# host (/var/run/docker.sock montado), de modo que `docker build` etc.
# rodam no daemon do host, não em um daemon aninhado.

# Usa o tag LTS corrente (atualizado pelo upstream do Jenkins) — alguns
# plugins listados em plugins.txt exigem versões recentes do controller.
# Pin específico pode quebrar quando a comunidade lança updates dos plugins.
FROM jenkins/jenkins:lts-jdk21

USER root

# Docker CLI + Compose plugin (instalados via repositório oficial Docker).
# O daemon NÃO roda aqui — só o cliente fala com o socket do host.
RUN apt-get update -y \
 && apt-get install -y --no-install-recommends \
        ca-certificates curl gnupg lsb-release \
 && install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL https://download.docker.com/linux/debian/gpg \
        | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
 && chmod a+r /etc/apt/keyrings/docker.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
         https://download.docker.com/linux/debian $(lsb_release -cs) stable" \
        > /etc/apt/sources.list.d/docker.list \
 && apt-get update -y \
 && apt-get install -y --no-install-recommends \
        docker-ce-cli docker-compose-plugin \
 && rm -rf /var/lib/apt/lists/*

# Acesso ao socket Docker do host. No Docker Desktop (Windows/macOS) o
# socket aparece como root:root 660 dentro do container — adicionar o
# usuário `jenkins` ao grupo root é a forma mais simples e cross-platform
# de habilitar `docker ps/build/push` a partir do pipeline. Tradeoff:
# group root dá acesso a outros arquivos root-owned do container. Em
# ambiente local de Jenkins isolado, esse risco é aceitável e
# documentado em docker/jenkins/README.md.
RUN usermod -aG root jenkins

USER jenkins

# Pré-instala plugins listados em plugins.txt.
COPY --chown=jenkins:jenkins plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

# Copia o YAML do JCasC. O Jenkins lê esse path no boot e aplica a config
# (admin user, autorização, etc.) — sem precisar do wizard.
COPY --chown=jenkins:jenkins jenkins.yaml /var/jenkins_home/casc/jenkins.yaml

# Desativa o setup wizard inicial; o JCasC faz o bootstrap completo.
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false" \
    CASC_JENKINS_CONFIG="/var/jenkins_home/casc/jenkins.yaml"
