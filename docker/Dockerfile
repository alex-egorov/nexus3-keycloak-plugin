# Build nexus-repository-heml
FROM maven:3-jdk-8 as nexus-repository-helm

ENV NEXUS_REPOSITORY_HELM_VERSION 0.0.6

#RUN apk add --no-cache git
RUN git clone -b v${NEXUS_REPOSITORY_HELM_VERSION} https://github.com/sonatype-nexus-community/nexus-repository-helm \
  && cd nexus-repository-helm \
  && mvn clean package -DskipTests


# build nexus-repositoru-apt
FROM maven:3-jdk-8 as nexus-repository-apt

ENV NEXUS_REPOSITORY_APT_VERSION 1.0.8

RUN git clone -b ${NEXUS_REPOSITORY_APT_VERSION} https://github.com/sonatype-nexus-community/nexus-repository-apt \
  && cd nexus-repository-apt \
  && mvn clean package -DskipTests


# build jmx-exporter-javaagent
FROM maven:3-jdk-8 as jmx-exporter

ENV JMX_EXPORTER_VERSION 0.3.1

RUN git clone -b parent-${JMX_EXPORTER_VERSION} https://github.com/prometheus/jmx_exporter \
  && cd jmx_exporter \
  && mvn clean package -DskipTests


# build nexus3-repositoru-apt
FROM maven:3-jdk-8 as nexus3-keycloak-plugin

ENV NEXUS_KEYCLOAK_PLUGIN_VERSION 0.4.0

RUN git clone -b ${NEXUS_KEYCLOAK_PLUGIN_VERSION} https://github.com/alex-egorov/nexus3-keycloak-plugin \
  && cd nexus3-keycloak-plugin \
  && mvn clean package -DskipTests


############################################################################

# main image
#FROM quay.io/pires/docker-jre:8u171_alpine_3.8.1
FROM alpine:3.8
#FROM alex202/jre

LABEL MAINTAINER Alex Egorov <aegorov@kublr.com>

ENV NEXUS_VERSION 3.14.0-04
ENV NEXUS_DOWNLOAD_URL "https://download.sonatype.com/nexus/3"
ENV NEXUS_TARBALL_URL "${NEXUS_DOWNLOAD_URL}/nexus-${NEXUS_VERSION}-unix.tar.gz"
ENV NEXUS_TARBALL_ASC_URL "${NEXUS_DOWNLOAD_URL}/nexus-${NEXUS_VERSION}-unix.tar.gz.asc"
ENV GPG_KEY 0374CF2E8DD1BDFD

ENV SONATYPE_DIR /opt/sonatype
ENV NEXUS_HOME "${SONATYPE_DIR}/nexus"
ENV NEXUS_DATA /nexus-data
ENV NEXUS_CONFIG /nexus-config
ENV NEXUS_CONTEXT ''
ENV SONATYPE_WORK ${SONATYPE_DIR}/sonatype-work

ENV NEXUS_REPOSITORY_HELM_VERSION 0.0.6
ENV NEXUS_REPOSITORY_APT_VERSION 1.0.8
ENV JMX_EXPORTER_VERSION 0.3.1
ENV NEXUS_KEYCLOAK_PLUGIN_VERSION 0.4.0

ENV JAVA_HOME=/usr/lib/jvm/default-jvm/jre

# Install nexus
RUN apk add --no-cache --update bash ca-certificates runit su-exec util-linux
RUN apk add --no-cache openjdk8-jre \
  && apk add --no-cache -t .build-deps wget gnupg openssl  \
  && cd /tmp \
  && echo "===> Installing Nexus ${NEXUS_VERSION}..." \
  && wget -O nexus.tar.gz $NEXUS_TARBALL_URL; \
  wget -O nexus.tar.gz.asc $NEXUS_TARBALL_ASC_URL; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --keyserver ha.pool.sks-keyservers.net --recv-keys $GPG_KEY; \
    gpg --batch --verify nexus.tar.gz.asc nexus.tar.gz; \
    rm -r $GNUPGHOME nexus.tar.gz.asc; \
  tar -xf nexus.tar.gz \
  && mkdir -p $SONATYPE_DIR \
  && mv nexus-$NEXUS_VERSION $NEXUS_HOME \
  && cd $NEXUS_HOME \
  && ls -las \
  && adduser -h $NEXUS_DATA -DH -s /sbin/nologin nexus \
  && chown -R nexus:nexus $NEXUS_HOME \
  && rm -rf /tmp/* \
  && apk del --purge .build-deps

# Configure nexus
RUN sed \
    -e '/^nexus-context/ s:$:${NEXUS_CONTEXT}:' \
    -i ${NEXUS_HOME}/etc/nexus-default.properties \
  && sed \
    -e '/^-Xms/d' \
    -e '/^-Xmx/d' \
    -i ${NEXUS_HOME}/bin/nexus.vmoptions

RUN mkdir -p ${NEXUS_DATA}/etc ${NEXUS_DATA}/log ${NEXUS_DATA}/tmp ${SONATYPE_WORK} \
  && ln -s ${NEXUS_DATA} ${SONATYPE_WORK}/nexus3 \
  && chown -R nexus:nexus ${NEXUS_DATA}

# Replace logback configuration
COPY logback.xml ${NEXUS_HOME}/etc/logback/logback.xml
COPY logback-access.xml ${NEXUS_HOME}/etc/logback/logback-access.xml

# Copy runnable script
COPY run /etc/service/nexus/run

# and make it executable
RUN chmod +x /etc/service/nexus/run

## Copy nexus-repository-help plugin
COPY --from=nexus-repository-helm  /nexus-repository-helm/target/nexus-repository-helm-${NEXUS_REPOSITORY_HELM_VERSION}.jar ${NEXUS_HOME}/system/org/sonatype/nexus/plugins/nexus-repository-helm/${NEXUS_REPOSITORY_HELM_VERSION}/nexus-repository-helm-${NEXUS_REPOSITORY_HELM_VERSION}.jar

# Add plugin into to nexus configs
RUN sed -i.bak \
    -e '/<feature prerequisite="true" dependency="false">wrap<\/feature>/i \        <feature prerequisite="false" dependency="false">nexus-repository-helm</feature>' \
    ${NEXUS_HOME}/system/org/sonatype/nexus/assemblies/nexus-core-feature/*/nexus-core-feature-*-features.xml \
  && sed -i -e "/<\/features>/i \    <feature name='nexus-repository-helm' description='org.sonatype.nexus.plugins:nexus-repository-helm' version='${NEXUS_REPOSITORY_HELM_VERSION}'>\n \
        <details>org.sonatype.nexus.plugins:nexus-repository-helm</details>\n \
        <bundle>mvn:org.sonatype.nexus.plugins/nexus-repository-helm/${NEXUS_REPOSITORY_HELM_VERSION}</bundle>\n \
    </feature>" \
  ${NEXUS_HOME}/system/org/sonatype/nexus/assemblies/nexus-core-feature/*/nexus-core-feature-*-features.xml


## Copy nexus-repository-apt plugin
COPY --from=nexus-repository-apt  /nexus-repository-apt/target/nexus-repository-apt-${NEXUS_REPOSITORY_APT_VERSION}.jar ${NEXUS_HOME}/system/net/staticsnow/nexus-repository-apt/${NEXUS_REPOSITORY_APT_VERSION}/nexus-repository-apt-${NEXUS_REPOSITORY_APT_VERSION}.jar

# Add plugin into to nexus configs
RUN sed -i.bak \
    -e '/mvn:org.tukaani/a \        <feature prerequisite="false" dependency="false">nexus-repository-apt</feature>' \
    ${NEXUS_HOME}/system/org/sonatype/nexus/assemblies/nexus-core-feature/*/nexus-core-feature-*-features.xml \
  && sed -i -e "/<\/features>/i \    <feature name='nexus-repository-apt' description='net.staticsnow:nexus-repository-apt' version='${NEXUS_REPOSITORY_APT_VERSION}'>\n \
        <details>net.staticsnow:nexus-repository-apt</details>\n \
        <bundle>mvn:net.staticsnow/nexus-repository-apt/${NEXUS_REPOSITORY_APT_VERSION}</bundle>\n \
        <bundle>mvn:org.apache.commons/commons-compress/1.18</bundle>\n \
        <bundle>mvn:org.tukaani/xz/1.8</bundle>\n \
    </feature>" \
  ${NEXUS_HOME}/system/org/sonatype/nexus/assemblies/nexus-core-feature/*/nexus-core-feature-*-features.xml


## Copy jmx-exportet-javaagent
COPY --from=jmx-exporter  /jmx_exporter/jmx_prometheus_javaagent/target/jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar ${NEXUS_HOME}/jmx-exporter/jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar
COPY jmx-exporter-config.yaml ${NEXUS_CONFIG}/jmx-exporter-config.yaml


## Copy nexus3-keycloak-plugin
COPY --from=nexus3-keycloak-plugin /nexus3-keycloak-plugin/target/nexus3-keycloak-plugin-${NEXUS_KEYCLOAK_PLUGIN_VERSION}.jar ${NEXUS_HOME}/system/org/github/flytreeleft/nexus3-keycloak-plugin/${NEXUS_KEYCLOAK_PLUGIN_VERSION}/nexus3-keycloak-plugin-${NEXUS_KEYCLOAK_PLUGIN_VERSION}.jar
RUN echo "mvn\\:org.github.flytreeleft/nexus3-keycloak-plugin/${NEXUS_KEYCLOAK_PLUGIN_VERSION} = 200" >> ${NEXUS_HOME}/etc/karaf/startup.properties


VOLUME ${NEXUS_DATA}

EXPOSE 8081

WORKDIR ${NEXUS_HOME}

ENV INSTALL4J_ADD_VM_PARAMS="-Xms1200m -Xmx1200m -javaagent:${NEXUS_HOME}/jmx-exporter/jmx_prometheus_javaagent-${JMX_EXPORTER_VERSION}.jar=8180:/nexus-config/jmx-exporter-config.yaml"

CMD ["/sbin/runsvdir", "-P", "/etc/service"]

