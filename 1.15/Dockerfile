FROM --platform=$BUILDPLATFORM golang:1.26.10-alpine3.23@sha256:0525c952defebc693cdaf62cdf43afa32be5a7f377e99260642ea19c240c708e AS build

ARG TARGETARCH
ARG REF=controller-v1.15.1
ARG COMMIT=0a5901f3c64f11e92e487799b8da3f00cca37515

ENV GOTOOLCHAIN=local CGO_ENABLED=0 GOOS=linux

RUN apk add --no-cache git
RUN git clone --depth 1 --branch "$REF" https://github.com/kubernetes/ingress-nginx /src && \
  test "$(git -C /src rev-parse HEAD)" = "$COMMIT"

WORKDIR /src
RUN go get golang.org/x/net@v0.61.0 golang.org/x/text@v0.43.0 && go mod tidy

RUN for t in nginx:nginx-ingress-controller dbg:dbg waitshutdown:wait-shutdown; do \
    GOARCH=$TARGETARCH go build -trimpath -buildvcs=false \
      -ldflags="-buildid= -w -s -X k8s.io/ingress-nginx/version.RELEASE=v1.15.1 -X k8s.io/ingress-nginx/version.COMMIT=$COMMIT -X k8s.io/ingress-nginx/version.REPO=https://github.com/kubernetes/ingress-nginx" \
      -o /out/${t#*:} ./cmd/${t%%:*}; \
  done

FROM registry.k8s.io/ingress-nginx/controller:v1.15.1@sha256:594ceea76b01c592858f803f9ff4d2cb40542cae2060410b2c95f75907d659e1

USER root
RUN apk upgrade --no-cache
COPY --from=build --chown=www-data:www-data /out/nginx-ingress-controller /out/dbg /out/wait-shutdown /
RUN apk add --no-cache libcap && \
  setcap cap_net_bind_service=+ep /nginx-ingress-controller && \
  setcap -v cap_net_bind_service=+ep /nginx-ingress-controller && \
  setcap cap_net_bind_service=+ep /usr/local/nginx/sbin/nginx && \
  setcap -v cap_net_bind_service=+ep /usr/local/nginx/sbin/nginx && \
  setcap cap_net_bind_service=+ep /usr/bin/dumb-init && \
  setcap -v cap_net_bind_service=+ep /usr/bin/dumb-init && \
  apk del libcap
USER www-data
