# Envoy Extension for Private Key Method (PKM) Provider

Envoy extension for private key method provider.

If you have multiple applications (using different SSL libraries) that need to connect to a server and do TLS mutual auth using a client certificate whose private key is in TPM, then you can use envoyproxy using this extension and have all the apps connect via this proxy. 

## 1. Check out `envoy` submodule

There are two ways to check out the `envoy` submodule:
- 1). `clone` + `submodule`:
  - a). `git clone <this repository>`.
  - b). `cd envoy-extn-pkm-provider`.
  - c). `git submodule init`.
  - d). `git submodule update`.
- 2). `clone --recurse-submodules`:
  - a). `git clone --recurse-submodules <this repository>`.

## 2. Build

### 2.1 Requirement

* bazel: [Installing Bazel on Ubuntu](https://bazel.build/install/ubuntu)
* libtspi-dev (apt-get install -y libtspi-dev)
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
   virtualenv
```

### 2.2 Build

* ci/do_ci.sh build
* ci/docker_build.sh

## 3. Configure

* See tls_context in the sample config file at config/tpm_proxy.yaml

## 4. Run

* sudo envoy -c config/tpm_proxy.yaml  [-l debug] - or
* TSS_TCSD_HOSTNAME=<host-ip> docker run -v $(pwd)/config/tpm_proxy.yaml:/etc/envoy/envoy.yaml -p 10000:10000 -e TSS_TCSD_HOSTNAME  envoy:latest -c /etc/envoy/envoy.yaml -l debug

## 5. Access

* curl http://localhost:10000/
* the envoyproxy instance listening on port 10000 further connects to the target server
  initiating TLS connection using the certificate specified in the config file and uses TPM
  for private key operations.
