### Traefik as an HTTP reverse proxy for Node-red Docker Containers.

![](https://github.com/luisgcu/Node-redDocker-Traefik/blob/master/docs/traefikreverseproxy.jpg)

### Introduction.

Node-RED is flow-based programing for the internet of things Node red would allow you to   program and  wiring together hardware devices, APIs and online services in new and interesting ways.

It provides a browser-based editor that makes it easy to wire together flows using the wide range of nodes in the palette that can be deployed to its runtime in a single-click.

### Why Traefik ?

For  testing purposes I need several Node-red instances running in small home server (IntelNuck)  accessible to  different customers, each Nodered dashboard's must  have its subdomain  with https enabled.

Traefik is very easy to use for this very basic setup, obviously after 7 hours of  research    and getting help from friends and forums. 

### Let's Encrypt & Docker

In this use case, we want to use Traefik as a *layer-7* load balancer with SSL termination for a set of micro-services used to run a web application.

We also want to automatically *discover any services* on the Docker host and let Traefik reconfigure itself automatically when containers get created (or shut down) so HTTP traffic can be routed accordingly.

In addition, we want to use Let's Encrypt to automatically generate and renew SSL certificates per hostname.

### Setting Up

In order for this to work, you'll need a server with a public IP address, with Docker and docker-compose installed on it.

In this example, we're using the fictitious domain *yourdomain.net*.

In real-life, you'll want to use your own domain and have the DNS configured accordingly so the hostname records you'll want to use point to the aforementioned public IP address.

***if you want to try this at home Do no forget to create a port forwarding rule in your router  of the ports  80 and 443  to the IP where you  pretend to host the Docker containers***

## Networking

Docker containers can only communicate with each other over TCP when they share at least one network. This makes sense from a topological point of view in the context of networking, since Docker under the hood creates IPTable rules so containers can't reach other containers *unless you'd want to*.

In this example, we're going to use a single network called `web` where all containers that are handling HTTP traffic (including Traefik) will reside in.

### Steps to follow.

*On the Docker host, run the following command:*

```
docker network create web
```

*Now, let's create a directory on the server where we will configure the rest of Traefik:*

```
mkdir -p /opt/traefik
```

*Within this directory, we're going to create 3 empty files:*

```
touch /opt/traefik/docker-compose.yml
touch /opt/traefik/acme.json 
chmod 600 /opt/traefik/acme.json
touch /opt/traefik/traefik.toml
```

The `docker-compose.yml` file will provide us with a simple, consistent and more importantly, a deterministic way to create Traefik and the Node-red containers.

**The contents of the file is as follows:**

```
version: '3'
# Create 3 node-red docker containers  ready to operate behind traefik Reverse proxy
services:
   nodered1:    #Nodered Docker container 1
    image: nodered/node-red-docker:latest
    restart: always
    user: root
    environment:
      - TZ= America/New_York
    networks:
      - web
    volumes:
      - /home/luisgcu/noder_data/node1/:/data  # Do nor forget to set  NR volumes   
    ports:
      - "1880:1880"
    labels:
      - "traefik.enable=true"
      - "traefik.backend=nodered1"
      - "traefik.docker.network=web"
      - "traefik.frontend.rule=Host:node1.yourdomain.net" 

   nodered2:   #Nodered Docker container 2
    image: nodered/node-red-docker:latest
    restart: always
    user: root
    environment:
      - TZ= America/New_York
    networks:
      - web
    volumes:
      - /home/luisgcu/noder_data/node2/:/data # Do nor forget to set  NR volumes
    ports:
      - "1881:1880"
    labels:
      - "traefik.enable=true"
      - "traefik.backend=nodered2"
      - "traefik.docker.network=web"
      - "traefik.frontend.rule=Host:node2.yourdomain.net"
      
   nodered3:  #Nodered Docker container 3 
    image: nodered/node-red-docker:latest
    restart: always
    user: root
    environment:
      - TZ= America/New_York
    networks:
      - web
    volumes:
      - /home/luisgcu/noder_data/node3/:/data  # Do nor forget to set  NR volumes
    ports:
      - "1882:1880"
    labels:
      - "traefik.enable=true"
      - "traefik.backend=nodered3"        
      - "traefik.docker.network=web"
      - "traefik.frontend.rule=Host:node3.yourdomain.net"  
      
   traefix:  # Traefix docker compose start here     
    image: traefik
    command: --api --docker
    restart: always    
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/traefik/traefik.toml:/traefik.toml
      - /opt/traefik/acme.json:/acme.json      
    networks:
      - web
    labels:
      - "traefik.frontend.headers.STSPreload=true"
      - "traefik.frontend.passHostHeader=true"  
 
networks:
  web:
    external: true

```

 As you can see, we're mounting the `traefik.toml` file as well as the (empty) `acme.json` file in the container.
Also, we're mounting the `/var/run/docker.sock` Docker socket in the container as well, so Traefik can listen to Docker events and reconfigure its own internal configuration when containers are created (or shut down).
Also, we're making sure the container is automatically restarted by the Docker engine in case of problems (or: if the server is rebooted). We're publishing the default HTTP ports `80` and `443` on the host, and making sure the container is placed within the `web` network we've created earlier on.

Let's take a look at a simple `traefik.toml` configuration as well before we'll create the Traefik container:

```
debug = true
logLevel = "ERROR"

defaultEntryPoints = ["http", "https"]

[api]
# Port for the status/dashboard page
dashboard = true

[entryPoints]
    [entryPoints.http]
    address = ":80"
        [entryPoints.http.redirect]
        entryPoint = "https"

    [entryPoints.https]
    address = ":443"
    [entryPoints.https.tls]

[retry]

[docker]
endpoint = "unix:///var/run/docker.sock"
domain = "yourdomain.net"         # Your domain here
watch = true
exposedByDefault = true

[acme]
email = "user@emailprovider.com"  # Your email here
storage = "acme.json"
entryPoint = "https"
onHostRule = true
    [acme.httpChallenge]
    entryPoint = "http"
```

*Now is time to run the docker compose*

```
$ sudo docker-compose up -d
```

![](https://github.com/luisgcu/Node-redDocker-Traefik/blob/master/docs/DockerComposeUp.jpg)

*Containers list running*

![](https://github.com/luisgcu/Node-redDocker-Traefik/blob/master/docs/ContainersList.jpg)

*Traefik Dashboard*

![](https://github.com/luisgcu/Node-redDocker-Traefik/blob/master/docs/traefikdashboard.jpg)

*Node-red*

![hh](https://github.com/luisgcu/Node-redDocker-Traefik/blob/master/docs/NRwebs.jpg)

### References used for learning .

[Docker and Lets encrypt guide](https://docs.traefik.io/user-guide/docker-and-lets-encrypt/)

[Configuration examples](https://docs.traefik.io/user-guide/examples/)

[Running a Docker container as a non-root user](https://medium.com/redbubble/running-a-docker-container-as-a-non-root-user-7d2e00f8ee15)

[Docker Compose file](https://docs.docker.com/compose/compose-file/)

[Traefik Examples](https://github.com/KamranAzeem/LearningDocker/tree/master/examples/traefik)

[Traefik as a Reverse Proxy for Docker Containers on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-ubuntu-18-04)

[Server Setup with traefik and docker-compose](https://blog.kilian.io/server-setup/)

[Traefik Forum](https://community.containo.us/t/custom-header-for-node-red/1426)

[Traefik As a Load Balancer / HTTP Reverse Proxy For Micro-Services](https://www.devtech101.com/2017/07/13/using-traefik-load-balancer-http-reverse-proxy-micro-services/)

[Local HTTPS Dev Proxy Using Lets Encrypt and Cloudflare](https://blog.rylander.io/2018/07/21/local-https-dev-proxy-using-lets-encrypt-and-cloudflare/)

[Docker-compose.yml and Traefik.toml template created  for this example](https://github.com/luisgcu/Node-redDocker-Traefik/tree/master/nodered-traefik)

Special Thanks to [@ludnadez](https://github.com/ldez) for his support and patience.

