# Docker Multi-Container App — Student List (API + Web Frontend)

A containerized two-tier application demonstrating Docker networking, container communication via DNS, and a transition from manual `docker run` commands to Infrastructure-as-Code with Docker Compose — including a private Docker registry with a web UI.

## Stack
- Docker
- Docker Compose
- Docker Registry (`registry:2`) + `joxit/docker-registry-ui`
- PHP / Apache (frontend)
- Python/Flask-style REST API (backend)

## Architecture

```
[ Browser ] --> [ webapp_pozos (PHP/Apache, port 80) ]
                          |
                    (bridge network: pozos_network, internal DNS)
                          |
              [ api_pozos (Flask API, port 5000) ]
```

The frontend calls the backend API by container name over a custom bridge network — no exposed backend port, no hardcoded IPs.

## What it does
- Builds a custom API image serving a student list from a mounted JSON file
- Creates an isolated Docker bridge network so containers resolve each other by name
- Runs the backend API container with a mounted data volume
- Runs a PHP/Apache frontend that queries the API and displays results
- Migrates the manual container setup into a `docker-compose.yml` for reproducible, ordered startup (`depends_on`)
- Adds a private Docker registry + GUI to tag, push, and manage images

## How to run

**Manual (step-by-step) version:**

```bash
# 1. Build the API image
cd ./mini-projet-docker/simple_api
docker build . -t api-pozos:1

# 2. Create a bridge network so containers can reach each other by name
docker network create pozos_network --driver=bridge

# 3. Run the backend API container
cd ..
docker run --rm -d --name api_pozos --network pozos_network -v ./simple_api/:/data/ api-pozos:1

# 4. Point the frontend at the API container (via network DNS)
sed -i 's/<api_ip_or_name:port>/api_pozos:5000/g' ./website/index.php

# 5. Run the frontend
docker run --rm -d --name webapp_pozos -p 80:80 --network pozos_network \
  -v ./website/:/var/www/html \
  -e USERNAME=toto -e PASSWORD=python php:apache
```

**Test the API through the frontend:**

```bash
docker exec webapp_pozos curl -u toto:python -X GET http://api_pozos:5000/pozos/api/v1.0/get_student_ages
```

Or open `localhost:80` in a browser (use `hostname -I` to find your IP if running on a remote VM).

**Clean up:**

```bash
docker stop api_pozos webapp_pozos
docker network rm pozos_network
```

**Docker Compose (IaC) version:**

```bash
# Run the app
docker-compose up -d

# Run the private registry + GUI
docker-compose -f docker-compose-registry.yml up -d
# GUI available at ip_addr:8080
```

**Push an image to the private registry:**

```bash
docker login localhost:5000
docker image tag api-pozos:1 localhost:5000/pozos/api-pozos:1
docker image push localhost:5000/pozos/api-pozos:1
```

## Notes
- Credentials for the frontend (`USERNAME`/`PASSWORD`) and the registry are set for demo purposes only and should be replaced with secrets management in any real deployment.
- The backend port (5000) is intentionally not exposed to the host — only reachable inside the Docker network.
