# qpdf

Container images with [qpdf](https://qpdf.sourceforge.io/), the command-line tool and C++ library for content-preserving changes to PDF files: merging, splitting, rotating, encrypting, decrypting, linearizing and repairing them. qpdf is compiled from the signed release tarball on Ubuntu and Alpine, with OpenSSL for encryption. `latest` also includes Ghostscript, for jobs that render or rewrite page content. The images are rebuilt when qpdf publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the qpdf project. Report problems with the image in this repository and problems with qpdf itself [upstream](https://github.com/qpdf/qpdf/issues).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/qpdf --empty --pages first.pdf second.pdf -- merged.pdf
```

Encrypt a PDF with AES-256, so that it needs a password to open:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/qpdf \
  --encrypt --user-password=open-me --owner-password=change-me --bits=256 -- input.pdf encrypted.pdf
```

The entrypoint runs `qpdf` under `tini` in `/work`, so file names are relative to the directory you mount. A few more commands:

```sh
# Split a PDF into one file per page, numbered from 1
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/qpdf \
  --split-pages input.pdf page-%d.pdf

# Keep pages 1 to 3 and the last page
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/qpdf \
  --empty --pages input.pdf 1-3,z -- excerpt.pdf

# Linearize a PDF for fast web view
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/qpdf \
  --linearize input.pdf web.pdf

# Remove the encryption from a PDF you have the password for
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/qpdf \
  --password=open-me --decrypt encrypted.pdf decrypted.pdf

# Check the structure of a PDF: exit status 0 means no problems, 2 errors, 3 warnings
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/qpdf \
  --check input.pdf
```

Passwords on the command line show up in the host's process list and shell history. `--password-file=<file>` reads the password for opening a file from `<file>`, and `@<file>` reads further arguments from `<file>`, one per line, which also works for `--encrypt`.

`fix-qdf` and `zlib-flate` are in the image too. Write a PDF in QDF form, edit it in a text editor, then let `fix-qdf` repair the offsets and stream lengths:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/qpdf \
  --qdf --object-streams=disable input.pdf input.qdf
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint fix-qdf \
  ghcr.io/randomcontainers/qpdf input.qdf > edited.pdf
```

The default image also has Ghostscript. Run it with `--entrypoint gs`, for example to make a PDF smaller before linearizing it with qpdf:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint gs ghcr.io/randomcontainers/qpdf \
  -sDEVICE=pdfwrite -dPDFSETTINGS=/ebook -o smaller.pdf input.pdf
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/qpdf \
  --linearize smaller.pdf web.pdf
```

The [qpdf manual](https://qpdf.readthedocs.io/) covers every option.

## What is in the image

| | slim | default |
|---|---|---|
| `qpdf`, `fix-qdf`, `zlib-flate` and the `libqpdf` shared library | yes | yes |
| Ghostscript: `gs`, `ps2pdf` and the other helper scripts, with the URW base 35 fonts | no | yes |

Ghostscript is the [randomcontainers/ghostscript](https://github.com/randomcontainers/ghostscript) build.

qpdf uses the distro's OpenSSL for encryption and decryption, libjpeg-turbo for `--optimize-images` and for decoding JPEG images with `--decode-level=all`, and zlib. Older encrypted files, with 40-bit keys or 128-bit keys without AES, use RC4, which comes from OpenSSL's legacy provider. The images include it (`openssl-provider-legacy` on Ubuntu, part of `libcrypto3` on Alpine). `qpdf --show-crypto` prints `openssl`.

Not included: qpdf's built-in crypto code and the GnuTLS provider, zopfli, the C++ headers and the pkg-config and CMake files for building against `libqpdf`, the example programs, and the manual, which is [online](https://qpdf.readthedocs.io/). The CMake options are in `/usr/local/share/randomcontainers/qpdf/buildinfo`.

## Default or slim

Use the default image (`latest`) when a job needs both tools: Ghostscript renders pages, converts PostScript and makes PDFs smaller by resampling their images, and qpdf merges, splits, encrypts or linearizes the result. `slim` has qpdf and the libraries it needs, without Ghostscript; use it to build your own image, or when you only need qpdf. The default image of [WeasyPrint](https://github.com/randomcontainers/weasyprint) includes the `slim` build.

The default image includes Ghostscript, which is licensed under the GNU Affero General Public License (AGPL-3.0-or-later). Use `slim` if your policy excludes AGPL. The default image is also published as `ghcr.io/randomcontainers/qpdf-ghostscript`, built in the [qpdf-ghostscript](https://github.com/randomcontainers/qpdf-ghostscript) repository with the same contents and a different digest.

## Tags

`<version>` is a qpdf release such as `12.4.1`. `<minor>` and `<major>` are its shorter forms, `12.4` and `12`, and follow the newest release in that series.

| Default (with Ghostscript) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current qpdf version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

With `--replace-input`, qpdf writes the new file next to the input and then renames it over the input, so the mounted directory must be writable.

## Untrusted files

qpdf reads the structure of a PDF and does not run scripts or other code from it. For files from unknown sources, still take away what the container does not need:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --network none --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 1g --pids-limit 64 \
  ghcr.io/randomcontainers/qpdf --check untrusted.pdf
```

In the default image, Ghostscript keeps its `-dSAFER` sandbox on. Read the [ghostscript README](https://github.com/randomcontainers/ghostscript#untrusted-files) before you run `gs` on such files, and add `--tmpfs /tmp` to the command above, because Ghostscript writes temporary files there.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no Ghostscript, so Ghostscript updates do not rebuild it. The packages qpdf needs are listed in `/usr/local/share/randomcontainers/qpdf/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/qpdf:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends poppler-utils \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, use `apk add --no-cache poppler-utils`. The entrypoint is `["tini", "--", "qpdf"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new qpdf releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/qpdf:latest \
  --repo randomcontainers/qpdf --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/qpdf-ghostscript` are built in that repository, so verify them with `--repo randomcontainers/qpdf-ghostscript` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/qpdf:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against Jay Berkenbilt's qpdf release signing key in `keys/qpdf-release.gpg` (fingerprint `C2C9 6B10 011F E009 E6D1 DF82 8A75 D109 9801 2C7E`), which the qpdf README names.

## Updates

The project checks the releases of [qpdf/qpdf](https://github.com/qpdf/qpdf/releases) every 15 minutes. A release is picked up once it is 24 hours old and its signature is published. Its tarball is checked against the release's `qpdf-<version>.sha256` file and the digest GitHub records for the asset, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new Ghostscript image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t qpdf:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

qpdf is licensed under the Apache License, version 2.0 (Apache-2.0). Its `NOTICE.md` adds that, at your option, you may continue to consider qpdf licensed under the Artistic License 2.0, the license of qpdf versions before 7. `LICENSE.txt`, `NOTICE.md` and `Artistic-2.0` are in `/usr/local/share/randomcontainers/qpdf/licenses/`, and the URLs of the source tarball and its signature are in `/usr/local/share/randomcontainers/qpdf/source`. The public-domain Rijndael code and the sphlib SHA-2 code that `NOTICE.md` lists belong to qpdf's built-in crypto, which these images do not compile.

The default image adds Ghostscript, licensed under AGPL-3.0-or-later. Its license files and corresponding source are described in the [ghostscript repository](https://github.com/randomcontainers/ghostscript#licenses). The libraries from Ubuntu or Alpine, such as OpenSSL, libjpeg-turbo and zlib, keep their own licenses; the SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
