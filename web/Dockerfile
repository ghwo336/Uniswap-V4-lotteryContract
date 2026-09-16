# syntax=docker/dockerfile:1

# ---------- build ----------
FROM node:20-alpine AS build
WORKDIR /app

# wagmi 의존성(ws → bufferutil/utf-8-validate)이 네이티브 빌드를 요구합니다.
# 최종 이미지는 nginx 라 이 툴체인은 남지 않습니다.
RUN apk add --no-cache python3 make g++

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

# Vite 는 VITE_* 값을 빌드 타임에 번들로 인라인합니다.
# 따라서 .env 가 없으면 조용히 "데모 모드" 이미지가 만들어집니다. 그 전에 실패시킵니다.
RUN test -f .env || { \
      echo "ERROR: web/.env 가 없습니다. 먼저 'npm run sync-env' 를 실행하세요." >&2; exit 1; }
RUN grep -Eq '^VITE_HOOK_ADDRESS=0x[0-9a-fA-F]{40}$' .env || { \
      echo "ERROR: web/.env 의 VITE_HOOK_ADDRESS 가 비어있습니다. 데모 모드로 빌드됩니다." >&2; exit 1; }

RUN npm run build

# ---------- serve ----------
FROM nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
  CMD wget -qO- http://127.0.0.1/healthz || exit 1
