# Docker container for Debian 8.2 (Jessie)

FROM debian:8.2
MAINTAINER ceej@couchbase.com

USER root

# Install SSH server
RUN apt-get update && \
    apt-get install -y openssh-server sudo curl && \
    rm -f /var/cache/apt/archives/*.deb && \
    mkdir /var/run/sshd # update 3

# Create couchbase user with password-less sudo privs, and give
# ownership of /opt/couchbase
RUN useradd couchbase -G sudo -m -s /bin/bash && \
    mkdir -p /opt/couchbase && chown -R couchbase:couchbase /opt/couchbase && \
    echo 'couchbase:couchbase' | chpasswd && \
    sed -ri 's/ALL\) ALL/ALL) NOPASSWD:ALL/' /etc/sudoers

# Jenkins Swarm jar
# https://unix.stackexchange.com/questions/342403/openjdk-8-jre-headless-depends-ca-certificates-java-but-it-is-not-going-to-be
RUN echo 'deb http://ftp.debian.org/debian jessie-backports main' >> /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -t jessie-backports -y openjdk-8-jre-headless ca-certificates-java && \
    apt-get clean
RUN curl \
    https://repo.jenkins-ci.org/releases/org/jenkins-ci/plugins/swarm-client/3.13/swarm-client-3.13.jar \
    -o /usr/local/lib/swarm-client.jar

# tini, to avoid zombie issues when running with Swarm jar
ENV TINI_VERSION v0.18.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini

# Install Couchbase build dependencies.
# Note: lsb-release currently required for "cbdeps" third-party build
# mechanism, but we hope to remove that requirement as it brings in
# a lot of unnecessary packages.

RUN apt-get update && \
    apt-get install -y \
        ccache \
        git-core \
        tar \
        libssl-dev \
        libnuma-dev \
        rubygems \
        rake \
        ncurses-dev \
        python-dev \
        devscripts \
        debhelper \
        dh-systemd \
        ed \
        man \
        pixz \
        lsb-release && \
    apt-get clean

# GCC (from couchbasebuild/debian-8-gcc)
COPY build/local /usr/local
RUN  (echo "/usr/local/lib64"; cat /etc/ld.so.conf) > /tmp/ld.so.conf && \
     mv /tmp/ld.so.conf /etc && ldconfig

#paramiko for testrunner
RUN apt-get update && apt-get install -y python-paramiko && apt-get clean

# pygerrit (for toy builds)
RUN apt-get install -y python-pip && pip install pygerrit2

# Install autoconf and friends - necessary for building some third-party deps
# from source, not for Couchbase.
RUN apt-get update && \
    apt-get install -y m4 autoconf automake libtool && \
    apt-get clean

RUN echo 'PATH="/usr/lib/ccache:$PATH"' >> /home/couchbase/.profile
RUN mkdir /tmp/deploy && \
    curl https://cmake.org/files/v3.6/cmake-3.6.1-Linux-x86_64.sh -o /tmp/deploy/cmake.sh && \
    (echo y; echo n) | sh /tmp/deploy/cmake.sh --prefix=/usr/local && \
    curl https://storage.googleapis.com/git-repo-downloads/repo -o /usr/local/bin/repo && \
    chmod a+x /usr/local/bin/repo && \
    cd /tmp && rm -rf /tmp/deploy

# Docker Swarm properties - keep this towards end of Dockerfile
COPY build/swarm*.properties /

# Expose SSH daemon and run our builder startup script
EXPOSE 22
ADD .ssh /home/couchbase/.ssh
COPY build/couchbuilder_start.sh /usr/sbin/
ENTRYPOINT [ "/tini", "--", "/usr/sbin/couchbuilder_start.sh" ]
CMD [ "swarm" ]

