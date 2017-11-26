# Docker Image Resource NG

Concourse resource to pull, build and push [Docker](https://docker.io) images

## Source Configuration

* `repository`: *Required.* The name of the repository, e.g.
`concourse/docker-image-resource`.

  Note: When configuring a private registry, you must include the port
  (e.g. :443 or :5000) even though the docker CLI does not require it.

* `tag`: *Optional.* The tag to track. Defaults to `latest`.

* `username`: *Optional.* The username to authenticate with when pushing.

* `password`: *Optional.* The password to use when authenticating.

* `login`: *Optional.* An array of additional username, password and registry
  to log in, following this format:

  ```yaml
  login:
  - username: USERNAME
    password: PASSWORD
    registry: REGISTRY # (optional)
  ```

* `aws_access_key_id`: *Optional.* AWS access key to use for acquiring ECR
  credentials.

* `aws_secret_access_key`: *Optional.* AWS secret key to use for acquiring ECR
  credentials.

* `insecure_registries`: *Optional.* An array of CIDRs or `host:port` addresses
  to whitelist for insecure access (either `http` or unverified `https`).
  This option overrides any entries in `ca_certs` with the same address.

* `registry_mirror`: *Optional.* A URL pointing to a docker registry mirror service.

* `ca_certs`: *Optional.* An array of objects with the following format:

  ```yaml
  ca_certs:
  - domain: example.com:443
    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
  - domain: 10.244.6.2:443
    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
  ```

  Each entry specifies the x509 CA certificate for the trusted docker registry
  residing at the specified domain. This is used to validate the certificate of
  the docker registry when the registry's certificate is signed by a custom
  authority (or itself).

  The domain should match the first component of `repository`, including the
  port. If the registry specified in `repository` does not use a custom cert,
  adding `ca_certs` will break the check script. This option is overridden by
  entries in `insecure_registries` with the same address or a matching CIDR.

* `client_certs`: *Optional.* An array of objects with the following format:

  ```yaml
  client_certs:
  - domain: example.com:443
    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----
  - domain: 10.244.6.2:443
    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----
  ```

  Each entry specifies the x509 certificate and key to use for authenticating
  against the docker registry residing at the specified domain. The domain
  should match the first component of `repository`, including the port.

## Behavior

### `check`: Check for new images.

The current image digest is fetched from the registry for the given tag of the
repository.


### `in`: Fetch the image from the registry.

Pulls down the repository image by the requested digest.

The following files will be placed in the destination:

* `/image`: If `save` is `true`, the `docker save`d image will be provided
  here.
* `/repository`: The name of the repository that was fetched.
* `/tag`: The tag of the repository that was fetched.
* `/image-id`: The fetched image ID.
* `/digest`: The fetched image digest.
* `/rootfs.tar`: If `rootfs` is `true`, the contents of the image will be
  provided here.
* `/metadata.json`: Collects custom metadata. Contains the container  `env` variables and running `user`.
* `/docker_inspect.json`: Output of the `docker inspect` on `image_id`. Useful if collecting `LABEL` [metadata](https://docs.docker.com/engine/userguide/labels-custom-metadata/) from your image.

#### Parameters

* `save`: *Optional.* Place a `docker save`d image in the destination.
* `rootfs`: *Optional.* Place a `.tar` file of the image in the destination.
* `skip_download`: *Optional.* Skip `docker pull` of image. Artifacts based
  on the image will not be present.


### `out`: Push an image, or build and push a `Dockerfile`.

Push a Docker image to the source's repository and tag. The resulting
version is the image's digest.

#### Parameters

* `build`: *Optional.* The path of a directory containing a `Dockerfile` to
  build.

* `load`: *Optional.* The path of a directory containing an image that was
  fetched using this same resource type with `save: true`.

* `dockerfile`: *Optional.* The path of the `Dockerfile` in the directory if
  it's not at the root of the directory.

* `cache`: *Optional.* Default `true`. When the `build` parameter is set,
  first pull `image:tag` from the Docker registry (so as to use cached
  intermediate images when building). If no image can be pulled, the **build continues without a cache image**.

* `cache_tag`: *Optional.* Default `tag`. The specific tag to pull before
  building when `cache` parameter is set. Instead of pulling the same tag
  that's going to be built, this allows picking a different tag like
  `latest` or the previous version. This will cause the resource to fail
  if it is set to a tag that does not exist yet.

* `cache_repository`: *Optional.* If your cache image is not in the same registry as the one to build, set it here.
   Defaults to the registry of the image you build

* `load_base`: *Optional.* A path to a directory containing an image to `docker
  load` before running `docker build`. The directory must have `image`,
  `image-id`, `repository`, and `tag` present, i.e. the tree produced by `/in`.

* `load_file`: *Optional.* A path to a file to `docker load` and then push.
  Requires `load_repository`.

* `load_repository`: *Optional.* The repository of the image loaded from `load_file`.

* `load_tag`: *Optional.* Default `latest`. The tag of image loaded from `load_file`

* `import_file`: *Optional.* A path to a file to `docker import` and then push.

* `tag`: *Optional.* The value should be a path to a file containing the name
  of the tag.

* `tag_prefix`: *Optional.* If specified, the tag read from the file will be
  prepended with this string. This is useful for adding `v` in front of version
  numbers.

* `tag_as_latest`: *Optional.*  Default `false`. If true, the pushed image will
  be tagged as `latest` in addition to whatever other tag was specified.

* `build_args`: *Optional.*  A map of Docker build arguments.

  Example:

  ```yaml
  build_args:
    do_thing: true
    how_many_things: 2
    email: me@yopmail.com
  ```

* `build_args_file`: *Optional.* Path to a JSON file containing Docker build
  arguments.

  Example file contents:

    ```yaml
    { "email": "me@yopmail.com", "how_many_things": 1, "do_thing": false }
    ```            

## resource_type configuration

Since this is a custom resource, you need to register it, so your jobs can use it.
Place this somewheren in your `pipeline.yml`

```yaml
resource_types:
- name:                          docker-image-resource-ng
  type:                          docker-image
  privileged:                    true
  source:
    repository:                  eugenmayer/concourse-docker-image-resource
    tag:                         latest
```

## Example

``` yaml
jobs:
- name: build-rootfs
  plan:
  - get: alpine-base-image
    # if our base image, so the one we FROM for, has an update, rebuild us
    trigger: true
    params:
      skip_download: true
  - get: docker-java-image-scm
    # if the dockerfile changes, trigger a rebuild
    trigger: true
    # this will trigger the build and push 
  - put: docker-java-image
    params:
      build: docker-java-image-scm

resources:
- name: docker-java-image-scm
  type: git
  source: 
    uri: eugenmayer/docker-java-image

- name: alpine-base-image
  # thats the important one
  type: docker-image-resource-ng
  source:
    repository: alpine
    tag: edge
    
- name: docker-java-image
  # thats the important one
  type: docker-image-resource-ng
  source:
    repository: eugenmayer/java
    username: eugenmayer
    password: secretdockerhubpassword
    tag: jre8


resource_types:    
- name:                          docker-image-resource-ng
  type:                          docker-image
  privileged:                    true
  source:
    repository:                  eugenmayer/concourse-docker-image-resource
    tag:                         latest    
```
