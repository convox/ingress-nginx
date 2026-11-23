FROM --platform=$BUILDPLATFORM golang:1.26.11-alpine3.23@sha256:db99415e2625ce0ee03eed09edf7f32ae8c58dc399f28cc602a87fe7238119e4 AS build

ARG TARGETARCH
ARG REF=controller-v1.15.1
ARG COMMIT=0a5901f3c64f11e92e487799b8da3f00cca37515

ENV GOTOOLCHAIN=local CGO_ENABLED=0 GOOS=linux

RUN apk add --no-cache git
RUN git clone --depth 1 --branch "$REF" https://github.com/kubernetes/ingress-nginx /src && \
  test "$(git -C /src rev-parse HEAD)" = "$COMMIT"

WORKDIR /src
RUN go get golang.org/x/net@v0.62.0 golang.org/x/text@v0.43.0 && go mod tidy

RUN for t in nginx:nginx-ingress-controller dbg:dbg waitshutdown:wait-shutdown; do \
    GOARCH=$TARGETARCH go build -trimpath -buildvcs=false \
      -ldflags="-buildid= -w -s -X k8s.io/ingress-nginx/version.RELEASE=v1.15.1 -X k8s.io/ingress-nginx/version.COMMIT=$COMMIT -X k8s.io/ingress-nginx/version.REPO=https://github.com/kubernetes/ingress-nginx" \
      -o /out/${t#*:} ./cmd/${t%%:*}; \
  done

FROM alpine:3.23.3@sha256:25109184c71bdad752c8312a8623239686a9a2071e8825f20acb8f2198c3f659 AS nginx

COPY nginx/build.sh /build.sh
COPY nginx/patches /patches
RUN apk update && apk upgrade && apk add -U bash --no-cache && bash /build.sh

FROM alpine:3.23.3@sha256:25109184c71bdad752c8312a8623239686a9a2071e8825f20acb8f2198c3f659

LABEL org.opencontainers.image.title="NGINX Ingress Controller for Kubernetes"
LABEL org.opencontainers.image.source="https://github.com/convox/ingress-nginx"
LABEL org.opencontainers.image.licenses="Apache-2.0"
LABEL org.opencontainers.image.version="v1.15.1"

ENV PATH=$PATH:/usr/local/luajit/bin:/usr/local/nginx/sbin:/usr/local/nginx/bin
ENV LUA_PATH="/usr/local/share/luajit-2.1.0-beta3/?.lua;/usr/local/share/lua/5.1/?.lua;/usr/local/lib/lua/?.lua;;"
ENV LUA_CPATH="/usr/local/lib/lua/?/?.so;/usr/local/lib/lua/?.so;;"

COPY --from=nginx /usr/local /usr/local
COPY --from=nginx /usr/lib/libopentelemetry* /usr/local/lib
COPY --from=nginx /opt /opt
COPY --from=nginx /etc/nginx /etc/nginx

RUN apk update && apk upgrade && apk add -U --no-cache \
    bash \
    openssl \
    pcre \
    zlib \
    ca-certificates \
    patch \
    yajl \
    lmdb \
    libxml2 \
    libmaxminddb \
    yaml-cpp \
    dumb-init \
    tzdata \
    grpc-cpp \
    libprotobuf \
    abseil-cpp-crc-cpu-detect \
    abseil-cpp-vlog-config-internal \
    diffutils && \
  ln -s /usr/local/nginx/sbin/nginx /sbin/nginx && \
  adduser -S -D -H -u 101 -h /usr/local/nginx -s /sbin/nologin -G www-data -g www-data www-data && \
  for dir in \
    /var/log/nginx \
    /var/lib/nginx/body \
    /var/lib/nginx/fastcgi \
    /var/lib/nginx/proxy \
    /var/lib/nginx/scgi \
    /var/lib/nginx/uwsgi \
    /var/log/audit \
    /etc/ingress-controller \
    /etc/ingress-controller/ssl \
    /etc/ingress-controller/auth \
    /etc/ingress-controller/geoip \
    /etc/ingress-controller/telemetry \
    /var/log \
    /tmp/nginx; do \
    mkdir -p "$dir" && chown -R www-data:www-data "$dir" || exit 1; \
  done && \
  echo "/lib:/usr/lib:/usr/local/lib:/modules_mount/etc/nginx/modules/otel" > /etc/ld-musl-x86_64.path

EXPOSE 80 443

WORKDIR /etc/nginx

COPY --from=build --chown=www-data:www-data /src/rootfs/etc /etc
COPY --from=build --chown=www-data:www-data /out/nginx-ingress-controller /out/dbg /out/wait-shutdown /

RUN apk add --no-cache libcap && \
  setcap cap_net_bind_service=+ep /nginx-ingress-controller && \
  setcap -v cap_net_bind_service=+ep /nginx-ingress-controller && \
  setcap cap_net_bind_service=+ep /usr/local/nginx/sbin/nginx && \
  setcap -v cap_net_bind_service=+ep /usr/local/nginx/sbin/nginx && \
  setcap cap_net_bind_service=+ep /usr/bin/dumb-init && \
  setcap -v cap_net_bind_service=+ep /usr/bin/dumb-init && \
  apk del libcap && \
  ln -sf /usr/local/nginx/sbin/nginx /usr/bin/nginx

USER www-data

RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
  ln -sf /dev/stderr /var/log/nginx/error.log

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["/nginx-ingress-controller"]
