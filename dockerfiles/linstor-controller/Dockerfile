FROM ubuntu:bionic
ARG VERSION=
ENV DEBIAN_FRONTEND noninteractive

# Install repos and system upgrade
RUN apt-get -y update \
 && apt-get install -y wget gnupg \
 && apt-key adv --keyserver keyserver.ubuntu.com --recv-keys CC1B5A793C04BB3905AD837734893610CEAA9512 \
 && echo "deb http://ppa.launchpad.net/linbit/linbit-drbd9-stack/ubuntu bionic main" \
      > /etc/apt/sources.list.d/linbit.list \
 && apt-get update \
 && apt-get -y upgrade \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

# Install tini
ENV TINI_VERSION v0.18.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini
ENTRYPOINT ["/tini", "--"]

# Install linstor-controller
RUN apt-get update \
 && apt-get install -y linstor-controller linstor-client \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* \
 && mkdir -p /config /logs \
 && /usr/share/linstor-server/bin/linstor-config create-db-file \
      /data/linstordb > /config/linstor.toml

CMD [ "/usr/share/linstor-server/bin/Controller", "--logs=/logs", "--config-directory=/config" ]
