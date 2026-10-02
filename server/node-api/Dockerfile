# syntax=docker/dockerfile:1

# ---------------------------------------------------------------------------
# EVAT Node API (Express + TypeScript)
# Build context: server/node-api
# ---------------------------------------------------------------------------

# ------------------------------- build stage -------------------------------
FROM node:24-alpine AS build

WORKDIR /app

# Install ALL dependencies (including devDependencies).
# tsc, ts-node and @types/* packages are required during the build.
COPY package*.json ./
RUN npm install

# Copy TypeScript source and configuration.
COPY tsconfig.json ./
COPY server.ts ./
COPY src ./src

# Compile the TypeScript application.
RUN npm run build && ls -la /app/server.js

# ------------------------------ runtime stage ------------------------------
FROM node:24-alpine AS runtime

ENV NODE_ENV=production

WORKDIR /app

# Install production dependencies only.
COPY package*.json ./
RUN npm install --omit=dev && npm cache clean --force

# Copy compiled application from build stage.
COPY --from=build /app/server.js ./server.js
COPY --from=build /app/src ./src

# EVAT backend runs on port 8081 because Jenkins uses port 8080.
ENV PORT=8081

EXPOSE 8081

# Docker health check for the EVAT API.
# The Swagger endpoint is used to verify that Express is responding.
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://127.0.0.1:8081/api/docs/ || exit 1

# Run as non-root user for better container security.
USER node

# Start the compiled production application.
CMD ["node", "server.js"]