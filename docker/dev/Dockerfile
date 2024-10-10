# https://hub.docker.com/_/almalinux
ARG VARIANT="8.10"
FROM almalinux:${VARIANT}

RUN dnf update -y \
    && dnf install -y epel-release \
    && dnf install -y --enablerepo=epel \
        glibc-langpack-ja \
        glibc-locale-source \
        procps \
        sudo \
        git \
        openssh-server \
        supervisor \
        wget \
        zip \
        nano \ 
        htop

# ロケール及びタイムゾーンを日本に設定
RUN localedef -f UTF-8 -i ja_JP ja_JP.UTF-8 \
    && echo 'LANG=ja_JP.UTF-8' > /etc/locale.conf \
    && ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
ENV LANG="ja_JP.UTF-8" \
    LANGUAGE="ja_JP:ja" \
    LC_ALL="ja_JP.UTF-8" \
    TZ="Asia/Tokyo"

# VSCode CLI をインストール
ARG VSCODE_VERSION="stable" \
    VSCODE_INSTALL_DIR="/usr/local/bin" \
    VSCODE_DATA_ROOT_DIR="/vscode"
ARG VSCODE_SERVER_DATA_DIR="${VSCODE_DATA_ROOT_DIR}/server-data" \
    VSCODE_USER_DATA_DIR="${VSCODE_DATA_ROOT_DIR}/user-data" \
    VSCODE_EXTENSIONS_DIR="${VSCODE_DATA_ROOT_DIR}/extensions" \
    VSCODE_CLI_DATA_DIR="${VSCODE_DATA_ROOT_DIR}/cli-data"
COPY vscode_install_extension.sh /
RUN mkdir -p "${VSCODE_SERVER_DATA_DIR}" \
    && mkdir -p "${VSCODE_USER_DATA_DIR}" \
    && mkdir -p "${VSCODE_EXTENSIONS_DIR}" \
    && mkdir -p "${VSCODE_CLI_DATA_DIR}" \
    && wget "https://code.visualstudio.com/sha/download?build=${VSCODE_VERSION}&os=cli-alpine-x64" -O /tmp/vscode_cli.tar.gz \
    && tar -xf /tmp/vscode_cli.tar.gz -C /tmp \
    && mv /tmp/code "${VSCODE_INSTALL_DIR}" \
    && rm /tmp/vscode_cli.tar.gz \
    && chmod +x /vscode_install_extension.sh
COPY settings.json "${VSCODE_SERVER_DATA_DIR}/data/Machine/"

# ----------------------------------------------------------------------------------------------------

# Rust のインストール
ARG CARGO_HOME=/usr/local/cargo \
    RUSTUP_HOME=/usr/local/rustup
RUN dnf install -y \
        gcc \
    && export CARGO_HOME=${CARGO_HOME} \
    && export RUSTUP_HOME=${RUSTUP_HOME} \
    && curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
    && export PATH="${CARGO_HOME}/bin:${RUSTUP_HOME}:\$PATH" \
    && rustup default stable \
    && rustup component add rust-src \
    && echo "export PATH=\$PATH:${CARGO_HOME}/bin:${RUSTUP_HOME}" > /etc/profile.d/rust_path.sh \
    && echo "export CARGO_HOME=${CARGO_HOME}/bin:${RUSTUP_HOME}" >> /etc/profile.d/rust_path.sh \
    && echo "export RUSTUP_HOME=${RUSTUP_HOME}" >> /etc/profile.d/rust_path.sh

# Rust 開発用の拡張機能をインストール
RUN /vscode_install_extension.sh "${VSCODE_EXTENSIONS_DIR}" rust-lang rust-analyzer 0.3.2137 linux-x64
RUN /vscode_install_extension.sh "${VSCODE_EXTENSIONS_DIR}" vadimcn vscode-lldb 1.11.0 linux-x64

# ----------------------------------------------------------------------------------------------------

# 一般ユーザー追加 
ARG USER_NAME \
    USER_PASSWORD \
    USER_UID \
    USER_GID
RUN groupadd --gid ${USER_GID} ${USER_NAME} \
    && useradd \
        --shell /bin/bash \
        --create-home \
        --home-dir /home/${USER_NAME} \
        --uid ${USER_UID} \
        --gid ${USER_GID} \
        ${USER_NAME} \
    && echo "${USER_NAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${USER_NAME} \
    && chmod 0440 /etc/sudoers.d/${USER_NAME} \
    && echo "${USER_NAME}:${USER_PASSWORD}" | chpasswd

# VSCode のデータの保存先にアクセス許可を設定 ※TODO: どう設定するのが適切？
RUN chmod -R 777 "${VSCODE_DATA_ROOT_DIR}"

# Entrypoint
COPY entrypoint.d /entrypoint.d
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh \
    && chmod +x -R /entrypoint.d

# Supervisor
ENV VSCODE_USER="${USER_NAME}" \
    VSCODE_SERVER_DATA_DIR="${VSCODE_SERVER_DATA_DIR}" \
    VSCODE_USER_DATA_DIR="${VSCODE_USER_DATA_DIR}" \
    VSCODE_EXTENSIONS_DIR="${VSCODE_EXTENSIONS_DIR}" \
    VSCODE_CLI_DATA_DIR="${VSCODE_CLI_DATA_DIR}"
COPY supervisord.conf /etc/supervisor/
RUN mkdir -p /var/run/supervisor /var/log/supervisor \
    && chmod -R 755 /var/run/supervisor /var/log/supervisor

# SSH
RUN mkdir -p /etc/ssh/ssh_host_keys \
    && sed -i 's|HostKey /etc/ssh/|HostKey /etc/ssh/ssh_host_keys/|' /etc/ssh/sshd_config

# 起動時の設定
EXPOSE 22
ENTRYPOINT ["sudo", "-E", "/entrypoint.sh"]
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
USER ${USER_NAME}
WORKDIR /home/${USER_NAME}
