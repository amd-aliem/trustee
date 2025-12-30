# Trustee CLI

The Trustee CLI (`trustee`) is a unified management tool that deploys and manages
trustee components: the Key Broker Service (KBS), Attestation Service (AS), and
Reference Value Provider Service (RVPS).

# Table of Contents

- [Quick Start](#quick-start)
- [Installation](#installation)
  - [Docker (Recommended)](#docker-recommended)
  - [Building Without Docker](#building-without-docker)
- [Command Reference](#command-reference)
  - [Trustee Run](#trustee-run)
  - [Trustee Keygen](#trustee-keygen)
  - [Trustee Cache-Preload](#trustee-cache-preload)
- [Related Documentation](#related-documentation)

# Quick Start

```bash
# Build and run with Docker
docker build -t trustee-cli -f tools/trustee-cli/Dockerfile .
docker run --rm -p 8080:8080 trustee-cli trustee run --allow-all
```

> [!WARNING]
> `--allow-all` disables authentication. Development only - never use in production.
> This default config is for minimal testing. It does not allow connecting to
> kbs via 8080 from outside the container. It's recommended to use docker-compose
> if you wish to do so (or see `./kbs/config/docker-compose/kbs-config.toml` for
> the `http_server.sockets` [kbs setting](../../kbs/docs/config.md)).

# Installation

## Local Build

```bash
cd tools/trustee-cli
make
sudo make install  # Installs to /usr/local/bin
```

> [!NOTE]
> x86_64 requires Intel SGX packages. See [Dockerfile](./Dockerfile) for list 
of dependencies to install. If not running intel, recommended to instead extract
binary from built docker container as shown in next section.

## Docker Build

```bash
docker build -t trustee-cli -f tools/trustee-cli/Dockerfile .
```

Either use trustee-cli through docker containers (mounting shared
volumes where needed), or you can extract the binary to your host machine:
```bash
docker create --name temp-trustee trustee-cli
sudo docker cp temp-trustee:/usr/local/bin/trustee /usr/local/bin/trustee
docker rm temp-trustee
```

# Command Reference

## Trustee Run

Launch trustee service (KBS, AS, RVPS).

```bash
trustee [--home DIR] run [--config-file CONFIG] [--allow-all]
```

### **Options:**
- `--config-file`: Path to KBS configuration file (TOML)
- `--allow-all`: Built-in allow-all policy (development only)
- `--home`: Config/data directory (env: `TRUSTEE_HOME`)
  - Root default: `/opt/confidential-containers`
  - Non-root default: `~/.trustee`

### **Examples:**
```bash
trustee run --config-file /etc/kbs-config.toml
trustee --home /custom/path run --allow-all
```

## Trustee Keygen

Generate Ed25519 authentication key pair.

```bash
trustee [--home DIR] keygen [-f OUTPUT_FILE]
```

### **Options:**
- `-f, --output-file`: Private key path (default: `$TRUSTEE_HOME/key`)

### **Example:**
```bash
trustee keygen -f /path/to/auth_key
# Creates: /path/to/auth_key (private) and /path/to/auth_key.pub (public)
```

> [!CAUTION]
> Protect private keys - never commit to version control.

## Trustee Cache-Preload

Preload URLs for offline use (i.e. VCEK certificates for air-gapped environments).

### **Options:**
- `-u, --urls-file`: File with URLs (one per line, required)
- `-c, --cache-dir`: Output directory (default: `./cache`)
- `-a, --archive`: Create tar.gz archive

### **Example:**
The following examples create a cache-preload directory `./cache` filled with
the URLS in `./urls.txt`.


1. Create a url file with desired urls:
```bash
echo "http://github.com
https://github.com/confidential-containers/trustee" > urls.txt
```

2. Run trustee-cli

From system with trustee-cli installed:
```bash
mkdir ./cache
trustee cache-preload --urls-file ./urls.txt -c ./cache
```

From system with trustee container image built:
```bash
mkdir ./cache
cp ./urls.txt ./cache/urls.txt
docker run -v $(pwd)/cache:/cache trustee-cli trustee cache-preload \
           --urls-file /cache/urls.txt -c /cache
```

# Related Documentation

- [KBS Documentation](../../kbs/README.md)
- [Attestation Service Documentation](../../attestation-service/README.md)
- [RVPS Documentation](../../rvps/README.md)
- [KBS Configuration Guide](../../kbs/docs/config.md)
- [Build Instructions](../../BUILD-INSTRUCTIONS.md)
