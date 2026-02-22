# GEMINI.md

This file provides a comprehensive overview of the `frappe_docker` project, its structure, and how to build, run, and develop with it.

## Project Overview

This project provides a Dockerized environment for running Frappe and ERPNext applications. It uses Docker Compose to define and manage the services required for a Frappe application, including the backend, frontend, websocket, and workers. The project is highly configurable through environment variables and allows for the use of custom Docker images.

The main technologies used are:
- Docker and Docker Compose
- Frappe Framework
- ERPNext
- Nginx
- Python
- Node.js

The architecture consists of several services:
- `configurator`: Sets up the initial configuration.
- `backend`: Runs the Frappe backend.
- `frontend`: The Nginx frontend that serves the Frappe application.
- `websocket`: Handles real-time communication.
- `queue-short` and `queue-long`: Background workers.

## Building and Running

### Environment Configuration

The project is configured using an `.env` file. An `example.env` file is provided with the available environment variables. The most important variables are:

- `ERPNEXT_VERSION`: The version of ERPNext to use.
- `DB_PASSWORD`: The password for the database.
- `LETSENCRYPT_EMAIL`: Your email address for Let's Encrypt.
- `SITES`: A list of sites for which to generate Let's Encrypt certificates.

### Building the Docker Images

The Docker images can be built using `docker buildx bake`. The `docker-bake.hcl` file defines the build process. To build the images, you can use the following command:

```bash
docker buildx bake
```

You can also build specific targets:

```bash
docker buildx bake erpnext
```

### Running the Application

To run the application, use Docker Compose. The `README.md` provides a simple way to get started:

```bash
docker compose -f pwd.yml up -d
```

For a more production-like setup, you can use the `compose.yaml` file along with override files in the `overrides` directory. For example, to run with a local database and Traefik for reverse proxy and SSL, you could use a command like this:

```bash
docker compose -f compose.yaml -f overrides/compose.mariadb.yaml -f overrides/compose.traefik.yaml up -d
```

## Development Conventions

### Code Style

The project uses `pre-commit` to enforce code style. The configuration is in `.pre-commit-config.yaml`. To use it, install `pre-commit` and run `pre-commit install` to set up the Git hooks.

### Contributing

The `CONTRIBUTING.md` file provides guidelines for contributing to the project. This repository is for container-related contributions. For contributions to Frappe, ERPNext, or Bench, please refer to their respective repositories.
