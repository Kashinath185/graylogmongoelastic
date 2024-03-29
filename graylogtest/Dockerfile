FROM centos:centos8 as graylog-downloader

ARG VCS_REF=1bed69e
ARG GRAYLOG_VERSION=5.1.2
ARG GRAYLOG_HOME=/usr/share/graylog
ARG GRAYLOG_UID=1100
ARG GRAYLOG_GID=1100

# ---------- CENTOS RELATED SETTINGS  START ----------
RUN cd /etc/yum.repos.d/ && \
        cat /etc/dnf/dnf.conf && \
        sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-* && \
        sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-* && \
        yum install -y dnf-plugins-core && \
        yum install 'dnf-command(config-manager)'
# ---------- CENTOS RELATED SETTINGS END -------

WORKDIR /tmp

# hadolint ignore=DL3008,DL3015,DL3059
RUN yum update -y && \
   yum install ca-certificates curl -y

COPY fetch-and-extract.sh /bin/fetch-and-extract
RUN chmod +x /bin/fetch-and-extract && fetch-and-extract \
    "https://packages.graylog2.org/releases/graylog/graylog-${GRAYLOG_VERSION}.tgz" \
    "https://packages.graylog2.org/releases/graylog/graylog-${GRAYLOG_VERSION}.tgz.sha256.txt"


RUN \
  install \
    --directory \
    --mode=0750 \
    /opt/graylog/data \
    /opt/graylog/data/journal \
    /opt/graylog/data/log \
    /opt/graylog/data/config \
    /opt/graylog/data/plugin \
    /opt/graylog/data/data

RUN mv /opt/graylog ${GRAYLOG_HOME} && chown -R ${GRAYLOG_UID}:${GRAYLOG_GID} ${GRAYLOG_HOME}
RUN mv ${GRAYLOG_HOME}/plugin ${GRAYLOG_HOME}/plugins-default
RUN install -d -o "${GRAYLOG_UID}" -g "${GRAYLOG_GID}" -m 0755 ${GRAYLOG_HOME}/plugins-merged && \
    install -d -o "${GRAYLOG_UID}" -g "${GRAYLOG_GID}" -m 0755 ${GRAYLOG_HOME}/plugin

COPY config ${GRAYLOG_HOME}/data/config

# -------------------------------------------------------------------------------------------------
#
# final layer
FROM centos:centos8

ARG VCS_REF=1bed69e
ARG GRAYLOG_VERSION=5.1.2
ARG BUILD_DATE=2023-06-30T05:52:32Z
ARG GRAYLOG_HOME=/usr/share/graylog
ARG GRAYLOG_USER=graylog
ARG GRAYLOG_UID=1100
ARG GRAYLOG_GROUP=graylog
ARG GRAYLOG_GID=1100

COPY --chown=${GRAYLOG_UID}:${GRAYLOG_GID} --from=graylog-downloader ${GRAYLOG_HOME} ${GRAYLOG_HOME}

WORKDIR ${GRAYLOG_HOME}

SHELL ["/bin/bash", "-o", "pipefail", "-c"]
# hadolint ignore=DL3027,DL3008
RUN \
  echo "export BUILD_DATE=${BUILD_DATE}"           >> /etc/profile.d/graylog.sh && \
  echo "export GRAYLOG_VERSION=${GRAYLOG_VERSION}" >> /etc/profile.d/graylog.sh && \
  echo "export GRAYLOG_SERVER_JAVA_OPTS='-Dlog4j2.formatMsgNoLookups=true -Djdk.tls.acknowledgeCloseNotify=true -XX:+UnlockExperimentalVMOptions -XX:-OmitStackTraceInFastThrow -XX:+UseG1GC -server'" >> /etc/profile.d/graylog.sh && \
  echo "export GRAYLOG_HOME=${GRAYLOG_HOME}"       >> /etc/profile.d/graylog.sh && \
  echo "export GRAYLOG_USER=${GRAYLOG_USER}"       >> /etc/profile.d/graylog.sh && \
  echo "export GRAYLOG_GROUP=${GRAYLOG_GROUP}"     >> /etc/profile.d/graylog.sh && \
  echo "export GRAYLOG_UID=${GRAYLOG_UID}"         >> /etc/profile.d/graylog.sh && \
  echo "export GRAYLOG_GID=${GRAYLOG_GID}"         >> /etc/profile.d/graylog.sh && \
  echo "export PATH=${GRAYLOG_HOME}/bin:${PATH}"   >> /etc/profile.d/graylog.sh 
# ---------- CENTOS RELATED SETTINGS  START ----------
RUN cd /etc/yum.repos.d/ && \
    cat /etc/dnf/dnf.conf && \
    sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-* && \
    sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-* && \
    yum install -y dnf-plugins-core && \
    yum install 'dnf-command(config-manager)'
# ---------- CENTOS RELATED SETTINGS END -------
 
RUN yum update -y  && \
    yum install -y epel-release && \
    yum install -y curl  libcap glib2-devel libX11 nss dejavu-sans-fonts fontconfig  

RUN dnf makecache --refresh && \
    dnf -y install tini    

RUN  groupadd -g "${GRAYLOG_GID}" "${GRAYLOG_GROUP}" && \
     adduser \
    --home ${GRAYLOG_HOME} \
    --uid "${GRAYLOG_UID}" \
    --gid "${GRAYLOG_GID}" \
    "${GRAYLOG_USER}" && \
   find "$JAVA_HOME/lib" -name '*.so' -exec dirname '{}' ';' | sort -u > /etc/ld.so.conf.d/docker-openjdk.conf && \
   ldconfig && \
   yum clean all > /dev/null 
COPY docker-entrypoint.sh /
COPY health_check.sh /
EXPOSE 9000
USER ${GRAYLOG_USER}
VOLUME ${GRAYLOG_HOME}/data
ENTRYPOINT ["tini", "--", "/docker-entrypoint.sh"]
CMD ["server"]

# add healthcheck
HEALTHCHECK \
  --interval=10s \
  --timeout=2s \
  --retries=12 \
  CMD /health_check.sh

# -------------------------------------------------------------------------------------------------

LABEL maintainer="Graylog, Inc. <hello@graylog.com>" \
      org.label-schema.name="Graylog Docker Image" \
      org.label-schema.description="Official Graylog Docker image" \
      org.label-schema.url="https://www.graylog.org/" \
      org.label-schema.vcs-ref=${VCS_REF} \
      org.label-schema.vcs-url="https://github.com/Graylog2/graylog-docker" \
      org.label-schema.vendor="Graylog, Inc." \
      org.label-schema.version=${GRAYLOG_VERSION} \
      org.label-schema.schema-version="1.0" \
      org.label-schema.build-date=${BUILD_DATE}