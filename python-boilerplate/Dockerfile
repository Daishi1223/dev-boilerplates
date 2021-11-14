# syntax=docker/dockerfile:experimental
FROM python:3.9.8-slim-buster as base

# Install tini
ENV TINI_VERSION v0.19.0
RUN set -xe \
    && apt-get update && apt-get install -y --no-install-recommends curl \
    && ARCH=`dpkg --print-architecture` \
    && curl -sSLo /tini https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-${ARCH} \
    && chmod +x /tini \
    && rm -rf /var/lib/apt/lists/* /root/.cache

# Install poetry
ENV POETRY_HOME /srv/poetry
RUN set -xe \
    && curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python \
    && chmod +x $POETRY_HOME/bin/poetry \
    && ln -s $POETRY_HOME/bin/poetry /usr/local/bin/poetry

# Install production dependencies
WORKDIR /src
ADD ./pyproject.toml  /src/
ADD ./poetry.lock     /src/
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
    && poetry config virtualenvs.create false \
    # install python dependencies
    && poetry install --no-dev \
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
RUN --mount=type=ssh if [ "x$dev" != "xFALSE" ] ; then \
        # install dev tools
        apt-get update && apt-get install -y --no-install-recommends \
            # for reference
            # build-essential \
            git \
            sudo \
        # install python dev dependencies
        && poetry install; \
    fi

ENTRYPOINT ["/tini", "--"]
CMD tail -f /dev/null


# ----------------------------------------------------------------------------------


FROM base as executable

RUN set -xe \
    # setup user:ubuntu group:ubuntu
    && addgroup --gid 1000 ubuntu \
    && adduser --gid 1000 --uid 1000 ubuntu --disabled-password \
    && if [ "x$dev" != "xFALSE" ] ; then \
        # make user:ubuntu be able to use sudo
        echo "ubuntu ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/90-cloud-init-users \
        ; \
    fi

USER ubuntu

COPY --chown=ubuntu:ubuntu . /src
