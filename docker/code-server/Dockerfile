FROM codercom/code-server:latest

# Install additional tools and extensions
USER root

# Install Node.js, Git, GitHub CLI and other development tools
RUN apt-get update && apt-get install -y \
    git \
    curl \
    wget \
    sudo \
    ca-certificates \
    gnupg \
    && mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" | tee /etc/apt/sources.list.d/nodesource.list \
    && curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | gpg --dearmor -o /etc/apt/keyrings/githubcli.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list \
    && apt-get update \
    && apt-get install -y nodejs gh \
    && npm install -g npm@latest \
    && rm -rf /var/lib/apt/lists/*

# Create workspace directory
RUN mkdir -p /home/coder/workspace && \
    chown -R coder:coder /home/coder/workspace

USER coder

# Install useful VS Code extensions
RUN code-server --install-extension dbaeumer.vscode-eslint && \
    code-server --install-extension esbenp.prettier-vscode && \
    code-server --install-extension bradlc.vscode-tailwindcss && \
    code-server --install-extension ms-vscode.vscode-typescript-next && \
    code-server --install-extension mikestead.dotenv && \
    code-server --install-extension eamodio.gitlens && \
    code-server --install-extension auchenberg.vscode-browser-preview

WORKDIR /home/coder/workspace

EXPOSE 8080

# Start code-server
CMD ["code-server", "--bind-addr", "0.0.0.0:8080", "--auth", "none", "/home/coder/workspace"]
