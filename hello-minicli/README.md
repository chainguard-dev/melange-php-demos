# Minicli melange + apko Demo

This demo creates an APK package for a minimalist CLI application in PHP written with [minicli](https://github.com/minicli/minicli), using [melange](https://github.com/chainguard-dev/melange).

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
docker run --rm -v "${PWD}":/work distroless.dev/melange keygen
```
This will generate a `melange.rsa` and `melange.rsa.pub` files in the current directory.

```
2022/08/05 14:46:05 generating keypair with a 4096 bit prime, please wait...
2022/08/05 14:46:08 wrote private key to melange.rsa
2022/08/05 14:46:08 wrote public key to melange.rsa.pub
```

### 2. Building the APK

Next, build the APK defined in the `melange.yaml` file:

```shell
docker run --privileged --rm -v "${PWD}":/work distroless.dev/melange build melange.yaml --arch x86,amd64 --keyring-append melange.rsa
```
This should get you `x86` and `x86_64` APK packages at the location `./packages`:

```
packages
├── x86
│   └── hello-minicli-0.1.0-r0.apk
└── x86_64
    └── hello-minicli-0.1.0-r0.apk

2 directories, 2 files

```

### 3. Creating the APK index
Before using the generated APKs you'll need to build an APK index from your packages. Run the following command, which will loop through your `packages` folder, generate an index, and sign it with the melange keypair:

```shell
docker run --rm -v "${PWD}":/work \
    --entrypoint sh \
    distroless.dev/melange -c \
        'cd packages && for d in `find . -type d -mindepth 1`; do \
            ( \
                cd $d && \
                apk index -o APKINDEX.tar.gz *.apk && \
                melange sign-index --signing-key=../../melange.rsa APKINDEX.tar.gz\
            ) \
        done'
```

For each architecture you've built your package, you should get output like this:

```
Index has 1 packages (of which 1 are new)
2022/08/05 14:57:29 signing index APKINDEX.tar.gz with key ../../melange.rsa
2022/08/05 14:57:29 appending signature to index APKINDEX.tar.gz
2022/08/05 14:57:29 writing signed index to APKINDEX.tar.gz
2022/08/05 14:57:29 signed index APKINDEX.tar.gz with key ../../melange.rsa
```
_Note: this step will be automated in the near future._

### 4. Building a container image with apko

With the APK packages and APK index in place, you can now build a container image and have your APK(s) installed within it.

```shell
docker run --rm -v ${PWD}:/work distroless.dev/apko build /work/apko.yaml hello-minicli:test /work/hello-minicli.tar -k melange.rsa.pub
```
This will build an OCI image based on your host system's architecture - most likely this will be `x86_64`. 

The command will generate a few new files in the app's directory:

- `hello-minicli.tar` - the packaged OCI image that can be imported with a `docker load` command
- `sbom-x86_64.cdx` - an SBOM file for `x86_64` architecture in `cdx` format
- `sbom-x86_64.spdx.json` - an SBOM file for `x86_64` architecture in `spdx-json` format

### 5. Importing and running your image with Docker

Load your image within Docker:

```shell
docker load < hello-minicli.tar
```
```
10f951ac3cd2: Loading layer [==================================================>]  7.764MB/7.764MB
Loaded image: hello-minicli:test
```
Now you can run your Minicli program with:

```shell
docker run --rm hello-minicli:test
```
The demo should output an advice slip such as:

```
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
### Common errors

>Error: failed to build layer image: initializing apk: failed to fixate apk world: exit status 1

This error usually means one of the requested APK packages could not be resolved. 

For Alpine packages, make sure you've added the relevant package repositories you need and the package name is correct - search the [Alpine APK index](https://pkgs.alpinelinux.org/packages) for reference.

If the issue is with your melange-built package(s), make sure you built the APK index (described in step 3).

## Optional Steps

Additional steps and debug tips can be found [here](https://github.com/chainguard-dev/hello-melange-apko).

Quick links:
- [Push the image to a container registry](https://github.com/chainguard-dev/hello-melange-apko#push-image-with-apko)
- [Sign the image with cosign](https://github.com/chainguard-dev/hello-melange-apko#sign-image-with-cosign)
- [Verify the signature](https://github.com/chainguard-dev/hello-melange-apko#verify-the-signature)
