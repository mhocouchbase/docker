FROM ubuntu:22.04

LABEL maintainer="docker@couchbase.com"

ARG UPDATE_COMMAND="apt-get update -y -q"
ARG CLEANUP_COMMAND="rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*"

# Install dependencies:
#  runit: for container process management
#  wget: for downloading .deb
#  tzdata: timezone info used by some N1QL functions
# Additional dependencies for system commands used by cbcollect_info:
#  lsof: lsof
#  lshw: lshw
#  sysstat: iostat, sar, mpstat
#  net-tools: ifconfig, arp, netstat
#  numactl: numactl
RUN set -x \
    && ${UPDATE_COMMAND} \
    && apt-get install -y -q wget tzdata \
      lsof lshw sysstat net-tools numactl bzip2 \
    && ${CLEANUP_COMMAND}

# Add runit
RUN set -x \
    && apt-get update \
    && apt-get install -y gcc git make \
    && cd /usr/src \
    && git clone https://github.com/couchbasedeps/runit \
    && cd runit \
    && git checkout edb631449d89d5b452a5992c6ffaa1e384fea697 \
    && ./package/compile \
    && cp ./command/* /sbin/ \
    && apt-get purge -y --autoremove gcc git make \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /usr/src/runit

ARG CB_RELEASE_URL=https://packages.couchbase.com/releases/7.6.1
ARG CB_PACKAGE=couchbase-server-community_7.6.1-linux_@@ARCH@@.deb
ARG CB_SKIP_CHECKSUM=false
ENV PATH=$PATH:/opt/couchbase/bin:/opt/couchbase/bin/tools:/opt/couchbase/bin/install

# Create couchbase user/group with fixed UID/GID 1000 for consistency across environments
# (modifies existing user/group in images which already have UID/GID 1000 - e.g. ubuntu:24.04)
RUN set -x \
    && if getent group 1000 >/dev/null; then \
          existing_group=$(getent group 1000 | cut -d: -f1); \
          groupmod --new-name couchbase "${existing_group}"; \
       else \
          groupadd -g 1000 couchbase; \
       fi \
    && if getent passwd 1000 >/dev/null; then \
          existing_user=$(getent passwd 1000 | cut -d: -f1); \
          usermod --login couchbase -d /home/couchbase -m -g couchbase -s /bin/sh "${existing_user}"; \
       else \
          useradd couchbase -u 1000 -g couchbase -M -s /bin/sh; \
       fi

# Install couchbase
RUN \
    set -x \
    && ${UPDATE_COMMAND} \
    && export INSTALL_DONT_START_SERVER=1 \
    && dpkgArch="$(dpkg --print-architecture)" \
    && case "${dpkgArch}" in \
         'arm64') \
           CB_SHA256=abfd2a4d57c930be72d501af1e54612e06d9a73faf948df549c342252f1d3e49 \
           ;; \
         'amd64') \
           CB_SHA256=c1e48a4175ed7f82532a1b2b858f0c2af08752ab83b4d084cb16d59f52437a82 \
           ;; \
       esac \
    && CB_PACKAGE=$(echo ${CB_PACKAGE} | sed -e "s/@@ARCH@@/${dpkgArch}/") \
    && wget -N --no-verbose $CB_RELEASE_URL/$CB_PACKAGE \
    && { ${CB_SKIP_CHECKSUM} || echo "$CB_SHA256  $CB_PACKAGE" | sha256sum -c - ; } \
    && apt-get install -y ./$CB_PACKAGE \
    && rm -f ./$CB_PACKAGE \
    && ${CLEANUP_COMMAND} \
    && rm -rf /tmp/* /var/tmp/*

# Update VARIANT.txt to indicate we're running in our Docker image
RUN sed -i -e '1 s/$/\/docker/' /opt/couchbase/VARIANT.txt

# Add runit service script for couchbase-server
COPY scripts/run /etc/service/couchbase-server/run
RUN set -x \
    && mkdir -p /etc/service/couchbase-server/supervise \
    && chown -R couchbase:couchbase \
                /etc/service \
                /etc/service/couchbase-server/supervise

# Add dummy script for commands invoked by cbcollect_info that
# make no sense in a Docker container
COPY scripts/dummy.sh /usr/local/bin/
RUN set -x \
    && ln -s dummy.sh /usr/local/bin/iptables-save \
    && ln -s dummy.sh /usr/local/bin/lvdisplay \
    && ln -s dummy.sh /usr/local/bin/vgdisplay \
    && ln -s dummy.sh /usr/local/bin/pvdisplay

# Fix curl RPATH if necessary - if curl.real exists, it's a new
# enough package that we don't need to do anything. If not, it
# may be OK, but just fix it
RUN set -ex \
    &&  if [ ! -e /opt/couchbase/bin/curl.real ]; then \
            ${UPDATE_COMMAND}; \
            apt-get install -y chrpath; \
            chrpath -r '$ORIGIN/../lib' /opt/couchbase/bin/curl; \
            apt-get remove -y chrpath; \
            apt-get autoremove -y; \
            ${CLEANUP_COMMAND}; \
        fi

# Add bootstrap script
COPY scripts/entrypoint.sh /
ENTRYPOINT ["/entrypoint.sh"]
CMD ["couchbase-server"]
# 8091: Cluster administration REST/HTTP traffic, including Couchbase Web Console
# 8092: Views and XDCR access
# 8093: Query service REST/HTTP traffic
# 8094: Search Service REST/HTTP traffic
# 8095: Analytics service REST/HTTP traffic
# 8096: Eventing service REST/HTTP traffic
# 8097: Backup service REST/HTTP traffic
# 9123: Analytics prometheus
# 11207: Data Service (SSL)
# 11210: Data Service
# 11280: Data Service prometheus
# 18091: Cluster administration REST/HTTP traffic, including Couchbase Web Console (SSL)
# 18092: Views and XDCR access (SSL)
# 18093: Query service REST/HTTP traffic (SSL)
# 18094: Search Service REST/HTTP traffic (SSL)
# 18095: Analytics service REST/HTTP traffic (SSL)
# 18096: Eventing service REST/HTTP traffic (SSL)
# 18097: Backup service REST/HTTP traffic (SSL)
EXPOSE 8091 \
       8092 \
       8093 \
       8094 \
       8095 \
       8096 \
       8097 \
       9123 \
       11207 \
       11210 \
       11280 \
       18091 \
       18092 \
       18093 \
       18094 \
       18095 \
       18096 \
       18097

VOLUME /opt/couchbase/var
