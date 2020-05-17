# Swarm Nginx Proxy

This is intended to be used as a Docker Swarm ingress proxy. It's built on the awesome [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) with a small tweak to let it pick up the correct container port.

### How does it work?

Make sure you add the proxy's network (in addition to other network if applicable) then add these environment variables to any container in a stack. Make sure you use the host facing port specified either by you or the docker file and it should pick it up.

```yml
networks:
  - proxy
environment:
  - VIRTUAL_PORT=80
  - VIRTUAL_HOST=example.com
```

> I like to spin this up separately so it's more portable and not tied to another stack.

## Example proxy configs

### Plain proxy

```yml
version: '3.8'

services:
  nginx-proxy:
    image: dev01d/nginx-proxy:alpine
    networks:
      - proxy
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - nginx-certs:/etc/nginx/certs:ro
      - nginx-vhosts:/etc/nginx/vhost.d
    ports:
      - '80:80'
    deploy:
      placement:
        constraints:
          - node.role == manager

networks:
  proxy:
    driver: overlay
    name: proxy

volumes:
  nginx-certs:
  nginx-vhosts:
```

### Proxy with [LetsEncrypt companion](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion)

```yml
version: '3.8'

services:
  nginx-proxy:
    image: dev01d/nginx-proxy:alpine
    networks:
      - proxy
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - nginx-certs:/etc/nginx/certs:ro
      - nginx-vhosts:/etc/nginx/vhost.d
      - nginx-letsencrypt-challenge:/usr/share/nginx/html
    ports:
      - '80:80'
      - '443:443'
    labels:
      - com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy
    deploy:
      placement:
        constraints:
          - node.role == manager

  letsencrypt-gen:
    image: jrcs/letsencrypt-nginx-proxy-companion:latest
    networks:
      - proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - nginx-certs:/etc/nginx/certs:rw
      - nginx-vhosts:/etc/nginx/vhost.d
      - nginx-letsencrypt-challenge:/usr/share/nginx/html
    environment:
      - DEFAULT_EMAIL=your@email.com
      - NGINX_PROXY_CONTAINER=nginx-proxy
      - NGINX_DOCKER_GEN_CONTAINER=nginx-proxy
    deploy:
      placement:
        constraints:
          - node.role == manager

networks:
  proxy:
    driver: overlay
    name: proxy

volumes:
  nginx-certs:
  nginx-vhosts:
  nginx-letsencrypt-challenge:
```
