# This file was generated using a Jinja2 template.
# Please make your changes in `Dockerfile.j2` and then `make` the individual Dockerfile's.

# Using multistage build:
# 	https://docs.docker.com/develop/develop-images/multistage-build/
# 	https://whitfin.io/speeding-up-rust-docker-builds/
####################### VAULT BUILD IMAGE  #######################
FROM rust:1.40 as vault

ENV VAULT_VERSION "v2.12.0d"

ENV URL "https://github.com/dani-garcia/bw_web_builds/releases/download/$VAULT_VERSION/bw_web_$VAULT_VERSION.tar.gz"

# Build time options to avoid dpkg warnings and help with reproducible builds.
ENV DEBIAN_FRONTEND=noninteractive LANG=C.UTF-8 TZ=UTC TERM=xterm-256color

WORKDIR /

SHELL ["/bin/bash", "-o", "nounset", "-o", "pipefail", "-o", "errexit", "-c"]

RUN curl -L $URL | tar xz
RUN ls

########################## BUILD IMAGE  ##########################
# We need to use the Rust build image, because
# we need the Rust compiler and Cargo tooling
FROM rust:1.40 as build

# set sqlite as default for DB ARG for backward compatibility
ARG DB=sqlite

# Build time options to avoid dpkg warnings and help with reproducible builds.
ENV DEBIAN_FRONTEND=noninteractive LANG=C.UTF-8 TZ=UTC TERM=xterm-256color

# Don't download rust docs
RUN rustup set profile minimal

# Install required build libs for armel architecture.
RUN sed 's/^deb/deb-src/' /etc/apt/sources.list > \
        /etc/apt/sources.list.d/deb-src.list \
    && dpkg --add-architecture armel \
    && apt-get update \
    && apt-get install -y \
        --no-install-recommends \
        libssl-dev:armel \
        libc6-dev:armel

RUN apt-get update \
    && apt-get install -y \
        --no-install-recommends \
        gcc-arm-linux-gnueabi \
    && mkdir -p ~/.cargo \
    && echo '[target.arm-unknown-linux-gnueabi]' >> ~/.cargo/config \
    && echo 'linker = "arm-linux-gnueabi-gcc"' >> ~/.cargo/config

ENV CARGO_HOME "/root/.cargo"
ENV USER "root"

# Creates a dummy project used to grab dependencies
RUN USER=root cargo new --bin /app
WORKDIR /app

# Copies over *only* your manifests and build files
COPY ./Cargo.* ./
COPY ./rust-toolchain ./rust-toolchain
COPY ./build.rs ./build.rs

ENV CC_arm_unknown_linux_gnueabi="/usr/bin/arm-linux-gnueabi-gcc"
ENV CROSS_COMPILE="1"
ENV OPENSSL_INCLUDE_DIR="/usr/include/arm-linux-gnueabi"
ENV OPENSSL_LIB_DIR="/usr/lib/arm-linux-gnueabi"
RUN rustup target add arm-unknown-linux-gnueabi

# Builds your dependencies and removes the
# dummy project, except the target folder
# This folder contains the compiled dependencies
RUN cargo build --features ${DB} --release
RUN find . -not -path "./target*" -delete

# Copies the complete project
# To avoid copying unneeded files, use .dockerignore
COPY . .

# Make sure that we actually build the project
RUN touch src/main.rs

# Builds again, this time it'll just be
# your actual source files being built
RUN cargo build --features ${DB} --release --target=arm-unknown-linux-gnueabi

######################## RUNTIME IMAGE  ########################
# Create a new stage with a minimal image
# because we already have a binary built
FROM balenalib/rpi-debian:buster

ENV ROCKET_ENV "staging"
ENV ROCKET_PORT=80
ENV ROCKET_WORKERS=10

RUN [ "cross-build-start" ]

# Install needed libraries
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    openssl \
    ca-certificates \
    curl \
    sqlite3 \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir /data

RUN [ "cross-build-end" ]

VOLUME /data
EXPOSE 80
EXPOSE 3012

# Copies the files from the context (Rocket.toml file and web-vault)
# and the binary from the "build" stage to the current stage
COPY Rocket.toml .
COPY --from=vault /web-vault ./web-vault
COPY --from=build /app/target/arm-unknown-linux-gnueabi/release/bitwarden_rs .

COPY docker/healthcheck.sh ./healthcheck.sh

HEALTHCHECK --interval=30s --timeout=3s CMD sh healthcheck.sh || exit 1

# Configures the startup!
WORKDIR /
CMD ["/bitwarden_rs"]
