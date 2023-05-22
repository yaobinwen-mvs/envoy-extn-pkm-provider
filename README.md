# Envoy Extension for Private Key Method (PKM) Provider

Envoy extension for private key method provider.

If you have multiple applications (using different SSL libraries) that need to connect to a server and do TLS mutual auth using a client certificate whose private key is in TPM, then you can use envoyproxy using this extension and have all the apps connect via this proxy. 

## 0. Current status (as of 2023-05-22)

- I have successfully built the PKM extension against Envoy using the following configuration:
  - Ubuntu 22.04 (using GCC 11, because some code needs GCC >= 9+).
  - Envoy: `v1.26.1` (although we are still using 1.22 in the work)
  - Bazel: `6.2.0` (see `.bazelversion`) (the Bazel version that built the original repository was too old)
- Build steps:
  - Run section 2.1 to install the required external packages.
  - Run `ci/do_ci.sh build` (as mentioned in section 2.2).
    - I haven't tried `ci/docker_build.sh` yet.
- Follow section 4.2 to run Envoy using `proxy.yaml`.
- See section 6 "TODOs" for the things to do.

## 1. Check out `envoy` submodule (updated 2023-05-22)

There are two ways to check out the `envoy` submodule:
- 1). `clone` + `submodule`:
  - a). `git clone <this repository>`.
  - b). `cd envoy-extn-pkm-provider`.
  - c). `git submodule init`.
  - d). `git submodule update`.
- 2). `clone --recurse-submodules`:
  - a). `git clone --recurse-submodules <this repository>`.

## 2. Build

### 2.1 Requirement (updated 2023-05-22)

* bazel: [Installing Bazel on Ubuntu](https://bazel.build/install/ubuntu)
* libtspi-dev (apt-get install -y libtspi-dev)
* apache2 (apt-get install -y apache2): This is used as an HTTP server for testing.
* The following items:

```
sudo apt-get install \
   autoconf \
   cmake \
   curl \
   libtool \
   ninja-build \
   patch \
   python3-pip \
   unzip \
   virtualenv \
   protobuf-compiler \
   libprotobuf-dev
```

### 2.2 Build (updated 2023-05-22)

* ci/do_ci.sh build
* ci/docker_build.sh (ywen: didn't try yet)

### 2.3 Output (updated 2023-05-22)

Once the build is complete, the result executable file `envoy` is under `bazel-bin`:

```
-r-xr-xr-x  1 ywen ywen 318302512 May 19 21:28 envoy*
-r-xr-xr-x  1 ywen ywen    677952 May 18 18:33 envoy-2.params*
drwxrwxr-x  3 ywen ywen      4096 May 19 14:25 envoy.runfiles/
-r-xr-xr-x  1 ywen ywen       390 May 18 18:18 envoy.runfiles_manifest*
drwxrwxrwx 40 ywen ywen      4096 May 19 13:26 external/
drwxrwxrwx  4 ywen ywen      4096 May 19 13:53 _objs/
drwxrwxr-x  3 ywen ywen      4096 May 18 18:33 _solib_k8/
```

## 3. Configure (updated 2023-05-22)

`config/tpm_proxy.yaml` is the configuration file of the original repository.

I am using `config/proxy.yaml` to test to use Envoy + TPM. This configuration file uses the `xDS` sub-directory for further configuration.

## 4. Run

### 4.1 Original instructions (I haven't tried yet as of 2023-05-22)

* sudo envoy -c config/tpm_proxy.yaml  [-l debug] - or
* TSS_TCSD_HOSTNAME=<host-ip> docker run -v $(pwd)/config/tpm_proxy.yaml:/etc/envoy/envoy.yaml -p 10000:10000 -e TSS_TCSD_HOSTNAME  envoy:latest -c /etc/envoy/envoy.yaml -l debug

### 4.2 My instructions as of 2023-05-22

* To run the test, we need a create a private key inside a TPM chip. Refer to the repository [yaobinwen-mvs/optiga-tpm-cheatsheet](https://github.com/yaobinwen-mvs/optiga-tpm-cheatsheet), section "Setup on Debian/Ubuntu" on how to do it.
* Follow the steps in ["Server-client TLS Communication"](https://github.com/yaobinwen-mvs/optiga-tpm-cheatsheet#server-client-tls-communication-2) to create a private key in TPM.
* Create `~/envoy` with the following contents:

```
-rw-r--r--  1 ywen ywen    0 May 19 19:56 admin.access_log.jsonl
lrwxrwxrwx  1 ywen ywen   58 May 19 18:18 config -> /home/ywen/wc/yaobinwen-mvs/envoy-extn-pkm-provider/config/
-rw-rw-r--  1 ywen ywen 1487 May 19 20:01 default.access_log.jsonl
lrwxrwxrwx  1 ywen ywen   67 May 19 14:27 envoy -> /home/ywen/wc/yaobinwen-mvs/envoy-extn-pkm-provider/bazel-bin/envoy*
```

* Because `proxy.yaml` assumes `xDS` is in the current working directroy, we need to run `envoy` under `config`, so follow the steps below:
  - `cd config`.
  - `sudo ~/envoy/envoy -l debug -c proxy.yaml` (`-l debug` prints debug messages for more debugging hints)

## 5. Access

* curl http://localhost:10000/
* the envoyproxy instance listening on port 10000 further connects to the target server
  initiating TLS connection using the certificate specified in the config file and uses TPM
  for private key operations.

## 6. TODOs

### 6.1 Figure out how to compile pkm_provider_config.proto via Bazel

The original repository used the built-in `google.protobuf.Struct` to pass PKM provider's configuration. I don't know how it was doable in Envoy 1.10.0 because I didn't seem to find the `.proto` file that defines `PrivateKeyProvider`. But this was doable since `v1.12.0` and at that time, `PrivateKeyProvider` was defined as follows in `api/v2/auth/cert.proto` (note this is API v2):

```protobuf
// BoringSSL private key method configuration. The private key methods are used for external
// (potentially asynchronous) signing and decryption operations. Some use cases for private key
// methods would be TPM support and TLS acceleration.
message PrivateKeyProvider {
  // Private key method provider name. The name must match a
  // supported private key method provider type.
  string provider_name = 1 [(validate.rules).string = {min_bytes: 1}];

  // Private key method provider specific configuration.
  oneof config_type {
    google.protobuf.Struct config = 2;

    google.protobuf.Any typed_config = 3;
  }
}
```

However, since we don't want to use API v2, we prefer not to use this definition. For API v3, the `extensions/transport_sockets/tls/v3/common.proto` (which is in `v.1.26.1`) removes `google.protobuf.Struct config`, so we need to use `google.protobuf.Any`. In order to use `google.protobuf.Any`, we need to define our own type of configuration (hence it's named as `typed_config`), so I added [`pkm_provider_config.proto`](./pkm_provider_config.proto).

This is why I needed to compile a `.proto` file. However, I haven't learned how to automatically compile a `.proto` file in Bazel, so I compiled it using `protoc` manually. Then Bazel complained that I was not using the right version of `protoc`.

I managed to figure out that Bazel was expecting [protoc 3.21.12](https://github.com/protocolbuffers/protobuf/releases/tag/v3.21.12), by using the `#pragma message` according to [this answer](https://stackoverflow.com/a/10227059):

```
pkm_provider_config.pb.h:17:49: note: '#pragma message: PROTOBUF_VERSION=3021012'
   17 | #pragma message(VAR_NAME_VALUE(PROTOBUF_VERSION))
      |                                                 ^
pkm_provider_config.pb.h:25:60: note: '#pragma message: PROTOBUF_MIN_PROTOC_VERSION=3021000'
```

This was another reason I wanted to learn how to compile `.proto` files in Bazel so to make sure the `.proto` files can be compiled using the same compiler version.

The pre-compiled binary of `protoc` can be found on their [Maven repository](https://repo1.maven.org/maven2/com/google/protobuf/protoc/3.21.12/). I downloaded this version and re-compiled `pkm_provider_config.proto` and the compilation could pass.

### 6.2 Bugs

See the sub-directory `debug`. The issues are sorted in the order I ran into them.

### 6.3 Try the instructions in Section 4.1 and Section 5

As of 2023-05-22, I haven't got a chance to try the instructions in Setction 4.1 (Original instructions) and Section 5 (Access). In Section 5, it uses `http` to access the service. This is kind of out of my expectation because I thought we should use HTTPS.
