---
title: Using Trivy to Scan Images for CVEs
date: 2025-06-08
description: Trivy is an open-source tool that scans container images for vulnerabilities and security issues. It can be used to identify CVEs (Common Vulnerabilities and Exposures) in container images, helping to ensure that your applications are secure and up-to-date.
tags: ["Open Source", "Cloud Engineering", "Kubernetes", "Cloud Native", "Software Development"]
Cover: https://images.unsplash.com/photo-1540164116753-0f3f75070f42?q=80&w=1696&auto=format&fit=crop&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---
_A local AquaSec tool for securing our precious containers, filesystems and git repositories!_

## CI/CD Pipelines Are Often Slow

Like any modern enterprise solution, my projects are built and bundled into a container image to be deployed into a cloud environment. In this process, there is often automated scans to check for linting & formatting, code quality, tests, hard-coded secrets, and CVEs. These all in a pipeline, but for some projects, it can take 20min before any real feedback is provided! Considering we _should_ be able to build our project locally, why can't we scan it as well? What if we could decrease the time between build and scans by working outside the pipeline first? Enter, Trivy!

> Trivy is a comprehensive and versatile security scanner. Trivy has scanners that look for security issues, and targets where it can find those issues.
>
> - github.com/aquasecurity/trivy

Being built as a binary tool which can run locally, Trivy is the answer to my complaint about slow feedback cycles within build pipelines. Now, I can scan whenever I want; as part of a `precommit` hook, as a sanity check during debugging, or as a way to reduce my pull request size by only committing builds that I've already tested and scanned locally. Let's see how it works.

## Installation

Being an open source project, Trivy can be built and installed from it's source, from a package manager, or used as a portable docker container!

### Building Trivy

Trivy is a Go-based project and thus, should be trivial to build on a machine. On a _Nix_ system, the build instructions would look similar to this:

```bash
#!/bin/bash
set -e

# Helper function
function error_exit {
    echo "Error: $1" >&2
    exit 1
}

if [ -w /usr/local/bin ]; then
    DEST="/usr/local/bin"
else
    DEST="$HOME/.local/bin"
    mkdir -p "$DEST" || exit_error "Failed to create $DEST"
fi


git clone https://github.com/aquasecurity/trivy || exit_error "Failed to clone repository"
cd trivy                                        || exit_error "Failed to enter directory"
go build -o trivy cmd/trivy/main.go             || exit_error "Failed to build binary"
chmod 755 ./trivy                               || exit_error "Failed to set execute permissions"
cp ./trivy "$DEST/"                             || exit_error "Failed to copy trivy binary"
```

### Downloading from Github Releases

Navigating to the [Github: trivy](https://github.com/aquasecurity/trivy/releases) will point you to the binary builds for `MacOS - Apple Sillicon`, `MacOS - Intel`, `Linux - Intel`, `Linux - Arm`, and `Windows - Intel`. Download the appropriate release binary and place it into the correct location for your chosen operating system. If done correctly, `trivy` should be within your PATH / Command Line. Similar to building trivy, you have to manually place it into a directory found on your PATH if you want to access it from any directory. The following would work on _Linux_ and _MacOS_ systems as a full install script:

```bash
#!/bin/bash
set -e

# Helper function
function error_exit {
    echo "Error: $1" >&2
    exit 1
}

# Validate we have the tools needed for the script
for cmd in curl tar uname grep sed; do
  if ! command -v $cmd >/dev/null 2>&1; then
    error_exit "$cmd is required but not installed."
  fi
done

# Detect OS & Architecture
OS=$(uname -s)
case "$OS" in
    Linux*)     TRIVY_OS="Linux";;
    Darwin*)    TRIVY_OS="macOS";;
    *)          error_exit "Unsupported OS: $OS";;
esac

ARCH=$(uname -m)
case "$ARCH" in
    x86_64          TRIVY_ARCH="64bit";;
    arm64|aarch64   TRIVY_ARCH="ARM64";;
    *)              error_exit "Unsupported architecture: $ARCH"
esac

# Retrieve latest trivy release build version
LATEST=$(curl -s https://api.github.com/repos/aquasecurity/trivy/releases/latest | grep '"tag_name":' | sed -E 's/.*"v([^"]+)".*/\1/')
if [ -z "$LATEST" ]; then
    error_exit "Could not determine latest Trivy version."
fi

# Create the download URL
TARBALL="trivy_${LATEST}_${TRIVY_OS}_${TRIVY_ARCH}.tar.gz"
URL="https://github.com/aquasecurity/trivy/releases/download/v${LATEST}/${TARBALL}"

# Determine binary install location
if [ -w /usr/local/bin ]; then
    DEST="/usr/local/bin"
else
    DEST="$HOME/.local/bin"
    mkdir -p "$DEST" || exit_error "Failed to create $DEST"
fi

# Download, extract and copy latest build from tar
TMPDIR=$(mktemp -d)             || exit_error "Failed to create temp directory"
cd "$TMPDIR"                    || exit_error "Failed to navigate to temp directory"
curl -sSL -o "$TARBALL" "$URL"  || exit_error "Failed to download tarball"
tar -xzf "$TARBALL"             || exit_error "Failed to extract tarball"
chmod 755 ./trivy               || exit_error "Failed to set execute permissions"
cp ./trivy "$DEST/"             || exit_error "Failed to copy trivy binary"
```

### Using a Package Manager

Trivy is found many the vast array of package managers thanks to official and community support!

| Name                                       | Ownership | Install Command              | Installation Link                                                                                                                    |
| ------------------------------------------ | ----------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| MacOS                                      | Official          | `brew install trivy`         | [Trivy.dev: Installation - MacOS](https://trivy.dev/latest/getting-started/installation/#homebrew-official)                           |
| Fedora / Red Hat Enterprise Linux / CentOS | Official          | `sudo yum -y install trivy`  | [Trivy.dev: Installation - RHEL/CentOS](https://trivy.dev/latest/getting-started/installation/#rhelcentos-official)                   |
| Debian / Ubuntu                            | Official          | `sudo apt-get install trivy` | [Trivy.dev: Installation - Debian/Ubuntu](https://trivy.dev/latest/getting-started/installation/#debianubuntu-official)               |
| OpenSUSE / SUSE Enterprise Linux           | Community         | `sudo zypper install trivy`  | [Trivy.dev: Installation - OpenSUSE/SUSE Enterprise Linux](https://trivy.dev/latest/getting-started/installation/#opensuse-community) |
| Arch Linux                                 | Community         | `sudo pacman -S trivy`       | [Trivy.dev: Installation - Arch Linux](https://trivy.dev/latest/getting-started/installation/#archlinux-community)                    |

### Containerized

If you wish to not have Trivy installed alongside your system's binaries, it can be run from a Docker image! There are some benefits to this, such as:

1. No installation required, you can easily delete and re-download the same version as required.
2. Cross-platform compatibility, the same version and build runs in the container for all host systems.
3. CI/CD integration is even easier, since it's provided as a container image which can be run in a pipeline.

```bash
podman run aquasec/trivy image docker.io/temporalio/admin-tools:latest

2025-06-06T22:38:10Z    WARN    [vulndb] Trivy DB may be corrupted and will be re-downloaded. If you manually downloaded DB - use the `--skip-db-update` flag to skip updating DB.
2025-06-06T22:38:10Z    INFO    [vulndb] Need to update DB
2025-06-06T22:38:10Z    INFO    [vulndb] Downloading vulnerability DB...
2025-06-06T22:38:10Z    INFO    [vulndb] Downloading artifact...        repo="mirror.gcr.io/aquasec/trivy-db:2"
2025-06-06T22:38:14Z    INFO    [vuln] Vulnerability scanning is enabled
2025-06-06T22:38:14Z    INFO    [secret] Secret scanning is enabled
2025-06-06T22:38:14Z    INFO    [secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2025-06-06T22:38:14Z    INFO    [secret] Please see also https://trivy.dev/v0.63/docs/scanner/secret#recommendation for faster secret detection
2025-06-06T22:38:15Z    INFO    [python] Licenses acquired from one or more METADATA files may be subject to additional terms. Use `--debug` flag to see all affected packages.
2025-06-06T22:38:17Z    INFO    Detected OS     family="alpine" version="3.21.3"
2025-06-06T22:38:17Z    INFO    [alpine] Detecting vulnerabilities...   os_version="3.21" repository="3.21" pkg_num=58
2025-06-06T22:38:17Z    INFO    Number of language-specific files       num=7
2025-06-06T22:38:17Z    INFO    [gobinary] Detecting vulnerabilities...
2025-06-06T22:38:17Z    INFO    [python-pkg] Detecting vulnerabilities...
2025-06-06T22:38:17Z    WARN    Using severities from other vendors for some vulnerabilities. Read https://trivy.dev/v0.63/docs/scanner/vulnerability#severity-selection for details.
```

## Usage

With the CLI being installed or available, we can run it with a simple `trivy`. With it, we can scan images and file-systems! Other notable features, some of which are deemed experimental are:

- Scanning a Kubernetes cluster
- Scanning a Virtual Machine image
- Scanning a repository
- Scanning a SBOM

For our use, let's go over the image scan. As we can see in the output below, to scan an image we use `trivy image <IMAGE_NAME>`.  The scan occurs locally, and the image can be local as well -an important part of my workflow as I update vendor & internal dependencies to avoid critical CVEs within our platform. In the example, I'm scanning for what vulnerabilities exist within the `temporalio/server` image.

```bash
trivy image docker.io/temporalio/server
2025-06-08T18:21:49-04:00       INFO    [vuln] Vulnerability scanning is enabled
2025-06-08T18:21:49-04:00       INFO    [secret] Secret scanning is enabled
2025-06-08T18:21:49-04:00       INFO    [secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2025-06-08T18:21:49-04:00       INFO    [secret] Please see also https://trivy.dev/dev/docs/scanner/secret#recommendation for faster secret detection
2025-06-08T18:21:52-04:00       INFO    Detected OS     family="alpine" version="3.21.3"
2025-06-08T18:21:52-04:00       INFO    [alpine] Detecting vulnerabilities...   os_version="3.21" repository="3.21" pkg_num=30
2025-06-08T18:21:52-04:00       INFO    Number of language-specific files       num=5
2025-06-08T18:21:52-04:00       INFO    [gobinary] Detecting vulnerabilities...
2025-06-08T18:21:52-04:00       WARN    Using severities from other vendors for some vulnerabilities. Read https://trivy.dev/dev/docs/scanner/vulnerability#severity-selection for details.

Report Summary

┌─────────────────────────────────────────────┬──────────┬─────────────────┬─────────┐
│                   Target                    │   Type   │ Vulnerabilities │ Secrets │
├─────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ docker.io/temporalio/server (alpine 3.21.3) │  alpine  │        1        │    -    │
├─────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ usr/local/bin/dockerize                     │ gobinary │        4        │    -    │
├─────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ usr/local/bin/tctl                          │ gobinary │        9        │    -    │
├─────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ usr/local/bin/tctl-authorization-plugin     │ gobinary │        7        │    -    │
├─────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ usr/local/bin/temporal                      │ gobinary │        7        │    -    │
├─────────────────────────────────────────────┼──────────┼─────────────────┼─────────┤
│ usr/local/bin/temporal-server               │ gobinary │        5        │    -    │
└─────────────────────────────────────────────┴──────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)


docker.io/temporalio/server (alpine 3.21.3)

Total: 1 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 1, CRITICAL: 0)

┌─────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┬───────────────────────────────────────────────────────┐
│ Library │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │                         Title                         │
├─────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┼───────────────────────────────────────────────────────┤
│ c-ares  │ CVE-2025-31498 │ HIGH     │ fixed  │ 1.34.3-r0         │ 1.34.5-r0     │ c-ares: c-ares has a use-after-free in read_answers() │
│         │                │          │        │                   │               │ https://avd.aquasec.com/nvd/cve-2025-31498            │
└─────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┴───────────────────────────────────────────────────────┘

usr/local/bin/dockerize (gobinary)

Total: 4 (UNKNOWN: 0, LOW: 0, MEDIUM: 3, HIGH: 1, CRITICAL: 0)

┌─────────────────────┬────────────────┬──────────┬────────┬───────────────────┬────────────────┬───────────────────────────────────────────────────────────┐
│       Library       │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version  │                           Title                           │
├─────────────────────┼────────────────┼──────────┼────────┼───────────────────┼────────────────┼───────────────────────────────────────────────────────────┤
│ golang.org/x/crypto │ CVE-2025-22869 │ HIGH     │ fixed  │ v0.32.0           │ 0.35.0         │ golang.org/x/crypto/ssh: Denial of Service in the Key     │
│                     │                │          │        │                   │                │ Exchange of golang.org/x/crypto/ssh                       │
│                     │                │          │        │                   │                │ https://avd.aquasec.com/nvd/cve-2025-22869                │
├─────────────────────┼────────────────┼──────────┤        ├───────────────────┼────────────────┼───────────────────────────────────────────────────────────┤
│ golang.org/x/net    │ CVE-2025-22870 │ MEDIUM   │        │ v0.34.0           │ 0.36.0         │ golang.org/x/net/proxy: golang.org/x/net/http/httpproxy:  │
│                     │                │          │        │                   │                │ HTTP Proxy bypass using IPv6 Zone IDs in golang.org/x/net │
│                     │                │          │        │                   │                │ https://avd.aquasec.com/nvd/cve-2025-22870                │
│                     ├────────────────┤          │        │                   ├────────────────┼───────────────────────────────────────────────────────────┤
│                     │ CVE-2025-22872 │          │        │                   │ 0.38.0         │ golang.org/x/net/html: Incorrect Neutralization of Input  │
│                     │                │          │        │                   │                │ During Web Page Generation in x/net in...                 │
│                     │                │          │        │                   │                │ https://avd.aquasec.com/nvd/cve-2025-22872                │
├─────────────────────┼────────────────┤          │        ├───────────────────┼────────────────┼───────────────────────────────────────────────────────────┤
│ stdlib              │ CVE-2025-22871 │          │        │ v1.23.6           │ 1.23.8, 1.24.2 │ net/http: Request smuggling due to acceptance of invalid  │
│                     │                │          │        │                   │                │ chunked data in net/http...                               │
│                     │                │          │        │                   │                │ https://avd.aquasec.com/nvd/cve-2025-22871                │
└─────────────────────┴────────────────┴──────────┴────────┴───────────────────┴────────────────┴───────────────────────────────────────────────────────────┘

usr/local/bin/tctl (gobinary)

Total: 9 (UNKNOWN: 0, LOW: 2, MEDIUM: 6, HIGH: 1, CRITICAL: 0)

┌──────────────────────────────────────────────────────────────┬────────────────┬──────────┬────────┬───────────────────────────────────────┬──────────────────────────────┬──────────────────────────────────────────────────────────────┐
│                           Library                            │ Vulnerability  │ Severity │ Status │           Installed Version           │        Fixed Version         │                            Title                             │
├──────────────────────────────────────────────────────────────┼────────────────┼──────────┼────────┼───────────────────────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ go.opentelemetry.io/contrib/instrumentation/google.golang.o- │ CVE-2023-47108 │ HIGH     │ fixed  │ v0.36.4                               │ 0.46.0                       │ opentelemetry-go-contrib: DoS vulnerability in otelgrpc due  │
│ rg/grpc/otelgrpc                                             │                │          │        │                                       │                              │ to unbound cardinality metrics                               │
│                                                              │                │          │        │                                       │                              │ https://avd.aquasec.com/nvd/cve-2023-47108                   │
├──────────────────────────────────────────────────────────────┼────────────────┼──────────┤        ├───────────────────────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ go.temporal.io/api                                           │ CVE-2025-1243  │ LOW      │        │ v1.18.1                               │ 1.44.1                       │ Unencrypted transmission in Temporal api-go library          │
│                                                              │                │          │        │                                       │                              │ https://avd.aquasec.com/nvd/cve-2025-1243                    │
├──────────────────────────────────────────────────────────────┼────────────────┤          │        ├───────────────────────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ go.temporal.io/server                                        │ CVE-2023-3485  │          │        │ v1.18.1-0.20230217005328-b313b7f58641 │ 1.20.0                       │ Temporal Server vulnerable to Incorrect Authorization and    │
│                                                              │                │          │        │                                       │                              │ Insecure Default Initialization of Resource...               │
│                                                              │                │          │        │                                       │                              │ https://avd.aquasec.com/nvd/cve-2023-3485                    │
├──────────────────────────────────────────────────────────────┼────────────────┼──────────┤        ├───────────────────────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ golang.org/x/net                                             │ CVE-2025-22870 │ MEDIUM   │        │ v0.35.0                               │ 0.36.0                       │ golang.org/x/net/proxy: golang.org/x/net/http/httpproxy:     │
│                                                              │                │          │        │                                       │                              │ HTTP Proxy bypass using IPv6 Zone IDs in golang.org/x/net    │
│                                                              │                │          │        │                                       │                              │ https://avd.aquasec.com/nvd/cve-2025-22870                   │
│                                                              ├────────────────┤          │        │                                       ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                                                              │ CVE-2025-22872 │          │        │                                       │ 0.38.0                       │ golang.org/x/net/html: Incorrect Neutralization of Input     │
│                                                              │                │          │        │                                       │                              │ During Web Page Generation in x/net in...                    │
│                                                              │                │          │        │                                       │                              │ https://avd.aquasec.com/nvd/cve-2025-22872                   │
├──────────────────────────────────────────────────────────────┼────────────────┤          │        ├───────────────────────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ stdlib                                                       │ CVE-2024-45336 │          │        │ v1.23.2                               │ 1.22.11, 1.23.5, 1.24.0-rc.2 │ golang: net/http: net/http: sensitive headers incorrectly    │
│                                                              │                │          │        │                                       │                              │ sent after cross-domain redirect                             │
│                                                              │                │          │        │                                       │                              │ https://avd.aquasec.com/nvd/cve-2024-45336                   │
│                                                              ├────────────────┤          │        │                                       │                              ├──────────────────────────────────────────────────────────────┤
│                                                              │ CVE-2024-45341 │          │        │                                       │                              │ golang: crypto/x509: crypto/x509: usage of IPv6 zone IDs can │
│                                                              │                │          │        │                                       │                              │ bypass URI name...                                           │
│                                                              │                │          │        │                                       │                              │ https://avd.aquasec.com/nvd/cve-2024-45341                   │
│                                                              ├────────────────┤          │        │                                       ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                                                              │ CVE-2025-22866 │          │        │                                       │ 1.22.12, 1.23.6, 1.24.0-rc.3 │ crypto/internal/nistec: golang: Timing sidechannel for P-256 │
│                                                              │                │          │        │                                       │                              │ on ppc64le in crypto/internal/nistec                         │
│                                                              │                │          │        │                                       │                              │ https://avd.aquasec.com/nvd/cve-2025-22866                   │
│                                                              ├────────────────┤          │        │                                       ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                                                              │ CVE-2025-22871 │          │        │                                       │ 1.23.8, 1.24.2               │ net/http: Request smuggling due to acceptance of invalid     │
│                                                              │                │          │        │                                       │                              │ chunked data in net/http...                                  │
│                                                              │                │          │        │                                       │                              │ https://avd.aquasec.com/nvd/cve-2025-22871                   │
└──────────────────────────────────────────────────────────────┴────────────────┴──────────┴────────┴───────────────────────────────────────┴──────────────────────────────┴──────────────────────────────────────────────────────────────┘

usr/local/bin/tctl-authorization-plugin (gobinary)

Total: 7 (UNKNOWN: 0, LOW: 1, MEDIUM: 6, HIGH: 0, CRITICAL: 0)

┌────────────────────┬────────────────┬──────────┬────────┬───────────────────┬──────────────────────────────┬──────────────────────────────────────────────────────────────┐
│      Library       │ Vulnerability  │ Severity │ Status │ Installed Version │        Fixed Version         │                            Title                             │
├────────────────────┼────────────────┼──────────┼────────┼───────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ go.temporal.io/api │ CVE-2025-1243  │ LOW      │ fixed  │ v1.18.1           │ 1.44.1                       │ Unencrypted transmission in Temporal api-go library          │
│                    │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2025-1243                    │
├────────────────────┼────────────────┼──────────┤        ├───────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ golang.org/x/net   │ CVE-2025-22870 │ MEDIUM   │        │ v0.35.0           │ 0.36.0                       │ golang.org/x/net/proxy: golang.org/x/net/http/httpproxy:     │
│                    │                │          │        │                   │                              │ HTTP Proxy bypass using IPv6 Zone IDs in golang.org/x/net    │
│                    │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2025-22870                   │
│                    ├────────────────┤          │        │                   ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2025-22872 │          │        │                   │ 0.38.0                       │ golang.org/x/net/html: Incorrect Neutralization of Input     │
│                    │                │          │        │                   │                              │ During Web Page Generation in x/net in...                    │
│                    │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2025-22872                   │
├────────────────────┼────────────────┤          │        ├───────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ stdlib             │ CVE-2024-45336 │          │        │ v1.23.2           │ 1.22.11, 1.23.5, 1.24.0-rc.2 │ golang: net/http: net/http: sensitive headers incorrectly    │
│                    │                │          │        │                   │                              │ sent after cross-domain redirect                             │
│                    │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2024-45336                   │
│                    ├────────────────┤          │        │                   │                              ├──────────────────────────────────────────────────────────────┤
│                    │ CVE-2024-45341 │          │        │                   │                              │ golang: crypto/x509: crypto/x509: usage of IPv6 zone IDs can │
│                    │                │          │        │                   │                              │ bypass URI name...                                           │
│                    │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2024-45341                   │
│                    ├────────────────┤          │        │                   ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2025-22866 │          │        │                   │ 1.22.12, 1.23.6, 1.24.0-rc.3 │ crypto/internal/nistec: golang: Timing sidechannel for P-256 │
│                    │                │          │        │                   │                              │ on ppc64le in crypto/internal/nistec                         │
│                    │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2025-22866                   │
│                    ├────────────────┤          │        │                   ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                    │ CVE-2025-22871 │          │        │                   │ 1.23.8, 1.24.2               │ net/http: Request smuggling due to acceptance of invalid     │
│                    │                │          │        │                   │                              │ chunked data in net/http...                                  │
│                    │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2025-22871                   │
└────────────────────┴────────────────┴──────────┴────────┴───────────────────┴──────────────────────────────┴──────────────────────────────────────────────────────────────┘

usr/local/bin/temporal (gobinary)

Total: 7 (UNKNOWN: 0, LOW: 0, MEDIUM: 5, HIGH: 2, CRITICAL: 0)

┌──────────────────────────────┬────────────────┬──────────┬──────────┬─────────────────────┬──────────────────────────────┬──────────────────────────────────────────────────────────────┐
│           Library            │ Vulnerability  │ Severity │  Status  │  Installed Version  │        Fixed Version         │                            Title                             │
├──────────────────────────────┼────────────────┼──────────┼──────────┼─────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ github.com/golang-jwt/jwt    │ CVE-2025-30204 │ HIGH     │ affected │ v3.2.2+incompatible │                              │ golang-jwt/jwt: jwt-go allows excessive memory allocation    │
│                              │                │          │          │                     │                              │ during header parsing                                        │
│                              │                │          │          │                     │                              │ https://avd.aquasec.com/nvd/cve-2025-30204                   │
├──────────────────────────────┤                │          ├──────────┼─────────────────────┼──────────────────────────────┤                                                              │
│ github.com/golang-jwt/jwt/v4 │                │          │ fixed    │ v4.5.1              │ 4.5.2                        │                                                              │
│                              │                │          │          │                     │                              │                                                              │
│                              │                │          │          │                     │                              │                                                              │
├──────────────────────────────┼────────────────┼──────────┤          ├─────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ golang.org/x/net             │ CVE-2025-22872 │ MEDIUM   │          │ v0.36.0             │ 0.38.0                       │ golang.org/x/net/html: Incorrect Neutralization of Input     │
│                              │                │          │          │                     │                              │ During Web Page Generation in x/net in...                    │
│                              │                │          │          │                     │                              │ https://avd.aquasec.com/nvd/cve-2025-22872                   │
├──────────────────────────────┼────────────────┤          │          ├─────────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ stdlib                       │ CVE-2024-45336 │          │          │ v1.23.2             │ 1.22.11, 1.23.5, 1.24.0-rc.2 │ golang: net/http: net/http: sensitive headers incorrectly    │
│                              │                │          │          │                     │                              │ sent after cross-domain redirect                             │
│                              │                │          │          │                     │                              │ https://avd.aquasec.com/nvd/cve-2024-45336                   │
│                              ├────────────────┤          │          │                     │                              ├──────────────────────────────────────────────────────────────┤
│                              │ CVE-2024-45341 │          │          │                     │                              │ golang: crypto/x509: crypto/x509: usage of IPv6 zone IDs can │
│                              │                │          │          │                     │                              │ bypass URI name...                                           │
│                              │                │          │          │                     │                              │ https://avd.aquasec.com/nvd/cve-2024-45341                   │
│                              ├────────────────┤          │          │                     ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                              │ CVE-2025-22866 │          │          │                     │ 1.22.12, 1.23.6, 1.24.0-rc.3 │ crypto/internal/nistec: golang: Timing sidechannel for P-256 │
│                              │                │          │          │                     │                              │ on ppc64le in crypto/internal/nistec                         │
│                              │                │          │          │                     │                              │ https://avd.aquasec.com/nvd/cve-2025-22866                   │
│                              ├────────────────┤          │          │                     ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                              │ CVE-2025-22871 │          │          │                     │ 1.23.8, 1.24.2               │ net/http: Request smuggling due to acceptance of invalid     │
│                              │                │          │          │                     │                              │ chunked data in net/http...                                  │
│                              │                │          │          │                     │                              │ https://avd.aquasec.com/nvd/cve-2025-22871                   │
└──────────────────────────────┴────────────────┴──────────┴──────────┴─────────────────────┴──────────────────────────────┴──────────────────────────────────────────────────────────────┘

usr/local/bin/temporal-server (gobinary)

Total: 5 (UNKNOWN: 0, LOW: 0, MEDIUM: 5, HIGH: 0, CRITICAL: 0)

┌──────────────────┬────────────────┬──────────┬────────┬───────────────────┬──────────────────────────────┬──────────────────────────────────────────────────────────────┐
│     Library      │ Vulnerability  │ Severity │ Status │ Installed Version │        Fixed Version         │                            Title                             │
├──────────────────┼────────────────┼──────────┼────────┼───────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ golang.org/x/net │ CVE-2025-22872 │ MEDIUM   │ fixed  │ v0.37.0           │ 0.38.0                       │ golang.org/x/net/html: Incorrect Neutralization of Input     │
│                  │                │          │        │                   │                              │ During Web Page Generation in x/net in...                    │
│                  │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2025-22872                   │
├──────────────────┼────────────────┤          │        ├───────────────────┼──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ stdlib           │ CVE-2024-45336 │          │        │ v1.23.2           │ 1.22.11, 1.23.5, 1.24.0-rc.2 │ golang: net/http: net/http: sensitive headers incorrectly    │
│                  │                │          │        │                   │                              │ sent after cross-domain redirect                             │
│                  │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2024-45336                   │
│                  ├────────────────┤          │        │                   │                              ├──────────────────────────────────────────────────────────────┤
│                  │ CVE-2024-45341 │          │        │                   │                              │ golang: crypto/x509: crypto/x509: usage of IPv6 zone IDs can │
│                  │                │          │        │                   │                              │ bypass URI name...                                           │
│                  │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2024-45341                   │
│                  ├────────────────┤          │        │                   ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                  │ CVE-2025-22866 │          │        │                   │ 1.22.12, 1.23.6, 1.24.0-rc.3 │ crypto/internal/nistec: golang: Timing sidechannel for P-256 │
│                  │                │          │        │                   │                              │ on ppc64le in crypto/internal/nistec                         │
│                  │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2025-22866                   │
│                  ├────────────────┤          │        │                   ├──────────────────────────────┼──────────────────────────────────────────────────────────────┤
│                  │ CVE-2025-22871 │          │        │                   │ 1.23.8, 1.24.2               │ net/http: Request smuggling due to acceptance of invalid     │
│                  │                │          │        │                   │                              │ chunked data in net/http...                                  │
│                  │                │          │        │                   │                              │ https://avd.aquasec.com/nvd/cve-2025-22871                   │
└──────────────────┴────────────────┴──────────┴────────┴───────────────────┴──────────────────────────────┴──────────────────────────────────────────────────────────────┘
```

---

See also:

- [Trivy](https://trivy.dev)
- [Cover Image: Photo by  Mathew Schwartz](https://unsplash.com/photos/white-flower-illustration--MY7-K4X5C0)
