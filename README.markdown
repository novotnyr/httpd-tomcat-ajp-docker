Useful commands
=================

## Check the Tomcat Manager availability

    curl 'http://localhost:19000/manager/'

## Check the ROOT availability

    curl 'http://localhost:19000/'
    
## Shell inside HTTPD container

    docker exec -it httpd-tomcat-docker_httpd_1 bash    
    
## Check host resolution

Let's check `httpd` availability from Tomcat.

    docker exec -it httpd-tomcat-docker_tomcat_1 ping httpd
    
* Bear in mind that `exec` operates on running containers and `run` starts a brand new unnetworked container that is not aware of `docker-compose`-aware network.