# Multi-stage Dockerfile for Backstage
# Stage 1: Build — installs all deps and compiles the app
# Stage 2: Runtime — lean production image with only what's needed
#
# Build context must be the repo root:
#   docker build -f packages/backend/Dockerfile -t backstage .

# ---- Stage 1: Build ----
FROM node:22-bookworm-slim AS build

# Install build tools needed for native modules (e.g. better-sqlite3)
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends \
      python3 \
      g++ \
      make \
      libsqlite3-dev && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy yarn config and lockfile first for better layer caching
COPY .yarn ./.yarn
COPY .yarnrc.yml yarn.lock package.json backstage.json ./

# Copy all package.json files (workspaces) so yarn can resolve the graph
COPY packages/app/package.json ./packages/app/package.json
COPY packages/backend/package.json ./packages/backend/package.json

# Install ALL dependencies (dev + prod) needed for the build
RUN --mount=type=cache,target=/root/.cache/yarn,sharing=locked \
    yarn install --immutable

# Copy the full source
COPY . .

# Build TypeScript types, then the backend bundle
RUN yarn tsc
RUN yarn build:backend --config ../../app-config.yaml

# ---- Stage 2: Runtime ----
FROM node:22-bookworm-slim AS runtime

# Install runtime deps for native modules
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends libsqlite3-dev && \
    rm -rf /var/lib/apt/lists/*

# Run as the least-privileged node user
USER node
WORKDIR /app

# Copy yarn config
COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml backstage.json ./

ENV NODE_ENV=production

# Copy repo skeleton + lockfile to install production deps only
COPY --chown=node:node yarn.lock package.json packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn workspaces focus --all --production && rm -rf "$(yarn cache clean)"

# Copy example catalog files (referenced by default app-config)
COPY --chown=node:node examples ./examples

# Copy the compiled backend bundle and both app-configs
COPY --chown=node:node packages/backend/dist/bundle.tar.gz app-config*.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

EXPOSE 7007

CMD ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
