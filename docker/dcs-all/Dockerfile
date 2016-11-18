FROM alpine:3.3

MAINTAINER Data Conservancy <dc-ird@googlegroups.com>

#####################
#Basic setup: Install java, curl, bash (required for jetty startup scripts)
RUN apk add --update openjdk8-jre && apk add --update curl && apk add --update bash && rm -rf /var/cache/apk/* && mkdir /shared && chmod a+rwx /shared

ENV SHARED=/shared \
    APPS=/opt \
    JAVA_HOME=/usr/lib/jvm/default-jvm \
    LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/lib/jvm/default-jvm/jre/lib/amd64:/usr/lib/jvm/default-jvm/jre/lib/amd64/jli:/usr/lib/jvm/default-jvm/jre/lib/amd64/server

#####################
#Jetty
ENV JETTY_HOME=${APPS}/jetty \
    JETTY_DEPLOY_DIR=${APPS}/jetty/webapps \
    JETTY_SHARED=${SHARED}/jetty \
    JETTY_EXE=${APPS}/jetty/bin/jetty_dcs.sh

# Download and unpack Jetty
# Set logging directory in /shared
# Create simplified startup script
RUN export JETTY_VERSION=9.3.9.v20160517 && \
    mkdir -p ${APPS} && \
    curl -L http://central.maven.org/maven2/org/eclipse/jetty/jetty-distribution/${JETTY_VERSION}/jetty-distribution-${JETTY_VERSION}.tar.gz | tar xzf - -C ${APPS} && \
    ln -s ${APPS}/jetty-distribution-${JETTY_VERSION} ${JETTY_HOME} && \
    echo "--module=logging" >> ${JETTY_HOME}/start.ini && \
    printf "#!/bin/sh\nmkdir -p ${JETTY_SHARED}/logs\njava -Djetty.home=${JETTY_HOME} -Djetty.base=${JETTY_HOME} -Djetty.logging.dir=${JETTY_SHARED}/logs -jar ${JETTY_HOME}/start.jar" \
        > ${JETTY_HOME}/bin/jetty_dcs.sh && \
    chmod a+x ${JETTY_HOME}/bin/jetty_dcs.sh

#####################
#Download & Deploy Fedora 4.5.0.  
#Modify jetty's start.ini to provide fedora home system property
RUN export FEDORA_VERSION=4.5.0 && \
    curl -L https://github.com/fcrepo4/fcrepo4/releases/download/fcrepo-${FEDORA_VERSION}/fcrepo-webapp-${FEDORA_VERSION}.war \
        > ${JETTY_DEPLOY_DIR}/fcrepo.war && \
    echo "-Dfcrepo.home=${SHARED}/fedora-data" >> ${JETTY_HOME}/start.ini

#####################
# Karaf
ENV KARAF_HOME=${APPS}/apache-karaf  \
    KARAF_DEPLOY=${APPS}/apache-karaf/deploy \
    KARAF_SHARED=${SHARED}/karaf \
    KARAF_OPTS=-Dkaraf.shared=${SHARED}/karaf
# Download and unpack Karaf
# Add public Data Conservancy Maven repo to repository list (so Karaf can pull artifacts from it)
# Add additional deploy directory (auto-deploys configs, bundles, features, etc) in /shared
# Install webconsole
# Change permissions so Karaf can be run by any unprivileged user
RUN export KARAF_VERSION=4.0.4 && \
    mkdir -p ${APPS} && \
    curl -L https://archive.apache.org/dist/karaf/${KARAF_VERSION}/apache-karaf-${KARAF_VERSION}.tar.gz | tar xzf - -C ${APPS} && \
    ln -s ${APPS}/apache-karaf-${KARAF_VERSION} ${KARAF_HOME} && \ 
    cat ${KARAF_HOME}/etc/org.ops4j.pax.url.mvn.cfg \
        | sed 's/org.ops4j.pax.url.mvn.repositories=/org.ops4j.pax.url.mvn.repositories = http:\/\/maven.dataconservancy.org\/public\/releases@id=dc.public.releases,http:\/\/maven.dataconservancy.org\/public\/snapshots@id=dc.public.snapshots@snapshots@noreleases,/' \
        > /tmp/org.ops4j.pax.url.mvn.cfg && \
    mv /tmp/org.ops4j.pax.url.mvn.cfg ${KARAF_HOME}/etc/org.ops4j.pax.url.mvn.cfg && \ 
    cat ${KARAF_HOME}/etc/org.ops4j.pax.logging.cfg | sed s/karaf.data/karaf.shared/ > /tmp/org.ops4j.pax.logging.cfg && \
    mv /tmp/org.ops4j.pax.logging.cfg ${KARAF_HOME}/etc/org.ops4j.pax.logging.cfg && \
    cat ${KARAF_HOME}/etc/org.apache.felix.fileinstall-deploy.cfg \ 
        | sed 's/${karaf.base}\/deploy/\/shared\/karaf\/deploy/' \
        > ${KARAF_HOME}/etc/org.apache.felix.fileinstall-shared.cfg && \
    ${KARAF_HOME}/bin/start && \
    echo "feature:install webconsole " \
        | ${KARAF_HOME}/bin/client -u karaf -b -r 10 &&  \
    ${KARAF_HOME}/bin/stop && \  
    chmod a+rw -R ${KARAF_HOME}/data ${KARAF_HOME}/etc ${KARAF_HOME}/instances ${KARAF_HOME}/lock && \
    rm -rf ${KARAF_SHARED}
    
    
#####################
# Deploy package ingest in karaf
RUN export PIS_VERSION=LATEST && \
    ${KARAF_HOME}/bin/start && \
    echo "feature:repo-add mvn:org.dataconservancy.packaging/package-ingest-karaf/${PIS_VERSION}/xml/features; " \
        "feature:install package-ingest-karaf" \
        | ${KARAF_HOME}/bin/client -u karaf -b -r 10 &&  \
    ${KARAF_HOME}/bin/stop && \
    rm -rf ${KARAF_SHARED}
    
#################################### 
# Package ingest default configuration
ENV KARAF_OPTS="${KARAF_OPTS} -Dfedora.baseURI=http://localhost:8080/fcrepo/rest -Ddeposit.dir.root=${SHARED}/package-ingest/packages -Ddeposit.dir.root.fail=${SHARED}/package-ingest/fail-packages"
RUN echo "package.extract.dir = /tmp" > ${KARAF_HOME}/etc/org.dataconservancy.packaging.impl.PackageFileAnalyzerFactory.cfg && \
    echo 'fedora.baseuri = ${fedora.baseURI}' > ${KARAF_HOME}/etc/org.dataconservancy.packaging.ingest.camel.impl.FedoraDepositDriver.cfg && \
    touch ${KARAF_HOME}/etc/org.dataconservancy.packaging.ingest.camel.impl.LoggerNotifications.cfg && \
    export FILE=${KARAF_HOME}/etc/org.dataconservancy.packaging.ingest.camel.impl.PackageFileDepositWorkflow-root.cfg && \
    echo 'deposit.location = ${fedora.baseURI}' >> ${FILE} && \ 
    echo 'package.deposit.dir = ${deposit.dir.root}' >> ${FILE} && \ 
    echo 'package.fail.dir = ${deposit.dir.root.fail}' >> ${FILE} && \
    chmod a+rw ${KARAF_HOME}/etc
    
##############################
# A script to start both karaf and jetty
RUN printf "#!/bin/sh\nsh ${KARAF_HOME}/bin/start\n${JETTY_EXE}" > /usr/bin/launch.sh && \
    chmod a+x /usr/bin/launch.sh
