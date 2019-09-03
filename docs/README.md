### Traefik as an HTTP reverse proxy for Node-red Docker Containers.

![](https://github.com/luisgcu/Node-redDocker-Traefik/blob/master/docs/traefikreverseproxy.jpg)

### Introduction.

Node-RED is flow-based programing for the internet of things Node red would allow you to   program and  wiring together hardware devices, APIs and online services in new and interesting ways.

It provides a browser-based editor that makes it easy to wire together flows using the wide range of nodes in the palette that can be deployed to its runtime in a single-click.

### Why Traefik ?.

For  testing purposes I need several Node-red instances running in small home server (IntelNuck)  accessible to  different customers, each Nodered dashboard's must  have its subdomain  with https enabled.

Traefik is very easy to setup for this very basic setup, obviously after 7 hours of  research    and getting help from friends and forums. 

### Let's Encrypt & Docker

In this use case, we want to use Traefik as a *layer-7* load balancer with SSL termination for a set of micro-services used to run a web application.

We also want to automatically *discover any services* on the Docker host and let Traefik reconfigure itself automatically when containers get created (or shut down) so HTTP traffic can be routed accordingly.

In addition, we want to use Let's Encrypt to automatically generate and renew SSL certificates per hostname.

### Setting Up

In order for this to work, you'll need a server with a public IP address, with Docker and docker-compose installed on it.

In this example, we're using the fictitious domain *yourdomain.net*.

In real-life, you'll want to use your own domain and have the DNS configured accordingly so the hostname records you'll want to use point to the aforementioned public IP address.

## Networking

Docker containers can only communicate with each other over TCP when they share at least one network. This makes sense from a topological point of view in the context of networking, since Docker under the hood creates IPTable rules so containers can't reach other containers *unless you'd want to*.

In this example, we're going to use a single network called `web` where all containers that are handling HTTP traffic (including Traefik) will reside in.

### Steps to follow.









 

