- Contribution Name: ARM Support
- Implementation Owner: [@aliariff](https://github.com/aliariff)
- Start Date: 2020-02-23
- Target Date:
- RFC PR: [linkerd/gsoc#1](https://github.com/linkerd/gsoc/pull/1)
- Linkerd Issue: [linkerd/linkerd2#1165](https://github.com/linkerd/linkerd2/issues/1165), [linkerd/linkerd2#3114](https://github.com/linkerd/linkerd2/issues/3114)
- Reviewers:

# Summary

[summary]: #summary

This project goal is to make Linkerd officially support ARM architecture (`arm` & `arm64`).

# Problem Statement

[problem-statement]: #problem-statement

Currently, Linkerd is only supporting `amd64` architecture officially. Supporting other architectures will make Linkerd more powerful and attractive for the users.

After the project implemented, releasing process will have more steps:
- Build docker images in ARM architecture
- Integration test for the resulting images
- Publish the images

# Design proposal

[design-proposal]: #design-proposal

There are 3 repositories need to change to support `arm`:

1. [linkerd2-proxy-init](https://github.com/linkerd/linkerd2-proxy-init)
2. [linkerd2-proxy](https://github.com/linkerd/linkerd2-proxy)
3. [linkerd2](https://github.com/linkerd/linkerd2)

## Build Strategy

### Option 1: Introduce ENV key
Adding a new key to define what architecture the image will be build, and use that key for indicator when performing cross compilation.

- `linkerd2-proxy-init`
  - Add Makefile
  - Define `build` instruction in Makefile to build & apply appropriate tag
    ```makefile
    docker build -t gcr.io/linkerd-io/proxy-init-${DOCKER_BUILD_ARCH:-amd64}:... \
      --build-arg "GOARCH=${DOCKER_BUILD_ARCH:-amd64} .
    ```
  - Define `push-all` instruction in Makefile to push the image & multi-arch manifest
    ```makefile
    docker push gcr.io/linkerd-io/proxy-init-amd64:...
    docker push gcr.io/linkerd-io/proxy-init-arm64:...
    docker push gcr.io/linkerd-io/proxy-init-arm:...

    docker manifest create gcr.io/linkerd-io/proxy-init:... \
      gcr.io/linkerd-io/proxy-init-amd64:... \
      gcr.io/linkerd-io/proxy-init-arm64:... \
      gcr.io/linkerd-io/proxy-init-arm:...
    ```
  - Use the `GOARCH` argument to perform cross compile in `Dockerfile`
    ```Dockerfile
    ARG GOARCH
    RUN CGO_ENABLED=0 GOOS=linux GOARCH=$GOARCH go build ...
    ```
  - Change the runtime image in `Dockerfile`
    ```Dockerfile
    FROM --platform=$GOARCH debian:stretch-20190812-slim
    ```
  - Add Github actions
  - Call the build inside github workflow
     ```yml
     make build # this will build the amd64 image
     DOCKER_BUILD_ARCH=arm64 make build # building arm64 image
     DOCKER_BUILD_ARCH=arm make build # building arm image
     ```
  - Push image & push multi arch manifest
    ```yml
    make push-all
    ```

- `linkerd2-proxy`
  - Add new step in `.github/workflows/release.yml`
  - Install requirement to perform cross compilation in Rust
    ```yml
    sudo apt-get install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
    ```
  - Set the appropriate environment
    ```yml
    env:
      CARGO_TARGET_ARCH: armv7-unknown-linux-gnueabihf # for arm
      CARGO_RELEASE: "1"
    run: make package

    or

    env:
      CARGO_TARGET_ARCH: aarch64-unknown-linux-gnu # for arm64
      CARGO_RELEASE: "1"
    run: make package
    ```
  - Change the Makefile to check for `CARGO_TARGET_ARCH` to perform cross compile
    ```makefile
    .
    .
    CARGO_BUILD = $(CARGO) build --frozen $(RELEASE) --target=$(CARGO_TARGET_ARCH)
    .
    .
    ```
  - Publish the artifact together with other architecture

- `linkerd2`

  There are 7 images to build: `controller`, `cli`, `cni-plugin`, `web`, `proxy`, `debug`, `grafana`

  - Export `DOCKER_BUILD_ARCH` in `bin/_docker.sh` & append the architecture in the repo name
    ```sh
    export DOCKER_BUILD_ARCH="${DOCKER_BUILD_ARCH:-}"
    .
    .
    .
    docker_repo() {
      .
      .
      .
      if [ -n "${DOCKER_BUILD_ARCH:-}" ]; then
        name="$name-$DOCKER_BUILD_ARCH"
      fi
      .
      .
      .
    }
    ```
  - Pass the architecture argument to docker build in every `docker-build-*` file
    ```sh
    docker_build .... --build-arg "GOARCH=${DOCKER_BUILD_ARCH:-amd64}"
    ```
  - Use the `GOARCH` argument to perform cross compile in all `Dockerfile`
    ```Dockerfile
    ARG GOARCH
    RUN CGO_ENABLED=0 GOOS=linux GOARCH=$GOARCH go build ...
    ```
  - Change the runtime image in `Dockerfile`
    ```Dockerfile
    FROM --platform=$GOARCH ...
    ```
  - Add `bin/docker-manifest` file to create and push multi-arch manifest
    ```sh
    for img in cli-bin cni-plugin controller debug grafana proxy web ; do
      docker manifest create gcr.io/linkerd-io/$img:$tag \
        gcr.io/linkerd-io/$img-amd64:$tag \
        gcr.io/linkerd-io/$img-arm64:$tag \
        gcr.io/linkerd-io/$img-arm:$tag
    done
    ```
  - Update `.github/workflows/release.yml`
    ```yml
    bin/docker-build
    DOCKER_BUILD_ARCH=arm64 bin/docker-build
    DOCKER_BUILD_ARCH=arm bin/docker-build
    .
    .
    .
    bin/docker-push $TAG
    DOCKER_BUILD_ARCH=arm64 bin/docker-push $TAG
    DOCKER_BUILD_ARCH=arm bin/docker-push $TAG

    bin/docker-manifest $TAG
    ```

### Option 2: Docker Buildx
Using the ... experimental feature of docker buildx will make things easier, but it comes also with some drawback.
There are 3 strategies that we can do to build the image.

1. QEMU
- Advantage: There is no need to configure

2. Remote Docker Machine with `arm` architecture

3. Cross-compile

## Test Strategy

# Prior art

[prior-art]: #prior-art

This project is already done in many CNCF projects like:
- [kubernetes/kubernetes#17981](https://github.com/kubernetes/kubernetes/issues/17981)
- [grafana/grafana#13186](https://github.com/grafana/grafana/issues/13186)
- [containous/maesh#206](https://github.com/containous/maesh/issues/206)

And also proof that Linkerd can be run on ARM [linkerd/linkerd2#1165](https://github.com/linkerd/linkerd2/issues/1165#issuecomment-515470739)

# Unresolved questions

[unresolved-questions]: #unresolved-questions

# Future possibilities

[future-possibilities]: #future-possibilities
