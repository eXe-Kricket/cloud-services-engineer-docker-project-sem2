# syntax=docker/dockerfile:1.7

FROM node:18-alpine AS builder

WORKDIR /app

ENV NODE_ENV=production
ENV NODE_OPTIONS=--openssl-legacy-provider

ARG VUE_APP_API_URL=/api
ENV VUE_APP_API_URL=${VUE_APP_API_URL}

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginxinc/nginx-unprivileged:1.27-alpine

COPY --from=builder --chown=101:101 /app/dist /usr/share/nginx/html/momo-store

COPY --chown=101:101 <<'EOF' /etc/nginx/conf.d/default.conf
server {
    listen 8080;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    location = / {
        return 302 /momo-store/;
    }

    location /momo-store/ {
        try_files $uri $uri/ /momo-store/index.html;
    }

    location /api/ {
        proxy_pass http://backend:8081/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget -qO- http://127.0.0.1:8080/momo-store/ >/dev/null || exit 1
