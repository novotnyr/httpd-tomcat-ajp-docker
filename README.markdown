What is this?
============

One dockerized `httpd` loadbalances two dockerized Tomcats:

- AJP is the loadbalancing protocol
- `mod_proxy` does the AJP proxying on the `httpd` side
- `mod_proxy_balancer` handles the loadbalancing on the `httpd` side
- Tomcat AJP connector handles the AJP protocol with `httpd`.
- A `jvmRoute` is dynamically configured on the Tomcat side to provide sticky sessions.

Details
========

Tomcat
------

On Tomcat side, the following changes are made:

- custom `server.xml` is provided. (See below)
- a unique instance name is assigned to each Tomcat
- optional ACL database is provided to access Tomcat Manager Webapp
- optional IP-restrictions are lifted to access Tomcat Manager Webappp

### Configuration in `server.xml`

Tomcat configuration is minimalistic:

- a single Connector is exposed: AJP protocol on 8009
- an optional `Realm` is declared to support authentification for `manager` Tomcat webapp. This is used to demo the HTTP sticky sessions. This *realm* reads ACL information from `tomcat-users.xml`. This is not necessary for production-scale deployments and for webapps that do not use Container-Based security.

### Loadbalancing Configuration via `jvmRoute`

To implement *sticky sessions*, each Tomcat instance must declare a unique name. This identifier will be appended to the session ID header / query parameter / cookie.

Sample session ID will look as follows:

    JSESSIONID=3FF62F97618CCBEE7D91FD462A6E095E.tomcat2

We declare a parameterized *route*:

    jvmRoute="${jvmRoute}"

Tomcat is able to resolve the parameterized values from Java System Properties (provided by `-D` in the commandline).

The actual values are provided in the `docker-compose.yml` (See below).

### Optional

Two optional customizations are made to enable *Manager* Webapp provided by Tomcat.

#### ACL Database

A custom `tomcat-users.xml` is provided to implement Container-Managed Security via in-memory Realm. The  actual implementation is declared in the `server.xml`.

#### Enabling Manager Webapp from nonlocalhost

By default, Tomcat provides a `Manager` Webapp, which is restricted to 127.0.0.1. However, with Docker and loadbalancing, we need to lift this restriction.

This is implemented by removing the `Valve` from `context.xml` of the `manager` webapp. 

We plainly copy `manager-meta-inf-context.xml` under proper name to Tomcat image, to the corresponding subfolder in the `webapps`.

Apache Webserver `httpd`
------------------------

On the `httpd` side, we load all necessary modules to support `mod_proxy` and loadbalancing.

Specific changes for Docker:

- via `mod_unixd`, a specific user/group must be defined to run the server

The following modules must be enabled for loadbalance and reverse proxy on AJP:

Reverse Proxy:

- `mod_proxy`
- `mod_proxy_ajp`
- `mod_proxy_balancer`

Loadbalancing:

- `mod_proxy_balancer`
- `mod_status`: required by Balancer Manager Web UI
- `mod_slotmem_shm`: low-level shared memory mechanism
- `mod_lbmethod_byrequests`: provides *by requests* load balancing strategy

### Loadbalancing on `httpd`

We define a *tomcat* balancer with two members accessible via AJP protocol:

1. `ajp://tomcat:8009`
2. `ajp://tomcat2:8009`

The hostnames will be automatically resolved by Docker network established in the Docker Compose file.

Please note that balancer members do not include URL path suffixes. (This is handled by `ProxyPass` mountpoint declarations.) One proxy will loadbalance both mountpoints (*manager* and *root* webapps).

### Proxying and loadbalancing

We define three mount points for proxying. They are defined from the longest URLs to the shortest ones, as per Apache HTTP documentation.

- `/balancer-manager`. This one is not proxied to Tomcat as it correspond to the Balancer Manager UI. Note the exclamation mark as an exclusion styntax.
- `/manager`. This is loadbalanced to the *tomcat* balancer.
- `/`. This is loadbalanced to the *tomcat* balancer, too.

The syntax is as follows:

    ProxyPass "/manager "balancer://tomcat/manager" stickysession=JSESSIONID|jsessionid scolonpathdelim=On
    
- `manager`: We declare a mount point as a URL Suffix used by the `httpd`.
- `balancer://tomcat/manager` points to the *tomcat* loadbalancer. The URL suffix `/manager` corresponds to Tomcat webapplication context path. If this context-path suffix is different than the mount point suffix, we need to declare `ReverseProxyPass` as well.
- *sticky sessions*: Tomcat instance (worker) name will be attached to the Java Session ID. We declare cookie/header name according to Tomcat conventions.
- *cookie/header format*. Java separates worker name from the session ID via nonstandard semicolon. We teach `httpd` to take that in mind via `scolonpathdelim`.


Docker / Docker Compose
------------------------

We declare three dockerized services:

- `tomcat` and `tomcat2` represented loadbalanced Tomcats.
- `httpd` representing a client-facing HTTP server. We remap the internal port to the `19000` for the host machine.

### Host names and networking

*Docker Compose* will create an internal network with three containers. Each container is able to reach all other components via hostname resolution. The actual hostname exactly matches the service name from the `docker-compose.yml`.

As an example, the Tomcat container can ping `httpd` container via

    ping httpd 

The `httpd` is the hostname.

### Loadbalancing declarations

Each Tomcat must be uniquely identified to provide sticky sessions. The `jvmRoute` is provided via JVM Parameter.

Since `catalina.sh` is able to pickup `JAVA_OPTS`, we use this mechanism to pass the necessary instance ID via container to `catalina.sh`, then to JVM properties and then to `server.xml` which is able to resolve such JVM properties to dynamic values.


Useful commands
=================

## Access the Load Balancer Manager

The loadbalancer manager is available at:

    http://localhost:19000/balancer-manager

We can disable or customize existing loadbalancing members. The credentials are provided in the `tomcat-users.xml`.

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

## List Docker Networks

    docker network list
    
This command will show the network created for containers in the *Docker Compose*. 

## Inspect a specific network

This will show detailed info about a specific network `httpd-tomcat-docker_default`:

    docker network inspect httpd-tomcat-docker_default    
    
Connected containers are provided as well.    
    
    