FROM jenkins/jenkins:lts-jdk21

USER root

RUN sed -i \
      -e 's|http://deb.debian.org/debian|https://mirrors.tuna.tsinghua.edu.cn/debian|g' \
      -e 's|http://security.debian.org/debian-security|https://mirrors.tuna.tsinghua.edu.cn/debian-security|g' \
      /etc/apt/sources.list.d/debian.sources \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
       python3 \
       python3-pip \
       python3-venv \
       openssh-client \
       ca-certificates \
       git \
    && python3 -m pip install --break-system-packages --no-cache-dir \
       -i https://pypi.tuna.tsinghua.edu.cn/simple \
       ansible \
    && rm -rf /var/lib/apt/lists/*

USER jenkins
