# How to test new “local AI” libraries without bleeding your data ?
Share this quick and easy docker micro infrastructure for strong network isolation and controlled service exposure: ideal for zero trust LLM and application sandboxing.

This is literally a 20-line security upgrade with a simple “internet switch” that everyone should know and use.

# Context
Companies and engineers want the shiny new LLM toys.
This implies to push and probe cutting-edge local AI stacks — inference wrappers, vector DBs, fine-tune toolkits — to learn their capabilities.
But sending prompts, chats, or uploaded files into opaque local solutions without knowing what they persist, where they persist it, or what telemetry they phone home is reckless.
Worse: a seemingly benign Python dependency or an automatic update can introduce data-exfiltration behavior or even ship malware — supply-chain and trojanized packages are real risks.
Treat every new library (and its updates) as untrusted.

# Description
The architecture consists of two services: sandbox-myapp (the application you want to sandbox) and sandbox-proxy (the reverse proxy), with the proxy routing traffic from/to the app via sandbox-myapp:8080. Networking is split into two networks: `isolated_net` for internal communication between the app and proxy, and `internet_net` for external access to the proxy through port 8080. In this setup the application is isolated, it cannot talk directly to the internet so your app cannot directly exfiltrate.

# Files hierarchy  
├── proxy/  
│   ├── Dockerfile  
│   └── Caddyfile  
├── myapp/  
│   ├── Dockerfile  
│   └── app_data  
├── docker-compose.yml  
└── README.md  

# The docker compose file
````yaml
services:
  sandbox-myapp:
    (...)
    volumes:
      - ./myapp/app_data:/app/data
    networks:
      - isolated_net
    expose:
      - "8080"

  sandbox-proxy:
    (...)
    ports:
      - "8080:8080"          # only proxy publishes this to host
    networks:
      - isolated_net         # so proxy can reach the app internally
      - internet_net         # so proxy can be reached from host/bridge

networks:
  # isolated_net is internal to Docker (no external container routing)
  isolated_net:
    internal: true
    attachable: true

  # external network that is routable by the host (bridge default)
  internet_net:
    internal: false
````
# The Caddy files

## Dockerfile
````Dockerfile
FROM caddy:2.10.2-alpine

# Create non-root user
RUN adduser -D -g 'caddy' caddy

# Switch to non-root user
USER caddy

# Copy Caddyfile into the container
COPY Caddyfile /etc/caddy/Caddyfile

# Expose port 8080
EXPOSE 8080
````
## Configuration file
````
:8080 {
    reverse_proxy sandbox-myapp:8080
}
````

# Procedure
## Enable internet and setup your app container
- setup the myapp docker image filling ./myapp/Dockerfile => Fill ./myapp/Dockerfile with the Dockerfile of your app to isolate
- edit docker-compose.yml: in networks section of the sandbox-myapp service, add internet_net
- also complete the volumes section: change :/app/data with folder path to map in the container. Add others paths for multiple folders to mount as volumes.
- eventually change or complete ports exposition and mapping if app is not 8080 api or web server
- run docker-compose up
Now test the app, download the models, set up the details and validate functionalities
## Disable internet and use 
- edit docker-compose.yml: in networks section of the sandbox-myapp service, remove internet_net; there must be only isolated_net in the networks section of the sandbox-myapp service
- restart services with docker-compose down && docker-compose up
**NOW YOUR APP CONTAINER CAN NOT REACH INTERNET, THE APP CONTAINER CAN NOT EXFILTRATE**

## Need to redo ? change models ?
Before changing image or connect to internet to download other models, plugins, or assets think of it:
- destruct the existing container
- remove or move volumes during internet connection
- rebuild infra

