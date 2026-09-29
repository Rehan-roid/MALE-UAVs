# =============================================================================
# Twin-Piston Aero Engine Digital Twin — Frontend Production Image
#
# This app is a TanStack Start (Nitro) SSR application, NOT a static SPA, so the
# runtime stage serves a generated Node server bundle rather than plain files.
#
# VITE_* values are inlined by Vite at BUILD time and are never read at runtime,
# so they must be passed as build arguments:
#
#   docker build \
#     --build-arg VITE_API_BASE_URL=https://api.example.com/api/v1 \
#     --build-arg VITE_WS_URL=wss://api.example.com/api/v1/ws/engine \
#     -t twin-piston-frontend .
# =============================================================================

# --- Stage 1: Build -------------------------------------------------------
FROM node:22-alpine AS build

WORKDIR /app

# Dependency layer, cached independently of source changes.
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund

COPY . .

# Baked into the client bundle by Vite. Defaults target the local backend.
ARG VITE_API_BASE_URL=http://localhost:8000/api/v1
ARG VITE_WS_URL=ws://localhost:8000/api/v1/ws/engine
ENV VITE_API_BASE_URL=$VITE_API_BASE_URL \
    VITE_WS_URL=$VITE_WS_URL

# The default Nitro preset is a Cloudflare Worker (cloudflare-module). For
# container hosting we need a standalone Node HTTP server instead.
ENV NITRO_PRESET=node-server
RUN npm run build

# --- Stage 2: Runtime -----------------------------------------------------
FROM node:22-alpine AS runtime

WORKDIR /app

RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001

# The Nitro bundle is self-contained: dependencies are vendored into
# .output/server/_libs and public assets are copied into .output/public, so no
# node_modules or npm install is required at runtime.
COPY --from=build --chown=nodejs:nodejs /app/.output ./

ENV NODE_ENV=production \
    HOST=0.0.0.0 \
    PORT=3000

USER nodejs
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD node -e "fetch('http://127.0.0.1:'+process.env.PORT+'/').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

CMD ["node", "server/index.mjs"]
