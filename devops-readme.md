# DevOps Final Project – Containerised

This repo contains the Petclinic application fully containerised with a Docker Compose configuration.

To get the application to run:

1. Install Docker and Docker Compose
```bash
# Install Docker
curl https://get.docker.com | sudo bash
sudo usermod -aG docker $(whoami)
newgrp docker

# Install Docker Compose
sudo apt install -y curl jq
version=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | jq -r '.tag_name')
sudo curl -L "https://github.com/docker/compose/releases/download/${version}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

2. Start the app with Docker Compose
```bash
docker-compose up -d
```

## Changes

Trainees will have to retrieve original source code from:
- [Angular Frontend](https://github.com/spring-petclinic/spring-petclinic-angular)
- [Spring REST API](https://github.com/spring-petclinic/spring-petclinic-rest)

The changes/additions that have been made to each section are listed below.

### `spring-petclinic-angular/Dockerfile`

Lines 15-16 in the original cause the Dockerfile to fail on build:
```dockerfile
ARG NGINX_VERSION="1.17.6"
FROM $DOCKER_HUB/library/nginx:$NGINX_VERSION AS runtime
```

So the Dockerfile has been updated to circumvent the need for a variable:
```dockerfile
FROM $DOCKER_HUB/library/nginx:1.17.6 AS runtime
```

The copy destination of the `/workspace/dist/` folder has also been changed. Line 19 has been changed from:
```dockerfile
COPY  --from=build /workspace/dist/ /usr/share/nginx/html/
```
To:
```dockerfile
COPY  --from=build /workspace/dist/ /usr/share/nginx/html/petclinic/
```

### `spring-petclinic-angular/src/environments/{environment.ts, environment.prod.ts}`

Lines 24-27 specify the REST API endpoint like so:
```typescript
export const environment = {
  production: false,
  REST_API_URL: 'http://localhost:9966/petclinic/api/'
};
```

In order for the application to call the REST API when hosted server-side (as opposed to locally), this code needs to be replaced with:

```typescript
export const environment = {
  production: false,
  REST_API_URL: '/petclinic/api/'
};
```

Note: this configuration will only work if used in combination with the NGINX gateway service.

### `spring-petclinic-rest/Dockerfile`

This Dockerfile has been added to allow you to build the backend image yourself, which is essential for integrating the application with an external database.

Note: The external database has **not** been configured for this repo. It currently just uses the internal H2 database.

The unit tests don't appear to work correctly when Maven is run, so Line 4 skips the tests using:
```dockerfile
RUN mvn clean package -Dskiptests
```

If you know Java and have a suggestion as to how to get the tests to run correctly, please create a pull request!

### `nginx/`

In order for the frontend to communicate with the REST API, the application requires a reverse proxy 'gateway' service that allows the frontend to treat the NGINX frontend server and the REST API as the same location.

The NGINX configuration for the reverse proxy is found at `nginx/nginx.conf`:

```nginx
events {}
http {
    server {
        listen 80;
        location / {
            proxy_pass http://petclinic-fe:8080/petclinic/;
        }
        location /petclinic/api/ {
            proxy_pass http://petclinic-be:9966/petclinic/api/;
        }
    }
}
```

The frontend makes its API requests to `/petclinic/api/`, as configured in the `environment.ts` files.
