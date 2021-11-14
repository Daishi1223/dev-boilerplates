# syntax=docker/dockerfile:experimental
FROM golang:1.17.3-bullseye as builder

# Install dependencies
WORKDIR /src
# NOTE: Forward to host's ssh-agent for private repositories
RUN --mount=type=ssh set -xe \
    # setup SSH
    && mkdir -p ~/.ssh && chmod 700 ~/.ssh \
    && echo 'Host *' > ~/.ssh/config \
    && echo '  StrictHostkeyChecking no' >> ~/.ssh/config \
    && chmod 600 ~/.ssh/config \
    # install dev tools
    && savedAptMark="$(apt-mark showmanual)" \
    # && apt-get update && apt-get install -y --no-install-recommends \
      # for reference
      # build-essential \
    # install dependencies
    # remove dev tools
    && apt-mark auto '.*' > /dev/null \
    && apt-mark manual $savedAptMark \
    && find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec ldd '{}' ';' \
        | awk '/=>/ { print $(NF-1) }' \
        | sort -u \
        | xargs -r dpkg-query --search \
        | cut -d: -f1 \
        | sort -u \
        | xargs -r apt-mark manual \
  	&& apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && rm -rf /var/lib/apt/lists/* /root/.cache

# [Optional] Install dev dependencies
ARG dev=FALSE
RUN --mount=type=ssh if [ "x$dev" == "xFALSE" ] ; then \
        echo "dry-run"; \
    fi

CMD tail -f /dev/null


# ----------------------------------------------------------------------------------


FROM bullseye-slim as executable

ENV TINI_VERSION v0.19.0
RUN set -xe \
    # setup user:ubuntu group:ubuntu
    && addgroup --gid 1000 ubuntu \
    && adduser --gid 1000 --uid 1000 ubuntu --disabled-password \
    # install tini
    && apt-get update && apt-get install -y --no-install-recommends curl \
    && ARCH=`dpkg --print-architecture` \
    && curl -sSLo /tini https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-${ARCH} \
    && chmod +x /tini \
    && rm -rf /var/lib/apt/lists/* /root/.cache

USER ubuntu
