# Docker with Traefik v2, MySQL, PhpMyAdmin, Redis, ... in one container for local development

## I had several issues getting clients website on my local machine because more and more required a specific setup, versions or even straight HTTPS plugin from WordPress. 

NOTE: in 2022 we should be all using HTTPS

1. Obvious, but get Docker for your environment : https://www.docker.com and verify installation by typing : 

```bash
docker --version
```
it should print out something like that
```bash
Docker version 20.10.17, build 100c701
```

2. Lets make sure you have certificates to generate self signed for localhost
```bash
#for Linux machines
# you need cerutil first as well
sudo apt install libnss3-tools -y
# then

#for MacOS machines
brew install mkcert && nss
#I let u handle urself for Windows
```

2. [BIS][OPTIONAL] At this point, I've created a folder inside my ~/Sites/ that I called traefik, so 
```bash
mkdir ~/Sites/traefik
```

3. Inside /traefik folder (to keep everything tidy as we all love), generate local CA with: 
```bash 
mkcert -install
```

4. Then, I choose to get certificate for *.docker.localhost, so first :
```bash
mkdir certs
# certificates will work for every sub domain of ${subdomainName}.docker.localhost
mkcert -cert-file certs/local-cert.pem -key-file certs/local-key.pem "docker.localhost" "*.docker.localhost"
# you can keep adding domains like that, if needed
mkcert -cert-file certs/local-cert.pem -key-file certs/local-key.pem "docker.localhost" "*.docker.localhost" "*.domain.local" "*.domain.local" 
```

5. Create a /traefik folder. Note that it can be a bit redondant, but I haven't come with another name yet. I could've named it reverse-proxy or config, but anyway. 
```bash
mkdir traefik
```

6. Then, for Traefik router to migrate everything from HTTP to HTTPS, you need a static configuration like below:

```yml 
# Note that you could've used .toml file for Traefik v1, but I hate toml 

# /traefik/traefik.yml
global:
  sendAnonymousUsage: false

api:
  dashboard: true
  insecure: true

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    watch: true
    exposedByDefault: false

  file:
    filename: /etc/traefik/config.yml
    watch: true

log:
  level: INFO
  format: common

entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
  https:
    address: ":443"
```

And for automatic configuration, we use following code:

```yml
# /traefik/config.yml
http:
  routers:
    traefik:
      rule: "Host(`traefik.docker.localhost`)"
      service: "api@internal"
      tls:
        domains:
          - main: "docker.localhost"
            sans:
              - "*.docker.localhost"
          - main: "domain.local"
            sans:
              - "*.domain.local"

tls:
  certificates:
    - certFile: "/etc/certs/local-cert.pem"
      keyFile: "/etc/certs/local-key.pem"
```

7. Then it is based on preferences and naming conventions. I personnaly prefer to create a network that has same name rather than a network that describe what it is used for (ex below)

```bash 
docker network create traefik
# or for most used naming conventions
docker network create proxy
```

8. Then, I looked up Traefik documentation and came up with this config:
```yml
# docker-compose.yml 

version: "3.3"

networks:
    # Allow the use of traefik in other docker-compose.yml files
    traefik:
        external: true
    backend:

services:

  traefik:
    image: traefik:v2.3
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      # Web
      - 80:80
      - 443:443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      # use of static config
      - ./traefik/traefik.yml:/etc/traefik/traefik.yml:ro
      # dynamic config
      - ./traefik/config.yml:/etc/traefik/config.yml:ro
      # certificates you generated before
      - ./certs:/etc/certs:ro
    networks:
      - traefik
    labels:
      # Allow traefik to act as router and have all needed informations
      # More infos : https://docs.traefik.io/providers/docker/#exposedbydefault
      - "traefik.enable=true"
      # dynamic config from: ./traefik/config.yml
      - "traefik.http.routers.traefik=true"

  whoami:
    image: containous/whoami
    container_name: whoami
    security_opt:
      - no-new-privileges:true
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.docker.localhost`)"
      - "traefik.http.routers.whoami.tls=true"
    networks:
      - traefik

  db:
    image: mysql:5.7
    restart: always
    logging:
      options:
        max-size: "10m"
        max-file: "3"
    environment:
      # change these with the name you want
      MYSQL_DATABASE: ${user}
      MYSQL_USER: ${user}
      MYSQL_PASSWORD: ${user}
      MYSQL_ROOT_PASSWORD: password
    volumes:
      # Persist the database on disk
      - ./db:/var/lib/mysql
    networks:
      - traefik


  phpmyadmin:
     depends_on:
       - db
     image: phpmyadmin/phpmyadmin
     restart: always
     ports:
       - 7777:80
     environment:
       PMA_HOST: db
       # login: root 
       MYSQL_ROOT_PASSWORD: password 
       UPLOAD_LIMIT: 300M
     labels:
      - "traefik.enable=true"
      # container URL
      - "traefik.http.routers.phpmyadmin.rule=Host(`phpmyadmin.docker.localhost`)"
      # TLS
      - "traefik.http.routers.phpmyadmin.tls=true"
     networks:
      - traefik

  redis:
    image: redis:6
    restart: always
    logging:
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "6379:6379"
    networks:
      - traefik
    # [optionnal but useful sometimes] 
    # launch Redis in cache mode with :
    #  - max memory up to 50% of your RAM if needed (--maxmemory 512mb)
    #  - deleting oldest data when max memory is reached (--maxmemory-policy allkeys-lru)
    entrypoint: redis-server --maxmemory 2048mb --maxmemory-policy allkeys-lru
```

9. Let's make a quick pause by checking that everything is working: 

```bash
docker-compose -f docker-compose.yml up
```
and visit https://traefik.docker.localhost and https://whoami.docker.localhost 

10. I won't go much deaper, but for WordPress, a docker-compose.yml will look something like that: 

Quick note: I first created a separate folder ${project_name}, so I can have as many folder as I want and every one of them are separate 
```bash
mkdir ~/Sites/${project_name}
```

[IMPORTANT]: replace all ${project_name} with desired name in the following

```yml 
version: '3'

networks:
  # enable connection with Traefik
  traefik:
    external: true

services:
  wordpress:
    build:
      # call the Dockerfile in ./wordpress
      context: ./wordpress
    restart: always
    logging:
      options:
        max-size: "1000m"
        max-file: "3"
    environment:
      # Connect WordPrerss to the database, which is running in our previous built traefik container (and network)
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: ${project_name}
      WORDPRESS_DB_PASSWORD: ${project_name}
      WORDPRESS_DB_NAME: ${project_name}
    volumes:
      # save the content of WordPress an enable local modifications
      - ./wordpress/data:/var/www/html
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${project_name}.rule=Host(`${project_name}.docker.localhost`)"
      - "traefik.http.routers.${project_name}.tls=true"
      
```

11. Enjoy and visit ${project_name}.docker.localhost 🥳