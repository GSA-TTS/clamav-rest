# Table of Contents
- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [Updates](#updates)
- [Installation](#installation)
- [Quick Start](#quick-start)
  - [Status Codes](#status-codes)
- [Endpoints](#endpoints)
  - [Utility endpoints](#utility-endpoints)
  - [Scanning endpoints](#scanning-endpoints)
- [Configuration](#configuration)
  - [Environment Variables](#environment-variables)
  - [Networking](#networking)
- [Maintenance / Monitoring](#maintenance--monitoring)
  - [Shell Access](#shell-access)
  - [Prometheus](#prometheus)
- [Development](#development)
- [Deprecations](#deprecations)
  - [`/scan` Endpoint](#scan-endpoint)
    - [Differences between `/scan` and `/v2/scan`](#differences-between-scan-and-v2scan)
  - [centos.Dockerfile](#centosdockerfile)
- [History](#history)
- [References](#references)


# Introduction

This is a two in one docker image which runs the open source virus scanner ClamAV (https://www.clamav.net/), performs automatic virus definition updates as a background process and provides a REST API interface to interact with the ClamAV process.

# Updates

2025-01-08: [PR 50](https://github.com/ajilach/clamav-rest/pull/50) integrated which now provides a new `/v2` endpoint returning more scan result information: status, description, http status and a list of scanned files. See the PR for more details. The old `/scan` endpoint is now considered deprecated. Also, a file size scan limit has been added which can be configured through the `MAX_FILE_SIZE` environment variable. This update also fixes a bug that would falsely return `200 OK` if the first file in a multi file scan was clean, regardless if any of the following files contained viruses. All endpoints now increment the Prometheus virus metric counter when a virus is discovered during a scan.

2024-10-21: freshclam notifies the correct `.clamd.conf` so that `clamd` is notified about updates and the correct version is returned now.
This is an additional fix to the latest fix from October 15 2024 which was not working. Thanks to [christianbumann](https://github.com/christianbumann) and [arizon-dread](https://github.com/arizon-dread).

2024-10-15: ClamAV was thought to handle database updates correctly thanks to [christianbumann](https://github.com/christianbumann). It turned out that this was not the case.

As of May 2024, the releases are built for multiple architectures thanks to efforts from [kcirtapfromspace](https://github.com/kcirtapfromspace) and support non-root read-only deployments thanks to [robaca](https://github.com/robaca).

The additional endpoint `/version` is now available to check the `clamd` version and signature date. Thanks [pastral](https://github.com/pastral).

Closed a security hole by upgrading our `Dockerfile` to the alpine base image version `3.19` thanks to [Marsup](https://github.com/Marsup).

# Installation

Automated builds of the image are available on [Registry](https://hub.docker.com/r/ajilaag/clamav-rest) and is the recommended method of installation.

```bash
docker pull hub.docker.com/ajilaag/clamav-rest:(imagetag)
```

The following image tags are available:
* `latest` - Most recent release of ClamAV with REST API
* `YYYYMMDD` - The day of the release
* `sha-...` - The git commit sha. This version ensures that the exact image is used and will be unique for each build

# Quick Start

See [this docker-compose file](docker-compose-nonroot.yml) for non-root read-only usage.

Run clamav-rest docker image:
```bash
docker run -p 9000:9000 -p 9443:9443 -itd --name clamav-rest ajilaag/clamav-rest
```

Test that service detects common test virus signature:

**HTTP:**

```bash
$ curl -i -F "file=@eicar.com.txt" http://localhost:9000/v2/scan
HTTP/1.1 100 Continue

HTTP/1.1 406 Not Acceptable
Content-Type: application/json; charset=utf-8
Date: Mon, 28 Aug 2017 20:22:34 GMT
Content-Length: 56

[{ "Status": "FOUND", "Description": "Eicar-Test-Signature","FileName":"eicar.com.txt"}]
```

**HTTPS:**

```bash
$ curl -i -k -F "file=@eicar.com.txt" https://localhost:9443/v2/scan
HTTP/1.1 100 Continue

HTTP/1.1 406 Not Acceptable
Content-Type: application/json; charset=utf-8
Date: Mon, 28 Aug 2017 20:22:34 GMT
Content-Length: 56

[{ "Status": "FOUND", "Description": "Eicar-Test-Signature","FileName":"eicar.com.txt"}]
```

Test that service returns 200 for clean file:

**HTTP:**

```bash
$ curl -i -F "file=@clamrest.go" http://localhost:9000/v2/scan

HTTP/1.1 100 Continue

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Mon, 28 Aug 2017 20:23:16 GMT
Content-Length: 33

[{ "Status": "OK", "Description": "","FileName":"clamrest.go"}]
```
**HTTPS:**

```bash
$ curl -i -k -F "file=@clamrest.go" https://localhost:9443/v2/scan

HTTP/1.1 100 Continue

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Mon, 28 Aug 2017 20:23:16 GMT
Content-Length: 33

[{ "Status": "OK", "Description": "","FileName":"clamrest.go"}]
```

## Status Codes
- 200 - clean file = no KNOWN infections
- 400 - ClamAV returned general error for file
- 406 - INFECTED
- 412 - unable to parse file
- 413 - request entity too large, the file exceeds the scannable limit. Set MAX_FILE_SIZE to scan larger files
- 422 - filename is missing in MimePart
- 501 - unknown request

# Endpoints  
## Utility endpoints 
| Endpoint | Description |
|----------|-------------| 
| `/` | Home endpoint, returns stats for the running process | 
| `/version` | Returns the clamav binary version and also the version of the virus signature databases and the signature update date. |
| `/metrics` | Prometheus endpoint for scraping metrics. |
## Scanning endpoints  
| Endpoint | Description |
|----------|-------------|
| `/v2/scan` | Scanning endpoint, accepts a multipart/form-data request with one or more files and returns a json array with status, description and filename, along with the most severe http status code that was possible to determine. <br/>**response sample:** <br/> `[{"Status":"OK","Description":"","FileName":"checksums.txt"}]` |
| `/scanPath?path=/folder` | A scanning endpoint that will scan a folder, a practical example would be to mount a share into the container where you dump files in a folder, call scanPath and let it scan them all, then continue processing them<br/> **response sample:**<br/> `[{"Raw":"/folder: OK","Description":"","Path":"/folder","Hash":"","Size":0,"Status":"OK"}]` |
| `/scanHandlerBody` | This endpoints scans the content in the HTTP POST request body. <br/> **response sample:**<br/> `{OK   200}` |
| `/scan` | [DEPRECATED] This endpoint scans in a similar manner to `/v2/scan` but does return one or more json objects without a containing structure in between (no json array). It also does not include the filename as a json property. It is still present in the api for backwards compatibility reasons for those who still use it but it will also return headers indicating deprecation and pointing out the new, updated endpoint, `/v2/scan`. It does accept a multipart/form-data endpoint that by http standards can accept multiple files, and does scan them all, but the implementation of the endpoint indicates that it was originally (probably) meant to only scan one file at a time. Please don't rely on this endpoint to exist in the future, the project has an intention to sunset it in the future when it becomes a pain to maintain. <br/>**response sample:** <br/>`{"Status":"OK","Description":""}` |
# Configuration

## Environment Variables

Below is the complete list of available options that can be used to customize your installation.

| Parameter | Description |
|-----------|-------------|
| `MAX_SCAN_SIZE` | Amount of data scanned for each file - Default `100M` |
| `MAX_FILE_SIZE` | Don't scan files larger than this size - Default `25M` |
| `MAX_RECURSION` | How many nested archives to scan - Default `16` |
| `MAX_FILES` | Number of files to scan withn archive - Default `10000` |
| `MAX_EMBEDDEDPE` | Maximum file size for embedded PE - Default `10M` |
| `MAX_HTMLNORMALIZE` | Maximum size of HTML to normalize - Default `10M` |
| `MAX_HTMLNOTAGS` | Maximum size of Normlized HTML File to scan- Default `2M` |
| `MAX_SCRIPTNORMALIZE` | Maximum size of a Script to normalize - Default `5M` |
| `MAX_ZIPTYPERCG` | Maximum size of ZIP to reanalyze type recognition - Default `1M` |
| `MAX_PARTITIONS` | How many partitions per Raw disk to scan - Default `50` |
| `MAX_ICONSPE` | How many Icons in PE to scan - Default `100` |
| `PCRE_MATCHLIMIT` | Maximum PCRE Match Calls - Default `100000` |
| `PCRE_RECMATCHLIMIT` | Maximum Recursive Match Calls to PCRE - Default `2000` |
| `SIGNATURE_CHECKS` | Check times per day for a new database signature. Must be between 1 and 50. - Default `2` |

## Networking

| Port | Description |
|-----------|-------------|
| `3310`    | ClamD Listening Port |

# Maintenance / Monitoring

## Shell Access

For debugging and maintenance purposes you may want access the containers shell.

```bash
docker exec -it (whatever your container name is e.g. clamav-rest) /bin/sh
```

Checking the version with the `clamscan` command requires to provide the custom database path.
The default value is overwritten to `/clamav/data` in the `/clamav/etc/clamd.conf`, and the `clamav` service
was started with this`/clamav/etc/clamd.conf` from the `entrypoint.sh`.

```bash
clamscan --database=/clamav/data --version
```

## Prometheus

[Prometheus metrics](https://prometheus.io/docs/guides/go-application/) were implemented, which can be retrieved as follows

**HTTP:**
curl http://localhost:9000/metrics

**HTTPS:**
curl https://localhost:9443/metrics

# Development

Source code can be found here: https://github.com/ajilach/clamav-rest

Build golang (linux) binary and docker image:

```bash
# env GOOS=linux GOARCH=amd64 go build
docker build . -t clamav-rest
docker run -p 9000:9000 -p 9443:9443 -itd --name clamav-rest clamav-rest
```

# Deprecations

## `/scan` Endpoint  
As of release [20250109](https://github.com/ajilach/clamav-rest/releases/tag/20250109) the `/scan` endpoint is deprecated and `/v2/scan` is now the preferred endpoint to use.  

### Differences between `/scan` and `/v2/scan`  
Since the endpoint can receive one or several files, the response has been updated to always be returned as a json array and the filename is now included as a property in the response, to make it easy to find out what file(s) contains virus. 

## centos.Dockerfile  
The centos.Dockerfile has been bumped in the release [20250109](https://github.com/ajilach/clamav-rest/releases/tag/20250109) but will not be maintained going forward. If there are community users using it, please consider contributing to maintain it.  

# History

This work is based on the awesome work done by [o20ne/clamav-rest](https://github.com/o20ne/clamav-rest) which is based on [niilo/clamav-rest](https://github.com/niilo/clamav-rest) which is based on the original code from [osterzel/clamav-rest](https://github.com/osterzel/clamav-rest).

# References

* https://www.clamav.net
* https://github.com/ajilach/clamav-rest
