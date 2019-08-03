FROM tomcat:9
EXPOSE 8009 8080
RUN rm -rf /usr/local/tomcat/webapps/examples/
RUN rm -rf /usr/local/tomcat/webapps/docs/
COPY server.xml /usr/local/tomcat/conf/
