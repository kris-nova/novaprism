FROM ubuntu:20.10

LABEL maintainer="Michael Mayer <michael@liquidbytes.net>"

ARG BUILD_TAG

ENV DEBIAN_FRONTEND noninteractive

# Configure apt
RUN echo 'Acquire::Retries "10";' > /etc/apt/apt.conf.d/80retry && \
    echo 'APT::Install-Recommends "false";' > /etc/apt/apt.conf.d/80recommends && \
    echo 'APT::Install-Suggests "false";' > /etc/apt/apt.conf.d/80suggests && \
    echo 'APT::Get::Assume-Yes "true";' > /etc/apt/apt.conf.d/80forceyes && \
    echo 'APT::Get::Fix-Missing "true";' > /etc/apt/apt.conf.d/80fixmissin

# Install dev / build dependencies
RUN apt-get update && apt-get upgrade && \
    apt-get install \
    build-essential \
    curl \
    chrpath \
    libssl-dev \
    libxft-dev \
    libfreetype6 \
    libfreetype6-dev \
    libfontconfig1 \
    libfontconfig1-dev \
    libhdf5-serial-dev \
    libpng-dev \
    libzmq3-dev \
    libx264-dev \
    libx265-dev \
    pkg-config \
    software-properties-common \
    rsync \
    unzip \
    zip \
    g++ \
    gcc \
    libc6-dev \
    gpg-agent \
    apt-utils \
    make \
    nano \
    wget \
    git \
    gettext \
    mariadb-client \
    sqlite3 \
    tzdata \
    gconf-service \
    firefox \
    libheif-examples \
    exiftool \
    darktable \
    rawtherapee \
    ffmpeg \
    ffmpegthumbnailer \
    libavcodec-extra \
    lsof \
    apache2-utils \
    fonts-roboto \
    sudo

# Install & configure TensorFlow for C,
# see https://www.tensorflow.org/install/lang_c
#
# We also have custom builds available:
# https://dl.photoprism.org/tensorflow/linux/
#
ENV LD_LIBRARY_PATH /root/.local/lib:/usr/local/lib:/usr/lib:/lib
ENV TF_CPP_MIN_LOG_LEVEL 0
RUN curl -L \
   "https://dl.photoprism.org/tensorflow/linux/libtensorflow-linux-cpu-1.15.2.tar.gz" | \
   tar -C "/usr" -xz
RUN ldconfig

# Install NodeJS
RUN curl -sL https://deb.nodesource.com/setup_14.x | bash -
RUN add-apt-repository -y ppa:saiarcot895/chromium-beta && \
    apt-get update && \
    apt-get dist-upgrade && \
    apt-get install chromium-browser chromium-codecs-ffmpeg-extra nodejs && \
    apt-get autoremove && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Set default https://en.wikipedia.org/wiki/Umask
RUN umask 0000

# Install and configure NodeJS Package Manager (npm)
ENV NODE_ENV production
RUN npm install --unsafe-perm=true --allow-root -g npm testcafe chromedriver && \
    npm config set cache ~/.cache/npm

# Install Go
ENV GOLANG_VERSION 1.16
RUN set -eux; \
	\
	url="https://golang.org/dl/go${GOLANG_VERSION}.linux-amd64.tar.gz"; \
	wget -O go.tgz "$url"; \
	echo "013a489ebb3e24ef3d915abe5b94c3286c070dfe0818d5bca8108f1d6e8440d2 *go.tgz" | sha256sum -c -; \
	tar -C /usr/local -xzf go.tgz; \
	rm go.tgz; \
	export PATH="/usr/local/go/bin:$PATH"; \
	go version

# Configure Go environment
ENV GOPATH /go
ENV GOBIN $GOPATH/bin
ENV PATH $GOBIN:/usr/local/go/bin:~/.local/bin:/root/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ENV TMPDIR /tmp
ENV GO111MODULE on
ENV CGO_CFLAGS "-g -O2 -Wno-return-local-addr"
RUN mkdir -p "$GOPATH/src" "$GOPATH/bin" && chmod -R 777 "$GOPATH"

# Download TensorFlow model and test files
RUN rm -rf /tmp/* && mkdir -p /tmp/photoprism && \
    wget "https://dl.photoprism.org/tensorflow/nsfw.zip?${BUILD_TAG}" -O /tmp/photoprism/nsfw.zip && \
    wget "https://dl.photoprism.org/tensorflow/nasnet.zip?${BUILD_TAG}" -O /tmp/photoprism/nasnet.zip && \
    wget "https://dl.photoprism.org/qa/testdata.zip?${BUILD_TAG}" -O /tmp/photoprism/testdata.zip

# Install additional tools
COPY --chown=root:root --chmod=755 /docker/scripts/heif-convert.sh /usr/local/bin/heif-convert
RUN env GO111MODULE=off /usr/local/go/bin/go get -u github.com/psampaz/go-mod-outdated && \
    env GO111MODULE=off /usr/local/go/bin/go get -u golang.org/x/tools/cmd/goimports && \
    env GO111MODULE=off /usr/local/go/bin/go get -u github.com/tsliwowicz/go-wrk && \
    env GO111MODULE=off /usr/local/go/bin/go get -u github.com/kyoh86/richgo && \
    env GO111MODULE=off /usr/local/go/bin/go get -u github.com/tianon/gosu && \
    env GO111MODULE=off /usr/local/go/bin/go get -u github.com/dsoprea/go-exif/v3/command/exif-read-tool && \
    echo "alias go=richgo" > /root/.bash_aliases && \
    cp /go/bin/gosu /bin/gosu

# Create photoprism user and directory for deployment
RUN useradd photoprism -m -d /photoprism && \
    mkdir -p /var/lib/photoprism /tmp/photoprism && \
    echo "alias go=richgo" > /photoprism/.bash_aliases && \
    echo "photoprism ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers && \
    chown -Rf photoprism:photoprism /photoprism /var/lib/photoprism /tmp/photoprism && \
    chmod -Rf a+rw /photoprism /var/lib/photoprism /tmp/photoprism /go && \
    find /go -type d -print0 | xargs -0 chmod 777

# Copy mysql client config for development
COPY --chown=root:root /docker/development/.my.cnf /root/.my.cnf
COPY --chown=photoprism:photoprism /docker/development/.my.cnf /photoprism/.my.cnf
RUN chmod 644 /root/.my.cnf /photoprism/.my.cnf

# Set up project directory
WORKDIR "/go/src/github.com/photoprism/photoprism"

# Expose HTTP port 2342, 2343 plus 9515 for chromedriver
EXPOSE 2342 2343 9515
VOLUME /var/lib/photoprism

# Configure entrypoint
COPY --chown=root:root /docker/development/entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]

# Run server
CMD ["tail", "-f", "/dev/null"]