FROM golang:1.17-alpine AS builder

WORKDIR /src

RUN apk add --no-cache ca-certificates

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -trimpath \
    -ldflags="-s -w" \
    -o /out/momo-backend \
    ./cmd/api

FROM alpine:3.20

RUN addgroup -S app && adduser -S -G app -h /app app \
    && apk add --no-cache ca-certificates \
    && mkdir -p /app/data \
    && chown -R app:app /app

WORKDIR /app

COPY --from=builder --chown=app:app /out/momo-backend /app/momo-backend

USER app

EXPOSE 8081

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget -qO- http://127.0.0.1:8081/health >/dev/null || exit 1

ENTRYPOINT ["/app/momo-backend"]
