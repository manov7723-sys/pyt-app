# syntax=docker/dockerfile:1

# ---- Stage 1: build the static bundle ----
FROM --platform=linux/amd64 node:22-slim AS build

WORKDIR /app

COPY package.json package-lock.json ./
# --include=optional keeps per-platform native binaries (SWC etc.) in the tree
RUN npm ci --include=optional

COPY . .
RUN npm run build

# ---- Stage 2: serve the bundle ----
FROM --platform=linux/amd64 node:22-slim AS runtime

# Static file server installed at build time — nothing is fetched at runtime.
RUN npm install -g serve@14.2.4 && npm cache clean --force

WORKDIR /app
COPY --from=build /app/build ./build

# readOnlyRootFilesystem: /tmp is the only writable path, so point HOME there
# (serve's update-check would otherwise try to write under ~/.config).
ENV NODE_ENV=production \
    HOME=/tmp \
    NO_UPDATE_NOTIFIER=1

EXPOSE 3000

USER 1000

# *-slim has no curl/wget; probe with node itself.
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD node -e "require('http').get('http://127.0.0.1:3000/', r => process.exit(r.statusCode < 500 ? 0 : 1)).on('error', () => process.exit(1))"

CMD ["serve", "-s", "build", "-l", "3000"]
