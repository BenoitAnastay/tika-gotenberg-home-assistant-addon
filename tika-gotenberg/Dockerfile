ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:7.7.1
ARG GOLANG_VERSION="1.24"

# The following part is a copy past of the Dockerfile from the Gotenberg project

# ----------------------------------------------
# pdfcpu binary build stage
# ----------------------------------------------
# Note: this stage is required as pdfcpu does not release an armhf variant by
# default.

FROM golang:$GOLANG_VERSION AS pdfcpu-binary-stage

ARG PDFCPU_VERSION=v0.8.1
ENV CGO_ENABLED=0

# Define the working directory outside of $GOPATH (we're using go modules).
WORKDIR /home

RUN curl -Ls "https://github.com/pdfcpu/pdfcpu/archive/refs/tags/$PDFCPU_VERSION.tar.gz" -o pdfcpu.tar.gz &&\
    tar --strip-components=1 -xvzf pdfcpu.tar.gz

# Install module dependencies.
RUN go mod download &&\
    go mod verify

RUN go build -o pdfcpu -ldflags "-s -w -X 'main.version=$PDFCPU_VERSION' -X 'github.com/pdfcpu/pdfcpu/pkg/pdfcpu.VersionStr=$PDFCPU_VERSION' -X main.builtBy=gotenberg" ./cmd/pdfcpu &&\
    # Verify installation.
    ./pdfcpu version

# ----------------------------------------------
# Gotenberg binary build stage
# ----------------------------------------------
FROM golang:$GOLANG_VERSION AS gotenberg-binary-stage

ARG GOTENBERG_VERSION="v8.18.0"
ENV CGO_ENABLED=0

# Define the working directory outside of $GOPATH (we're using go modules).
WORKDIR /home

RUN curl -Ls "https://github.com/gotenberg/gotenberg/archive/refs/tags/$GOTENBERG_VERSION.tar.gz" -o gotenberg.tar.gz &&\
    tar --strip-components=1 -xvzf gotenberg.tar.gz

RUN go mod download &&\
    go mod verify

RUN go build -o gotenberg -ldflags "-X 'github.com/gotenberg/gotenberg/v8/cmd.Version=$GOTENBERG_VERSION'" cmd/gotenberg/main.go

FROM debian:bookworm AS fetch_tika

ARG CHECK_SIG=true
ARG TIKA_VERSION="3.1.0"

ENV NEAREST_TIKA_SERVER_URL="https://dlcdn.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar" \
    ARCHIVE_TIKA_SERVER_URL="https://archive.apache.org/dist/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar" \
    BACKUP_TIKA_SERVER_URL="https://downloads.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar" \
    DEFAULT_TIKA_SERVER_ASC_URL="https://downloads.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar.asc" \
    ARCHIVE_TIKA_SERVER_ASC_URL="https://archive.apache.org/dist/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar.asc" \
    TIKA_VERSION=$TIKA_VERSION

SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN DEBIAN_FRONTEND=noninteractive apt-get update && apt-get --no-install-recommends -y install  gnupg2=2.2.40-1.1 \
    wget=1.21.3-1+deb12u1 \
    ca-certificates=20230311 \
    && wget --progress=dot:giga  -t 10 --max-redirect 1 --retry-connrefused -qO- https://downloads.apache.org/tika/KEYS | gpg --import \
    && wget --progress=dot:giga  -t 10 --max-redirect 1 --retry-connrefused $NEAREST_TIKA_SERVER_URL -O /tika-server-standard.jar || rm /tika-server-standard-${TIKA_VERSION}.jar \
    && sh -c "[ -f /tika-server-standard-${TIKA_VERSION}.jar ]" || wget --progress=dot:giga  $ARCHIVE_TIKA_SERVER_URL -O /tika-server-standard-${TIKA_VERSION}.jar || rm /tika-server-standard-${TIKA_VERSION}.jar \
    && sh -c "[ -f /tika-server-standard-${TIKA_VERSION}.jar ]" || wget --progress=dot:giga  $BACKUP_TIKA_SERVER_URL -O /tika-server-standard-${TIKA_VERSION}.jar || rm /tika-server-standard-${TIKA_VERSION}.jar \
    && sh -c "[ -f /tika-server-standard-${TIKA_VERSION}.jar ]" || exit 1 \
    && wget --progress=dot:giga  -t 10 --max-redirect 1 --retry-connrefused $DEFAULT_TIKA_SERVER_ASC_URL -O /tika-server-standard-${TIKA_VERSION}.jar.asc  || rm /tika-server-standard-${TIKA_VERSION}.jar.asc \
    && sh -c "[ -f /tika-server-standard-${TIKA_VERSION}.jar.asc ]" || wget --progress=dot:giga  $ARCHIVE_TIKA_SERVER_ASC_URL -O /tika-server-standard-${TIKA_VERSION}.jar.asc || rm /tika-server-standard-${TIKA_VERSION}.jar.asc \
    && sh -c "[ -f /tika-server-standard-${TIKA_VERSION}.jar.asc ]" || exit 1 \
    && gpg --verify /tika-server-standard-${TIKA_VERSION}.jar.asc /tika-server-standard-${TIKA_VERSION}.jar \
    && mv /tika-server-standard-${TIKA_VERSION}.jar /tika-server-standard.jar


# ----------------------------------------------
# Final stage
# ----------------------------------------------
# hadolint ignore=DL3006
FROM ${BUILD_FROM}

ARG GOTENBERG_VERSION
ARG GOTENBERG_USER_GID
ARG GOTENBERG_USER_UID
ARG NOTO_COLOR_EMOJI_VERSION
ARG PDFTK_VERSION="v3.3.3"

# Copy root filesystem
COPY rootfs/ /


RUN \
    # Create a non-root user.
    # All processes in the Docker container will run with this dedicated user.
    groupadd --gid 1000 gotenberg &&\
    useradd --uid 1000 --gid gotenberg --shell /bin/bash --home /home/gotenberg --no-create-home gotenberg &&\
    mkdir /home/gotenberg &&\
    chown gotenberg: /home/gotenberg &&\
    # Install system dependencies required for the next instructions or debugging.
    # Note: tini is a helper for reaping zombie processes.
    apt-get update -qq &&\
    apt-get upgrade -yqq &&\
    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends  \
    curl=7.88.1-10+deb12u12 \
    gnupg=2.2.40-1.1 \
    tini=0.19.0-1 \
    python3=3.11.2-1+b1 \
    default-jre-headless=2:1.17-74 \
    && \
    # Cleanup.
    # Note: the Debian image does automatically a clean after each install thanks to a hook.
    # Therefore, there is no need for apt-get clean.
    # See https://stackoverflow.com/a/24417119/3248473.
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* &&\
    # Install fonts.
    # Credits:
    # https://github.com/arachnys/athenapdf/blob/master/cli/Dockerfile.
    # https://help.accusoft.com/PrizmDoc/v12.1/HTML/Installing_Asian_Fonts_on_Ubuntu_and_Debian.html.
    curl -o ./ttf-mscorefonts-installer_3.8.1_all.deb http://httpredir.debian.org/debian/pool/contrib/m/msttcorefonts/ttf-mscorefonts-installer_3.8.1_all.deb &&\
    apt-get update -qq &&\
    apt-get upgrade -yqq &&\
    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
    ./ttf-mscorefonts-installer_3.8.1_all.deb \
    apt-utils=2.6.1 \
    culmus=0.133-1 \
    fonts-beng=2:1.3 \
    fonts-hosny-amiri=0.113-1 \
    fonts-lklug-sinhala=0.6-4 \
    fonts-lohit-guru=2.91.2-3 \
    fonts-lohit-knda=2.5.4-3 \
    fonts-samyak-gujr=1.2.2-6 \
    fonts-samyak-mlym=1.2.2-6 \
    fonts-samyak-taml=1.2.2-6 \
    fonts-sarai=1.0-3 \
    fonts-sil-abyssinica=2.100-3 \
    fonts-sil-padauk=5.000-3 \
    fonts-telu=2:1.3 \
    fonts-thai-tlwg=1:0.7.3-1 \
    fonts-wqy-zenhei=0.9.45-8\
    fonts-arphic-ukai=0.2.20080216.2-5 \
    fonts-arphic-uming=0.2.20080216.2-11 \
    fonts-ipafont-mincho=00303-23 \
    fonts-ipafont-gothic=00303-23 \
    fonts-unfonts-core=1:1.0.2-080608-18 \
    fonts-crosextra-caladea=20200211-1 \
    fonts-crosextra-carlito=20220224-1 \
    fonts-dejavu=2.37-6 \
    fonts-dejavu-extra=2.37-6 \
    fonts-liberation=1:1.07.4-11 \
    fonts-liberation2=2.1.5-1 \
    fonts-linuxlibertine=5.3.0-6 \
    fonts-noto-cjk=1:20220127+repack1-1 \
    fonts-noto-core=20201225-1 \
    fonts-noto-mono=20201225-1 \
    fonts-noto-ui-core=20201225-1 \
    fonts-sil-gentium=20081126:1.03-4 \
    fonts-sil-gentium-basic=1.102-1.1 \
    && \
    rm -f ./ttf-mscorefonts-installer_3.8.1_all.deb &&\
    # Add Color and Black-and-White Noto emoji font.
    # Credits:
    # https://github.com/gotenberg/gotenberg/pull/325.
    # https://github.com/googlefonts/noto-emoji.
    curl -Ls "https://github.com/googlefonts/noto-emoji/raw/$NOTO_COLOR_EMOJI_VERSION/fonts/NotoColorEmoji.ttf" -o /usr/local/share/fonts/NotoColorEmoji.ttf &&\
    # Cleanup.
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* &&\
    # Install either Google Chrome stable on amd64 architecture or
    # Chromium on other architectures.
    # See https://github.com/gotenberg/gotenberg/issues/328.
    /bin/bash -c \
    'set -e &&\
    if [[ "$(dpkg --print-architecture)" == "amd64" ]]; then \
      curl https://dl.google.com/linux/linux_signing_key.pub | apt-key add - &&\
      echo "deb http://dl.google.com/linux/chrome/deb/ stable main" | tee /etc/apt/sources.list.d/google-chrome.list &&\
      apt-get update -qq &&\
      apt-get upgrade -yqq &&\
      DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends --allow-unauthenticated google-chrome-stable &&\
      mv /usr/bin/google-chrome-stable /usr/bin/chromium; \
    else \
      apt-get update -qq &&\
      apt-get upgrade -yqq &&\
      DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends chromium; \
    fi' &&\
    # Verify installation.
    chromium --version &&\
    # Cleanup.
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* &&\
    # Install LibreOffice & unoconverter.
    echo "deb http://deb.debian.org/debian bookworm-backports main" >> /etc/apt/sources.list &&\
    apt-get update -qq &&\
    apt-get upgrade -yqq &&\
    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends -t bookworm-backports libreoffice=4:25.2.1-3~bpo12+1 &&\
    curl -Ls https://raw.githubusercontent.com/gotenberg/unoconverter/v0.1.1/unoconv -o /usr/bin/unoconverter &&\
    chmod +x /usr/bin/unoconverter &&\
    # unoconverter will look for the Python binary, which has to be at version 3.
    ln -s /usr/bin/python3 /usr/bin/python &&\
    # Verify installations.
    libreoffice --version &&\
    unoconverter --version &&\
    # Cleanup.
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* &&\
    # Install PDFtk, QPDF & ExifTool (PDF engines).
    # See https://github.com/gotenberg/gotenberg/pull/273.
    curl -o /usr/bin/pdftk-all.jar "https://gitlab.com/api/v4/projects/5024297/packages/generic/pdftk-java/$PDFTK_VERSION/pdftk-all.jar" &&\
    chmod a+x /usr/bin/pdftk-all.jar &&\
    chmod +x /usr/bin/pdftk &&\
    apt-get update -qq &&\
    apt-get upgrade -yqq &&\
    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends qpdf=11.3.0-1+deb12u1 libimage-exiftool-perl=12.57+dfsg-1 &&\
    # See https://github.com/nextcloud/docker/issues/380.
    mkdir -p /usr/share/man/man1 &&\
    # Verify installations.
    pdftk --version &&\
    qpdf --version &&\
    exiftool --version &&\
    # Cleanup.
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Improve fonts subpixel hinting and smoothing.
# Credits:
# https://github.com/arachnys/athenapdf/issues/69.
# https://github.com/arachnys/athenapdf/commit/ba25a8d80a25d08d58865519c4cd8756dc9a336d.
COPY --from=gotenberg-binary-stage /home/build/fonts.conf /etc/fonts/conf.d/100-gotenberg.conf

# Copy the pdfcpu binary from the pdfcpu-binary-stage.
COPY --from=pdfcpu-binary-stage /home/pdfcpu /usr/bin/

# Copy the Gotenberg binary from the gotenberg-binary-stage.
COPY --from=gotenberg-binary-stage /home/gotenberg /usr/bin/

COPY --from=fetch_tika /tika-server-standard.jar /usr/bin/tika-server-standard.jar

# Environment variables required by modules or else.
ENV CHROMIUM_BIN_PATH=/usr/bin/chromium
ENV LIBREOFFICE_BIN_PATH=/usr/lib/libreoffice/program/soffice.bin
ENV UNOCONVERTER_BIN_PATH=/usr/bin/unoconverter
ENV PDFTK_BIN_PATH=/usr/bin/pdftk
ENV QPDF_BIN_PATH=/usr/bin/qpdf
ENV EXIFTOOL_BIN_PATH=/usr/bin/exiftool
ENV PDFCPU_BIN_PATH=/usr/bin/pdfcpu

# USER gotenberg
# WORKDIR /home/gotenberg

# # Default API port.
# EXPOSE 3000

# ENTRYPOINT [ "/usr/bin/tini", "--" ]
# CMD [ "gotenberg" ]

# Build arguments
ARG BUILD_ARCH
ARG BUILD_DATE
ARG BUILD_DESCRIPTION
ARG BUILD_NAME
ARG BUILD_REF
ARG BUILD_REPOSITORY
ARG BUILD_VERSION

# Labels
LABEL \
    io.hass.name="${BUILD_NAME}" \
    io.hass.description="${BUILD_DESCRIPTION}" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version=${BUILD_VERSION} \
    maintainer="Benoit Anastay <benoit@anastay.dev>" \
    org.opencontainers.image.title="${BUILD_NAME}" \
    org.opencontainers.image.description="${BUILD_DESCRIPTION}" \
    org.opencontainers.image.vendor="Benoit Anastay Add-on" \
    org.opencontainers.image.authors="Benoit Anastay <benoit@anastay.dev>" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.url="https://github.com/BenoitAnastay/tika-gotenberg-home-assistant-addon" \
    org.opencontainers.image.source="https://github.com/${BUILD_REPOSITORY}" \
    org.opencontainers.image.documentation="https://github.com/${BUILD_REPOSITORY}/blob/main/README.md" \
    org.opencontainers.image.created=${BUILD_DATE} \
    org.opencontainers.image.revision=${BUILD_REF} \
    org.opencontainers.image.version=${BUILD_VERSION}