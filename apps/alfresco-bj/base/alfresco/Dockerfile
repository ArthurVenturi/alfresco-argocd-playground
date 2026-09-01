FROM alfresco/alfresco-content-repository-community:latest

ARG TOMCAT_DIR=/usr/local/tomcat  
# default user is alfresco (added on the base image alfresco/alfresco-content-r>
# change to root user to be able to install the addons and packages
USER root

# Install modules and addons
RUN mkdir -p $TOMCAT_DIR/amps
COPY modules/amps $TOMCAT_DIR/amps
COPY modules/jars $TOMCAT_DIR/webapps/alfresco/WEB-INF/lib

# custom log4j
COPY logs/custom-log4j2.properties $TOMCAT_DIR/shared/classes/alfresco/extension/custom-log4j2.properties 

# use a subshell to find the exact name of the MMT jar file automatically
RUN MMT_PATH=$(ls $TOMCAT_DIR/alfresco-mmt/alfresco-mmt*.jar | head -n 1) && \
    java -jar $MMT_PATH install \
    $TOMCAT_DIR/amps $TOMCAT_DIR/webapps/alfresco -directory -nobackup -force -verbose
