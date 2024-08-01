# Minicli melange + apko Demo

This demo creates an APK package for a minimalist CLI application in PHP written with [minicli](https://github.com/minicli/minicli), using [melange](https://github.com/chainguard-dev/melange). Check the [Getting Started with melange](https://edu.chainguard.dev/open-source/build-tools/melange/getting-started-with-melange/) guide for detailed step-by-step instructions and explanations around how to run this demo.

The demo app uses the [advice slip API](https://api.adviceslip.com/) to output a random piece of advice.

With the included `apko.yaml` file, you can create a minimalist container image using [apko](https://github.com/chainguard-dev/apko) and have the melange-generated APK installed within your image.

## Requirements

You'll need Docker to run this demo.

## Building and running the demo

The demo app is a single script file named `minicli`, using dependencies defined in the `composer.json` file.

### 1. Creating a temporary melange keypair

First make sure you're at the `/hello-minicli` directory.

To get started, create a temporary keypair to sign your melange packages:

```shell
docker run --rm -v "${PWD}":/work cgr.dev/chainguard/melange keygen
```

This will generate a `melange.rsa` and `melange.rsa.pub` files in the current directory.

```text
2024/08/01 16:55:31 INFO generating keypair with a 4096 bit prime, please wait...
2024/08/01 16:55:33 INFO wrote private key to melange.rsa
2024/08/01 16:55:33 INFO wrote public key to melange.rsa.pub
```

### 2. Building the APK

Next, build the APK defined in the `melange.yaml` file:

```shell
docker run --privileged --rm -v "${PWD}":/work \                         
  cgr.dev/chainguard/melange build melange.yaml \
  --arch amd64,aarch64 \
  --signing-key melange.rsa

```

This should get you `aarch64` and `x86_64` APK packages at the location `./packages`:

```text
packages
├── aarch64
│   ├── APKINDEX.json
│   ├── APKINDEX.tar.gz
│   └── hello-minicli-0.1.0-r0.apk
└── x86_64
    ├── APKINDEX.json
    ├── APKINDEX.tar.gz
    └── hello-minicli-0.1.0-r0.apk

3 directories, 6 files

```

### 3. Building a container image with apko

With the APK packages and APK index in place, you can now build a container image and have your APK(s) installed within it.

```shell
docker run --rm --workdir /work -v ${PWD}:/work cgr.dev/chainguard/apko \
  build apko.yaml hello-minicli:test hello-minicli.tar \
  --arch host
```

This will build an OCI image based on your host system's architecture - most likely this will be `x86_64`.

The command will generate a few new files in the app's directory:

- `hello-minicli.tar` - the packaged OCI image that can be imported with a `docker load` command
- `sbom-x86_64.spdx.json` - an SBOM file for `x86_64` architecture in `spdx-json` format

### 4. Importing and running your image with Docker

Load your image within Docker:

```shell
docker load < hello-minicli.tar
```

```text
10f951ac3cd2: Loading layer [==================================================>]  7.764MB/7.764MB
Loaded image: hello-minicli:test
```

Now you can run your Minicli program with:

```shell
docker run --rm hello-minicli:test
```

The demo should output an advice slip such as:

```text
Don't feed Mogwais after midnight.
```

## Debugging

### melange

To include debug-level information on melange builds, edit your `melange.yaml` file and include `set -x` in your pipeline:

```yaml
...
pipeline:
  - name: Build Minicli application
    runs: |
      set -x
      APP_HOME="${{targets.destdir}}/usr/share/hello-minicli"
...
```

### apko

To include debug-level information on apko builds, add `--debug` to your build command:

```shell
docker run --rm -v ${PWD}:/work distroless.dev/apko build --debug /work/apko.yaml hello-minicli:test /work/hello-minicli.tar -k melange.rsa.pub
```
