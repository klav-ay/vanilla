FROM mcr.microsoft.com/devcontainers/javascript-node:22-bullseye

# Install PostgreSQL client tools
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && apt-get -y install --no-install-recommends postgresql-client \
    && apt-get clean -y && rm -rf /var/lib/apt/lists/*

# Install global node packages
RUN su node -c "npm install -g prisma typescript ts-node"

# Set up project directories
RUN mkdir -p /workspaces/vanilla

# Configure project permissions
RUN chown -R node:node /workspaces/vanilla

# Switch to non-root user
USER node

# Set environment variables
ENV NODE_ENV=development \
    PATH=/workspaces/vanilla/node_modules/.bin:$PATH

# Set working directory
WORKDIR /workspaces/vanilla
