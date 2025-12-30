# Offline AMD VCEK Caching Guide

This document describes the process to pre-load VCEKs (Versioned Chip
Endorsement Keys) into the trustee environment to allow normal attestation
without connection to the AMD KDS. Trustee supports certificate caching through
the rust `http_cache_reqwest` crate, which caches URL responses into a
specified folder that can be preloaded with certificates using the
[cache-preloader](../../tools/cache-preloader) tool. This use case is primarily
targeted at air-gapped environments as a stopgap solution while a more
comprehensive approach is developed.

> [!Note]
> Currently this guide shows a method that builds on using
> [docker](https://confidentialcontainers.org/docs/attestation/installation/docker/)
> to manage kbs services. Additional deployment methods will be covered in future releases.

## Contents

- [Prerequisites](#prerequisites)
- [Enabling Offline Disk Cache in Attestation Service](#enabling-offline-disk-cache-in-attestation-service)
- [Understanding the Cache Structure](#understanding-the-cache-structure)
- [Limitations](#limitations)

## Prerequisites

- On system used to build cache:
  - Network access to AMD KDS (https://kdsintf.amd.com)
  - Rust toolchain installed (for building cache-preloader)
- On AMD EPYC host(s) with guests to be attested:
  - snphost tool ([link](https://github.com/virtee/snphost))

## Enabling Offline Disk Cache in Attestation Service

### 1. Create VCEK URL target file

Create a text file with the list of URLs that you would like cached - one URL
per line. Trustee will need a VCEK URL for each physical AMD EPYC server that
the KBS will be servicing. The machine's VCEK URL can be fetched using the
[snphost tool](https://github.com/virtee/snphost).

On each AMD SNP host, with root/admin access run:
```
sudo snphost show vcek-url | sed 's/[a-zA-Z0-9]\{128\}/\L&/'
```

You should get a URL of the form:
```
https://kdsintf.amd.com/vcek/{version}/{machine}/{product_name}/{hwid}?{params}
```

> [!IMPORTANT]
> - Note that the VCEK URL is specific to the hardware AMD firmware of the
> machine. If the firmware is updated, the VCEK URL will change. See the
> [AMD VCEK documentation](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/57230.pdf)
> for more information about the VCEK URL format.
> - Note that the URL parameters are case-sensitive to trustee, and the `{hwid}`
> field must be lowercase. The sed command above ensures this.


### 2. Run `trustee cache-preload` to generate cache contents

On a system with access to the AMD KDS, run `trustee cache-preload` specifying
the VCEK URL file and output directory.

```
# Build the trustee cli if not already installed
docker build -t trustee-cli -f tools/trustee-cli/Dockerfile .

# Preload the cache (optionally create an archive, if copying to
# another system)
mkdir ./vcek-cache
cp ./urls.txt ./vcek-cache/urls.txt
docker run -v $(pwd)/vcek-cache:/cache trustee-cli trustee cache-preload \
           --urls-file /cache/urls.txt -c /cache
```

### 3. Configure the attestation service

Update the attestation config file to use a predefined cache:

`kbs/config/as-config.json`:

```json
{
    // ... other fields ...
    "verifier_config": {
        "snp_verifier": {
            "vcek_cache": {
               "type": "OfflineDiskCache",
               "path": "/opt/confidential-containers/vcek-cache"
            }
        }
    }
}
```

### 4. Install cache contents on air-gapped attestation deployment

Copy output directory contents to your air-gapped attestation deployment
maintaining the same directory structure. There are two primary methods to do this:

- **Install on running attestation service**

You may copy the cache to the container after it has started - ensure that
the as-config.json has picked up the DiskCache settings by checking the
debug logs:
```bash
echo "RUST_LOG=debug" > debug.env
docker compose --env-file debug.env up -d
docker logs trustee-as-1 | grep vcek_cache
                            vcek_cache: OfflineDiskCache,
```

Then copy your archive (making sure to add the ending `/` in both paths):
```
sudo docker cp ./vcek-cache/ trustee-as-1:/opt/confidential-containers/vcek-cache/
```

- **Mount shared directory**

You may also mount a shared directory from the host into the container by
updating `docker-compose.yml` with a specified directory mapping:

```yaml
  as:
    ... <existing configuration>
    volumes:
    ... <existing volumes>
    - ./vcek-cache:/opt/confidential-containers/vcek-cache:rw
```

### Example:

On a system with network access to AMD KDS:
```bash
# Create a URLs file - one line for each target machine that kds will service
cat > vcek_urls.txt << EOF
https://kdsintf.amd.com/vcek/v-1/Milan/0123456789abcdef?blSPL=03&teeSPL=00&snpSPL=08&ucodeSPL=115
https://kdsintf.amd.com/vcek/v-1/Genoa/fedcba9876543210?blSPL=02&teeSPL=00&snpSPL=03&ucodeSPL=209
EOF

# Build the trustee cli if not already installed
docker build -t trustee-cli -f tools/trustee-cli/Dockerfile .

# Create cache archive
mkdir ./vcek-cache
cp ./urls.txt ./vcek-cache/urls.txt
docker run -v $(pwd)/vcek-cache:/cache trustee-cli trustee cache-preload \
           --urls-file /cache/urls.txt -a /cache/vcek-cache.tar.gz

# Copy the archive to your air-gapped trustee attestation-service deployment
scp cache/vcek-cache.tar.gz user@target-host:
```

On the air-gapped trustee attestation-service host:
```bash
# Update as-config.json to enable Disk Caching
cd /path/to/trustee
vi kbs/config/as-config.json
# Update the verifier_config section to:
#     "verifier_config": {
#        "snp_verifier": {
#            "vcek_cache": {
#                "type": "OfflineDiskCache",
#                 "path": "/opt/confidential-containers/vcek-cache"
#            }
#        }
#    }

# Extract the archive into the desired cache directory
mkdir vcek-cache
cd vcek-cache
tar -xzf ../vcek-cache.tar.gz
cd ..

docker compose build as
docker compose up -d

# Copy the cache into the running attestation service container
sudo docker cp ./vcek-cache/ trustee-as-1:/opt/confidential-containers/vcek-cache/

# Alternative to above copy would be to add to following to
# docker-compose.yml under as.volumes section:
# - ./vcek-cache:/opt/confidential-containers/vcek-cache:rw
```

## Understanding the Cache Structure

The cache-preloader tool uses the `http-cache-reqwest` crate which creates a
content-addressable cache with the following structure:

```
cache-directory/
├── content-v2/
│   └── sha256/
│       └── XX/
│           └── YY/
│               └── <hash> (actual certificate data)
└── index-v5/
    └── XX/
        └── YY/
            └── <hash> (cache metadata and mappings)
```

- **content-v2/sha256/**: Stores the actual HTTP response bodies (the VCEK
  certificates) indexed by content hash
- **index-v5/**: Stores cache metadata including URL-to-content mappings,
  headers, and cache control information

When transferring the cache to an air-gapped environment, you must preserve
this exact directory structure. The entire cache directory should be copied
as-is to `/opt/confidential-containers/vcek-cache` in the attestation service
container.

## Limitations

This offline caching solution has several limitations that users should be
aware of:

- **Firmware-Specific Certificates**: VCEKs are tied to specific firmware
  versions. If the AMD firmware is updated on any host, new VCEKs must be
  fetched and added to the cache. The existing cached certificates will become
  invalid for that host.

- **Manual Cache Management**: There is no automatic mechanism to detect when
  cached certificates become stale or when new certificates are needed.
  Administrators must manually track firmware updates and rebuild the cache
  accordingly.

- **No Certificate Revocation Checking**: When operating in offline mode, the
  system cannot check for certificate revocations or updates from the AMD KDS.
  This is an inherent limitation of air-gapped deployments.

- **TCB Version Dependencies**: The VCEK URLs include TCB (Trusted Computing
  Base) version parameters (blSPL, teeSPL, snpSPL, ucodeSPL). If these versions
  change due to firmware updates, the cached certificates will not match and
  attestation will fail.

- **Storage Requirements**: Each VCEK certificate must be cached separately.
  For large deployments with many hosts, ensure adequate storage is available
  for the cache directory.

- **Inserting New Certificates**: There is currently not a tool for
  adding new certificates to an existing cache. Administrators may need to
  modify cache files or rebuild the cache from scratch when new certificates
  are required.
