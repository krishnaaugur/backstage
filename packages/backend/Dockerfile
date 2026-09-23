# Multi-stage Dockerfile for Backstage
# Stage 1: Build — installs all deps and compiles the backend bundle inside Docker
# Stage 2: Runtime — lean production image with only compiled artifacts
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

# Copy yarn config first for better layer caching
COPY .yarn ./.yarn
COPY .yarnrc.yml yarn.lock package.json backstage.json ./

# Copy all workspace package.json files so yarn can resolve the full monorepo graph
# This includes both packages/ and plugins/
COPY packages/ ./packages/
COPY plugins/ ./plugins/

# Install ALL dependencies (dev + prod) needed for the build
RUN --mount=type=cache,target=/root/.cache/yarn,sharing=locked \
    yarn install --immutable

# Copy full source (node_modules, dist, etc. are excluded via .dockerignore)
COPY . .

# Build the backend bundle — this compiles TypeScript internally, no separate yarn tsc needed
RUN yarn build:backend --config ../../app-config.yaml

# ---- Stage 2: Runtime ----
FROM node:22-bookworm-slim AS runtime

# Install runtime native module dependencies
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends libsqlite3-dev && \
    rm -rf /var/lib/apt/lists/*

# Run as the least-privileged node user
USER node
WORKDIR /app

# Copy yarn config from build context
COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml backstage.json ./

ENV NODE_ENV=production

# Copy yarn.lock + package.json from context
COPY --chown=node:node yarn.lock package.json ./

# Copy skeleton.tar.gz from build stage (contains workspace package.json structure)
COPY --from=build --chown=node:node /app/packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

# Install only production dependencies
RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn workspaces focus --all --production && rm -rf "$(yarn cache clean)"

# Copy example catalog data (referenced by default app-config.yaml)
COPY --chown=node:node examples ./examples

# Copy app configs from build context
COPY --chown=node:node app-config*.yaml ./

# Copy the compiled backend bundle from the build stage
COPY --from=build --chown=node:node /app/packages/backend/dist/bundle.tar.gz ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

EXPOSE 7007

CMD ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
