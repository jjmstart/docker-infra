# Jenkins LTS + JDK 21，与官方镜像主线对齐，便于获得安全补丁
FROM jenkins/jenkins:lts-jdk21

# 安装系统包与 pip 需要 root；装完后切回 jenkins 用户，避免长期以 root 跑进程
USER root

# 国内访问 Debian 官方源较慢，换成清华镜像；再装 Python/Ansible 与 CI 常用工具
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

# 官方镜像默认非 root，符合 Jenkins 安全实践
USER jenkins
