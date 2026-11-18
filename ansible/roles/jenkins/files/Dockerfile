FROM jenkins/jenkins:lts-jdk17
USER root
RUN apt-get update && apt-get install -y ca-certificates curl gnupg \
 && install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
 && chmod a+r /etc/apt/keyrings/docker.asc \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list \
 && apt-get update \
 && apt-get install -y docker-ce-cli docker-compose-plugin \
 && rm -rf /var/lib/apt/lists/*
RUN apt-get update \
 && apt-get install -y python3 python3-venv \
 && rm -rf /var/lib/apt/lists/*
RUN python3 -m venv /opt/ansible \
 && /opt/ansible/bin/pip install --no-cache-dir ansible-core kubernetes PyYAML jsonpatch \
 && /opt/ansible/bin/ansible-galaxy collection install kubernetes.core community.docker --collections-path /usr/share/ansible/collections
ENV PATH="/opt/ansible/bin:${PATH}"
RUN curl -fsSL --retry 5 --retry-all-errors --retry-delay 3 -o /usr/local/bin/kubectl "https://dl.k8s.io/release/$(curl -fsSL --retry 5 --retry-all-errors https://dl.k8s.io/release/stable.txt)/bin/linux/$(dpkg --print-architecture)/kubectl" \
 && chmod 0755 /usr/local/bin/kubectl
