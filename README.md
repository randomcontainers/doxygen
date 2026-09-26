# doxygen

Container images with [Doxygen](https://www.doxygen.nl/), which generates HTML, LaTeX, man page and XML documentation from comments in C++, C, Java, Python and other source code. `latest` also includes Graphviz, whose `dot` program draws Doxygen's class, collaboration, include and call graphs. Doxygen is compiled from the release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when Doxygen publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the Doxygen project. Report problems with the image in this repository and Doxygen bugs [upstream](https://github.com/doxygen/doxygen/issues).

## Quick start

Generate the documentation that the `Doxyfile` in the current directory describes:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/doxygen Doxyfile
```

The same images can also be pulled as `randomcontainers.com/doxygen`.

The entrypoint runs `doxygen` under `tini` in `/work`, and without arguments it prints Doxygen's help. Paths in the Doxyfile, such as `INPUT` and `OUTPUT_DIRECTORY`, are relative to `/work`, the directory you mount.

Write a template Doxyfile to start a new project from:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/doxygen -g Doxyfile
```

Doxygen reads the configuration from standard input when the file name is `-`. When a tag is assigned more than once, the last assignment wins, so a job can change a setting for one run without editing the Doxyfile:

```sh
{ cat Doxyfile; echo "PROJECT_NUMBER = 2.1"; } | docker run --rm -i --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/doxygen -
```

Doxygen runs `dot` only when the Doxyfile sets `HAVE_DOT = YES` (the default is `NO`), and only the default image has it. Without it, Doxygen draws class diagrams itself. The [Doxygen manual](https://www.doxygen.nl/manual/config.html) covers every configuration tag.

## What is in the image

| | slim | default |
|---|---|---|
| `doxygen`, with its HTML and LaTeX templates built in | yes | yes |
| Graphviz, for the graphs of `HAVE_DOT = YES` and for `\dot` blocks in comments | no | yes |

Graphviz is the [randomcontainers/graphviz](https://github.com/randomcontainers/graphviz) build.

Doxygen is built with the copies of fmt, spdlog and SQLite that come with its source, as in upstream's own builds. Its built-in renderers draw class diagrams and `\msc` message sequence charts without other programs, and `GENERATE_SQLITE3` works in both images. On Alpine, `INPUT_ENCODING` conversions go through musl's iconv, which knows fewer encodings than the glibc one on Ubuntu.

Not included: doxywizard (the GUI), doxysearch and doxyindexer for server-side search, and parsing with libclang (`CLANG_ASSISTED_PARSING`). Some settings call programs that are not in the images either:

- LaTeX output, which a new Doxyfile turns on, is written as `.tex` files; building the PDF needs a TeX distribution. With the default `USE_PDFLATEX = YES`, Doxygen also runs `epstopdf` for each built-in class diagram and `\msc` chart in the LaTeX output and reports an error when it is missing. Set `GENERATE_LATEX = NO` if you only need HTML, or `USE_PDFLATEX = NO` to keep those images as EPS.
- Formulas in HTML output need LaTeX unless `USE_MATHJAX = YES`.
- `\cite` with `CITE_BIB_FILES` needs `perl` and `bibtex`.
- `\startuml` and `\diafile` need PlantUML and Dia. `\msc` needs no external `mscgen`.

The CMake options are in `/usr/local/share/randomcontainers/doxygen/buildinfo`.

## Default or slim

The default image (`latest`) has Doxygen and Graphviz and is recommended for general use: most Doxyfiles that draw graphs set `HAVE_DOT = YES` and expect `dot` on the `PATH`. Use `slim`, which has Doxygen only, when you do not need those graphs, or as the base when you build your own image.

The default image is also published as `ghcr.io/randomcontainers/doxygen-graphviz`, built in the [doxygen-graphviz](https://github.com/randomcontainers/doxygen-graphviz) repository with the same contents and a different digest.

## Tags

`<version>` is a Doxygen release such as `1.18.0`. `<minor>` and `<major>` are its shorter forms, `1.18` and `1`, and follow the newest release in that series.

| Default (with Graphviz) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Doxygen version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

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

An absolute path from the host, in `INPUT`, `EXAMPLE_PATH` or `IMAGE_PATH` for example, does not exist in the container. Make it relative to the mounted directory, or mount that directory at the same path with another `-v`.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Doxygen release and are rebuilt when the base image changes. The default tags work the same way when your image also needs Graphviz. Switch to root to install more, for example `git` for a `FILE_VERSION_FILTER` that asks git for each file's last commit, then back:

```dockerfile
FROM ghcr.io/randomcontainers/doxygen:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends git \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache git`. The entrypoint is `["tini", "--", "doxygen"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new Doxygen releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/doxygen/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages Doxygen needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/doxygen:latest \
  --repo randomcontainers/doxygen --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/doxygen-graphviz` are built in that repository, so verify them with `--repo randomcontainers/doxygen-graphviz` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/doxygen:latest --format '{{ json .SBOM }}'
```

Doxygen publishes no signatures or checksum files. The SHA-256 in `package.yml` is the digest GitHub records for the `doxygen-<version>.src.tar.gz` release asset, which is also the hash on the [Doxygen download page](https://www.doxygen.nl/download.html). Before compiling, the build checks the tarball against it.

## Updates

The project checks the releases of [doxygen/doxygen](https://github.com/doxygen/doxygen/releases) every 15 minutes. A release is picked up once it is 24 hours old. Its source tarball is checked against the digest GitHub records for the release asset, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new Graphviz image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t doxygen:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci) and layers this image onto the Graphviz image.

## Licenses

Doxygen is licensed under the GNU General Public License. Its source files name no version and the tarball's `LICENSE` is version 2, so the image is labeled GPL-2.0-or-later. Code compiled into `doxygen` and the templates built into it include parts under other licenses:

- MIT: fmt, spdlog, ghc::filesystem, svg.js, the doxygen-awesome code in the HTML scripts, and Doxygen's own HTML scripts (`fmt.license.rst`, `LICENSE.spdlog`, `COPYING.filesystem.hpp`, `COPYING.svg-<version>.js`, `COPYING.clipboard.js`, `COPYING.dynsections.js`).
- Zlib: LodePNG and TinyDeflate (`COPYING.lodepng.cpp`, `COPYING.gunzip.hh`).
- BSD-2-Clause-Views: SVGPan, which zooms and pans SVG graphs in HTML output (`COPYING.svgpan-<version>.js`).
- Apache-2.0: the Material Symbols font of the HTML output (`MaterialSymbolsOutlined_LICENSE.txt`).
- LPPL-1.3c: etoc, which the LaTeX output uses as `etoc_doxygen.sty` (`COPYING.etoc_doxygen.sty`).
- blessing: SQLite (`COPYING.sqlite3.c`).
- GPL-2.0-or-later: mscgen, which draws `\msc` charts (`COPYING.mscgen_adraw.c`). The `bib2xhtml.pl` script for `\cite` is under the GPL with no version given (`COPYING.bib2xhtml.pl`).
- Permissive notices that have no SPDX identifier: the part of the GD graphics library that mscgen uses (`COPYING.gd.h`) and the BibTeX style `doxygen.bst` (`COPYING.doxygen.bst`).
- Public domain: the MD5 code (`COPYING.md5.c`).

The files named above are in `/usr/local/share/randomcontainers/doxygen/licenses/`, with Doxygen's `LICENSE`. The image's license label is `GPL-2.0-or-later AND Apache-2.0 AND BSD-2-Clause-Views AND LPPL-1.3c AND MIT AND Zlib AND blessing`. Doxygen's license does not cover the documents it generates, but HTML and LaTeX output contain some of the files above, such as the scripts, the font and `etoc_doxygen.sty`, under their own licenses. The Ubuntu and Alpine packages in the image keep their own licenses.

The corresponding source for each image:

- Doxygen: every version has a GitHub release in this repository, named `v<version>`, with the exact `doxygen-<version>.src.tar.gz` that was compiled. The build applies no patches. The download URL is in `/usr/local/share/randomcontainers/doxygen/source`.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every CMake option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

`latest` also contains Graphviz, which is mainly licensed under the Eclipse Public License 2.0 (EPL-2.0). Its license files and corresponding source are described in the [graphviz repository](https://github.com/randomcontainers/graphviz#licenses).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
