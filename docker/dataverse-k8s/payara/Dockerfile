# Copyright 2019 Forschungszentrum Jülich GmbH
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
# http://www.apache.org/licenses/LICENSE-2.0

FROM payara/server-full:5.201
LABEL maintainer="FDM FZJ <forschungsdaten@fz-juelich.de>"

ARG VERSION=4.19
ARG DOMAIN=domain1

ENV DATA_DIR=/data\
    DOCROOT_DIR=/docroot\
    METADATA_DIR=/metadata\
    SECRETS_DIR=/secrets\
    DOMAIN_DIR=${PAYARA_DIR}/glassfish/domains/${DOMAIN_NAME}\
    DATAVERSE_VERSION=${VERSION}\
    DATAVERSE_PKG=https://github.com/IQSS/dataverse/releases/download/v${VERSION}/dvinstall.zip\
    PGDRIVER_PKG=https://jdbc.postgresql.org/download/postgresql-42.2.12.jar\
    MEM_MAX_RAM_PERCENTAGE=70.0\
    MEM_XSS=512k

# Create basic pathes
USER root
RUN mkdir -p ${HOME_DIR} ${SCRIPT_DIR} ${SECRETS_DIR} && \
    mkdir -p ${DATA_DIR} ${METADATA_DIR} ${DOCROOT_DIR} && \
    chown -R payara: ${DATA_DIR} ${METADATA_DIR} ${DOCROOT_DIR} ${SECRETS_DIR}

# WORKAROUND MEMORY ISSUES UNTIL UPSTREAM FIXES THEM IN NEW RELEASE
RUN ${PAYARA_DIR}/bin/asadmin --user=${ADMIN_USER} --passwordfile=${PASSWORD_FILE} start-domain ${DOMAIN_NAME} && \
    ${PAYARA_DIR}/bin/asadmin --user=${ADMIN_USER} --passwordfile=${PASSWORD_FILE} delete-jvm-options \
        '-XX\:+UnlockExperimentalVMOptions:-XX\:+UseCGroupMemoryLimitForHeap:-XX\:MaxRAMFraction=1' && \
    ${PAYARA_DIR}/bin/asadmin --user=${ADMIN_USER} --passwordfile=${PASSWORD_FILE} create-jvm-options \
        '-XX\:+UseContainerSupport:-XX\:MaxRAMPercentage=${ENV=MEM_MAX_RAM_PERCENTAGE}:-Xss${ENV=MEM_XSS}' && \
    ${PAYARA_DIR}/bin/asadmin --user=${ADMIN_USER} --passwordfile=${PASSWORD_FILE} stop-domain ${DOMAIN_NAME} && \
    # Cleanup after initialization
     rm -rf \
        ${PAYARA_DIR}/glassfish/domains/${DOMAIN_NAME}/osgi-cache \
        ${PAYARA_DIR}/glassfish/domains/${DOMAIN_NAME}/logs

# Install prerequisites
RUN apt-get -qq update && \
    apt-get -qqy install postgresql-client jq imagemagick curl

# Install esh template engine from Github
RUN wget --no-verbose -O esh https://raw.githubusercontent.com/jirutka/esh/v0.3.0/esh && \
    echo 'fe030e23fc1383780d08128eecf322257cec743b esh' | sha1sum -c - && \
    chmod +x esh && mv esh /usr/local/bin

# Install PostgreSQL JDBC driver in AppServer
# TODO: remove this once upstream includes the Postgres Client lib in the WAR.
USER payara
RUN wget --no-verbose -O postgresql.jar ${PGDRIVER_PKG} && \
    mv postgresql.jar ${PAYARA_DIR}/glassfish/lib

# Make docroot of Payara reside in higher level directory for easier targeting
# Due to IQSS/dataverse-kubernetes#177: create the generated pathes so they are
# writeable by us. TBR with #178.
RUN rm -rf ${DOMAIN_DIR}/docroot && \
    ln -s ${DOCROOT_DIR} ${DOMAIN_DIR}/docroot && \
    mkdir -p ${DOMAIN_DIR}/generated/jsp/dataverse

# Retrieve the Dataverse install package, extract and remove ZIP,
#    symlink WAR file and remove Harvard custom metadatablocks
RUN cd ${HOME_DIR} && \
    wget --no-verbose -O dvinstall.zip ${DATAVERSE_PKG} && \
    unzip -qq dvinstall.zip -d ./ && \
    rm dvinstall.zip && \
    ln -s ${HOME_DIR}/dvinstall/dataverse.war ${DEPLOY_DIR}/dataverse.war && \
    find . -iname "custom*.tsv" -delete

# Copy across docker scripts
COPY --chown=payara:payara docker/dataverse-k8s/bin/* docker/dataverse-k8s/payara/bin/*.sh ${SCRIPT_DIR}/
RUN chmod +x ${SCRIPT_DIR}/*
