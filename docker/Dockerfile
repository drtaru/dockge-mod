############################################
# Docker CLI
# The Docker CLI binary from Docker's official image.
# This avoids Docker's apt repository and
# the curl, gnupg, and unzip packages that it needed.
#
# buildx: `docker compose build` for stacks with a build section.
############################################
FROM docker:29.10.0-cli@sha256:34507eb5069d8be82c95dd3b6987d4250fd5fee49b82e2509b7d7a7030740d6d AS docker_cli

############################################
# Base
# Node.js on Debian 13, with Git's runtime libraries and dumb-init. The upgrade applies the
# Debian security updates that are newer than the Node.js image.
#
# git: stacks that are git checkouts (branch badge, Pull & Redeploy).
############################################
# Node.js publishes Debian 13 images for amd64 and arm64. armv7 uses
# Debian 12, with its security updates applied by the upgrade below.
# BuildKit builds only the stage of the target platform.
FROM node:22.23.5-trixie-slim@sha256:68c76cfad89921b927eae28ba21407475e29d0ce450afcdbe70c1532db042b64 AS node-amd64
FROM node:22.23.5-trixie-slim@sha256:68c76cfad89921b927eae28ba21407475e29d0ce450afcdbe70c1532db042b64 AS node-arm64
FROM node:22.23.5-bookworm-slim@sha256:e9f565d32859f0cbfb52125eafe8be39390f24dac302dc3cdf566e8ba3ab21ee AS node-arm
FROM node-${TARGETARCH} AS base
RUN apt-get update \
    && apt-get upgrade --yes --with-new-pkgs \
    && apt-get install --yes --no-install-recommends \
        ca-certificates \
        dumb-init \
        libbrotli1 \
        liberror-perl \
        libgnutls30 \
        libgssapi-krb5-2 \
        libidn2-0 \
        libnghttp2-14 \
        libpcre2-8-0 \
        libpsl5 \
        libssh2-1 \
        libzstd1 \
        perl \
        zlib1g \
    && rm -rf /var/lib/apt/lists/*
COPY --from=docker_cli /usr/local/bin/docker /usr/local/bin/docker

# Build Git and its patched HTTP and WebDAV libraries on the target Debian release.
FROM base AS build_git
RUN apt-get update \
    && apt-get install --yes --no-install-recommends \
        gcc gettext libc6-dev libbrotli-dev libgnutls28-dev libidn2-dev libkrb5-dev \
        libnghttp2-dev libpcre2-dev libpsl-dev libssh2-1-dev libzstd-dev make patch pkg-config xz-utils zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*
ADD --checksum=sha256:8dadd7d2536a314f982ad05a651b713f426bf7d02cc2c76cffb9796f2218e6c5 https://curl.se/download/curl-8.23.0.tar.xz /tmp/curl.tar.xz
WORKDIR /src/curl
RUN tar -xJf /tmp/curl.tar.xz --strip-components=1 \
    && ./configure --prefix=/usr/local --disable-static --disable-docs --disable-manual \
        --with-gnutls --with-gssapi --with-nghttp2 --with-libidn2 --with-libpsl \
        --with-libssh2 --with-brotli --with-zstd --with-ca-bundle=/etc/ssl/certs/ca-certificates.crt \
    && make -j"$(nproc)" install \
    && ldconfig \
    && curl-config --features | grep -qx GSS-API

ADD --checksum=sha256:c624ba2998fc65df1163a49641ff3f9ab73138b11641d92c86bee0fb01b1bc7f https://github.com/libexpat/libexpat/releases/download/R_2_10_0/expat-2.10.0.tar.xz /tmp/expat.tar.xz
# Expat 2.10.0 needs the upstream CVE-2026-124317 fix until the next release.
ADD --checksum=sha256:56b71deecec296aba42cd80ebde30215527aec9fa5c3ea183d0a056f2bb1520e https://github.com/libexpat/libexpat/commit/bf26290d2f021111cf806753dcb87d165f53571b.patch /tmp/expat-prepare.patch
ADD --checksum=sha256:ca8dc281de3b045d1ed1d0ac7fda819a31d8bb7bac3134391c809830d743de97 https://github.com/libexpat/libexpat/commit/3d77567612994172f9d09b46f31bb604e16fb08d.patch /tmp/expat-fix.patch
WORKDIR /src/expat
RUN tar -xJf /tmp/expat.tar.xz --strip-components=1 \
    && patch -p2 -i /tmp/expat-prepare.patch \
    && patch -p2 -i /tmp/expat-fix.patch \
    && ./configure --prefix=/usr/local --disable-static --without-docbook --without-tests --without-examples --without-xmlwf \
    && make -j"$(nproc)" install

ADD --checksum=sha256:f0b0bf89264b9bd2e1c640363e7529643989da76281600473c50745cd34a607f https://www.kernel.org/pub/software/scm/git/git-2.56.1.tar.xz /tmp/git.tar.xz
WORKDIR /src/git
RUN tar -xJf /tmp/git.tar.xz --strip-components=1 \
    && make -j"$(nproc)" prefix=/usr gitexecdir=/usr/lib/git-core DESTDIR=/out INSTALL_STRIP=-s \
        NO_RUST=1 NO_TCLTK=1 NO_OPENSSL=1 USE_LIBPCRE2=1 install \
    && mkdir -p /out/usr/local/lib \
    && cp -a /usr/local/lib/libcurl.so.4* /usr/local/lib/libexpat.so.1* /out/usr/local/lib/

############################################
# Healthcheck Binary
# Go compiles for the target architecture on the build platform, so
# this stage does not run under emulation.
############################################
FROM --platform=$BUILDPLATFORM golang:1.27.4-trixie@sha256:f18c3d73a184fb0582828510722e723b63a93f7188d11dd223d9be6f6880e65d AS go_builder
ENV GOTOOLCHAIN=local
FROM go_builder AS build_healthcheck
WORKDIR /app
ARG TARGETOS
ARG TARGETARCH
ARG TARGETVARIANT
COPY ./extra/healthcheck.go ./extra/healthcheck.go
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH GOARM=${TARGETVARIANT#v} \
    go build -trimpath -ldflags="-s -w" -o ./extra/healthcheck ./extra/healthcheck.go

# Use Docker's Compose release with patched dependencies until the CLI image
# includes fixes for CVE-2026-116591 and CVE-2026-78661.
FROM go_builder AS build_compose
WORKDIR /src
ARG TARGETOS
ARG TARGETARCH
ARG TARGETVARIANT
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
    compose_version=v5.7.1 \
    && go mod download "github.com/docker/compose/v5@$compose_version" \
    && cp -R "/go/pkg/mod/github.com/docker/compose/v5@$compose_version/." . \
    && chmod u+w go.mod go.sum \
    && go get github.com/klauspost/compress@v1.20.2 golang.org/x/net@v0.62.0 golang.org/x/crypto@v0.59.0 \
    && CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH GOARM=${TARGETVARIANT#v} \
        go build -mod=mod -trimpath -ldflags="-s -w -X github.com/docker/compose/v5/internal.Version=$compose_version" -o /docker-compose ./cmd

# Use the patched archive dependency until Docker's Buildx release includes it.
# Check the Docker daemon exclusions in docker/openvex.json against the linked packages.
FROM go_builder AS build_buildx
WORKDIR /src
ARG TARGETOS
ARG TARGETARCH
ARG TARGETVARIANT
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
    buildx_version=v0.39.1 \
    && go mod download "github.com/docker/buildx@$buildx_version" \
    && cp -R "/go/pkg/mod/github.com/docker/buildx@$buildx_version/." . \
    && chmod u+w go.mod go.sum \
    && go get github.com/moby/go-archive@v0.3.3 golang.org/x/crypto@v0.59.0 \
    && export CGO_ENABLED=0 GOOS="$TARGETOS" GOARCH="$TARGETARCH" GOARM="${TARGETVARIANT#v}" \
    && go list -mod=mod -deps ./cmd/buildx > /tmp/buildx-packages \
    && test "$(grep '^github.com/docker/docker/' /tmp/buildx-packages)" = github.com/docker/docker/pkg/namesgenerator \
    && go build -mod=mod -trimpath -ldflags="-s -w -X github.com/docker/buildx/version.Version=$buildx_version" -o /docker-buildx ./cmd/buildx

############################################
# Build
############################################
FROM base AS build
WORKDIR /app
ARG APP_VERSION
COPY --chown=node:node  ./package.json ./package.json
COPY --chown=node:node  ./package-lock.json ./package-lock.json
RUN npm ci --omit=dev
RUN node -e "require('node:sqlite'); require('@homebridge/node-pty-prebuilt-multiarch')"
# The release stage has no npm, so the version is set here
RUN if [ -n "$APP_VERSION" ]; then npm pkg set version="$APP_VERSION"; fi

############################################
# Frontend
# Built here so that `docker build` alone produces a working image. Without this
# stage the image starts and then exits on a missing frontend-dist/index.html.
# --platform=$BUILDPLATFORM: the output is arch-independent JS/CSS, so a
# multi-arch build must not recompile it per target under QEMU. This stage only
# needs Node.js, thus it starts from the plain node image.
############################################
FROM --platform=$BUILDPLATFORM node:22.23.5-trixie-slim@sha256:68c76cfad89921b927eae28ba21407475e29d0ce450afcdbe70c1532db042b64 AS frontend_build
WORKDIR /app
ARG APP_VERSION
COPY --chown=node:node  ./package.json ./package.json
COPY --chown=node:node  ./package-lock.json ./package-lock.json
RUN npm ci
# The interface shows this version, and it must agree with the backend
RUN if [ -n "$APP_VERSION" ]; then npm pkg set version="$APP_VERSION"; fi
COPY --chown=node:node  ./tsconfig.json ./tsconfig.json
COPY --chown=node:node  ./common ./common
COPY --chown=node:node  ./frontend ./frontend
RUN npm run build:frontend

############################################
# ⭐ Main Image
############################################
FROM base AS release
WORKDIR /app
# The server never runs npm. Removing it, and corepack, removes their
# bundled dependencies from the image.
RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/corepack \
        /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack
COPY --chown=node:node --from=build_healthcheck /app/extra/healthcheck /app/extra/healthcheck
COPY --from=build_compose /docker-compose /usr/local/libexec/docker/cli-plugins/docker-compose
COPY --from=build_buildx /docker-buildx /usr/local/libexec/docker/cli-plugins/docker-buildx
COPY --from=build_git /out/usr/ /usr/
COPY --from=build_git /src/curl/COPYING /usr/local/share/doc/curl/COPYING
COPY --from=build_git /src/expat/COPYING /usr/local/share/doc/expat/COPYING
RUN ldconfig \
    && ldd /usr/lib/git-core/git-http-push > /tmp/git-http-libraries \
    && grep -Fq 'libcurl.so.4 => /usr/local/lib/libcurl.so.4' /tmp/git-http-libraries \
    && grep -Fq 'libexpat.so.1 => /usr/local/lib/libexpat.so.1' /tmp/git-http-libraries \
    && ! grep -Fq 'not found' /tmp/git-http-libraries \
    && rm /tmp/git-http-libraries
COPY --from=build /app/node_modules /app/node_modules
# Only what the server needs at runtime. package.json comes from the build
# stage, which holds the version of the image.
COPY --chown=node:node --from=build /app/package.json ./package.json
COPY --chown=node:node ./tsconfig.json ./
COPY --chown=node:node ./backend ./backend
COPY --chown=node:node ./common ./common
COPY --chown=node:node ./extra/reset-password.ts ./extra/reset-password.ts
COPY --chown=node:node --from=frontend_build /app/frontend-dist /app/frontend-dist
# Dockge has tsx on the PATH. Keep that for compose files and commands that
# call it, pointing at the version of the lock file.
RUN mkdir ./data && ln -s /app/node_modules/.bin/tsx /usr/local/bin/tsx


# It is just for safe, as by default, it is disabled in the latest Node.js now.
# Read more:
# - https://github.com/sagemathinc/cocalc/issues/6963
# - https://github.com/microsoft/node-pty/issues/630#issuecomment-1987212447
ENV UV_USE_IO_URING=0

VOLUME /app/data
EXPOSE 5001
HEALTHCHECK --interval=60s --timeout=30s --start-period=60s --retries=5 CMD extra/healthcheck
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
# The tsx of the lock file, not a global install
CMD ["/app/node_modules/.bin/tsx", "./backend/index.ts"]
