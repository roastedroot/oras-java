# ORAS Java SDK (Java 11 Fork)

[![Build](https://github.com/roastedroot/oras-java/actions/workflows/build.yml/badge.svg)](https://github.com/roastedroot/oras-java/actions/workflows/build.yml)

> **Note:** This is a fork of [oras-project/oras-java](https://github.com/oras-project/oras-java)
> maintained with **Java 11** compatibility. The upstream project targets Java 17+.
>
> The Java packages remain `land.oras.*` — you can switch between this fork and
> upstream by changing only the Maven/Gradle coordinates.

OCI Registry as Storage enables libraries to push OCI Artifacts to [OCI Conformant](https://github.com/opencontainers/oci-conformance) registries. This is a Java SDK for Java developers to empower them to do this in their applications.

## Consuming SDK

### Maven

```xml
<dependency>
    <groupId>io.roastedroot</groupId>
    <artifactId>oras-java</artifactId>
    <version>VERSION_HERE</version>
</dependency>
```

### Gradle

```groovy
implementation 'io.roastedroot:oras-java:VERSION_HERE'
```

## Examples

### Authentication

Using existing credentials from `~/.docker/config.json` or `$XDG_RUNTIME_DIR/containers/auth.json`:

```java
Registry registry = Registry.builder().defaults().build();
```

Using a username and password:

```java
Registry registry = Registry.builder().defaults("username", "password").build();
```

### Push a single file

```java
Registry registry = Registry.builder().insecure().build();
LocalPath artifact = LocalPath.of(Path.of("my-file.txt"));
Manifest manifest = registry.pushArtifact(ContainerRef.parse("localhost:5000/hello:v1"), artifact);
```

### Push multiple files with a custom artifact type

Push several files at once with a custom artifact type, per-file media types, and manifest-level annotations:

```java
Registry registry = Registry.builder().insecure().build();

Annotations annotations = Annotations.ofManifest(Map.of("build-tool", "maven"))
        .withFileAnnotations("pom.xml", Map.of("format", "xml"));

Manifest manifest = registry.pushArtifact(
        ContainerRef.parse("localhost:5000/my-app:v1"),
        ArtifactType.from("application/vnd.maven+type"),
        annotations,
        LocalPath.of(Path.of("pom.xml"), "application/xml"),
        LocalPath.of(Path.of("target/app.jar"), "application/java-archive"));
```

The resulting manifest will contain one layer per file, each annotated with its filename via `org.opencontainers.image.title`.

### Push a directory

Directories are automatically compressed as a `tar+gzip` archive and tagged with
`org.opencontainers.image.title` set to the directory name. The `io.deis.oras.content.unpack`
annotation is set to `true` so the SDK automatically extracts the archive on pull.

```java
Registry registry = Registry.builder().insecure().build();
Manifest manifest = registry.pushArtifact(
        ContainerRef.parse("localhost:5000/my-configs:v1"),
        LocalPath.of(Path.of("config-dir")));
```

To push a directory as a plain zip instead:

```java
Manifest manifest = registry.pushArtifact(
        ContainerRef.parse("localhost:5000/my-configs:v1"),
        LocalPath.of(Path.of("config-dir"), "application/zip"));
```

### Pull an artifact

Files are automatically written using the `org.opencontainers.image.title` layer annotation as the filename.
The third argument controls whether existing files are overwritten:

```java
Registry registry = Registry.builder().insecure().build();
registry.pullArtifact(ContainerRef.parse("localhost:5000/hello:v1"), Path.of("output-dir"), true);
```

### Attach an artifact (referrers)

Attach a signature or attestation to an already-pushed artifact. The attached manifest references the
original via its `subject` field and is discoverable through the [Referrers API](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#listing-referrers):

```java
Registry registry = Registry.builder().insecure().build();
ContainerRef ref = ContainerRef.parse("localhost:5000/my-app:v1");

// Push the main artifact first
Manifest manifest = registry.pushArtifact(ref,
        ArtifactType.from("application/vnd.maven+type"),
        LocalPath.of(Path.of("pom.xml"), "application/xml"));

// Attach a signature as a referrer
Manifest signatureManifest = registry.attachArtifact(
        ref,
        ArtifactType.from("application/vnd.example.signature"),
        LocalPath.of(Path.of("pom.xml.asc")));

// List all referrers for the artifact
Referrers referrers = registry.getReferrers(
        ref.withDigest(manifest.getDescriptor().getDigest()), null);
```

### Assemble a manifest from individual blobs

For fine-grained control, push blobs and configs individually before assembling and pushing the manifest:

```java
Registry registry = Registry.builder().insecure().build();
ContainerRef ref = ContainerRef.parse("localhost:5000/my-app:v1");

// Push individual layers
Layer layer1 = registry.pushBlob(ref, Files.readAllBytes(Path.of("schema.json")))
        .withAnnotations(Map.of(Const.ANNOTATION_TITLE, "schema.json"));
Layer layer2 = registry.pushBlob(ref, Files.readAllBytes(Path.of("data.csv")))
        .withAnnotations(Map.of(Const.ANNOTATION_TITLE, "data.csv"));

// Push a custom config
Config config = registry.pushConfig(ref, Config.empty().withMediaType("application/vnd.example.config+json"));

// Assemble and push the manifest
Manifest manifest = Manifest.empty()
        .withConfig(config)
        .withLayers(List.of(layer1, layer2));
registry.pushManifest(ref, manifest);
```

### Copy between registries

Copy a tagged artifact — including all its blobs — from one registry to another:

```java
Registry source = Registry.builder().defaults("user", "pass").insecure().build();
Registry target = Registry.builder().defaults("user", "pass").build();

ContainerRef from = ContainerRef.parse("localhost:5000/my-app:v1");
ContainerRef to   = ContainerRef.parse("registry.example.com/my-app:v1");

CopyUtils.copy(source, from, target, to, CopyUtils.CopyOptions.shallow());
```

### OCI Layout

OCI Layout lets you work with artifacts stored on disk in the [OCI Image Layout](https://github.com/opencontainers/image-spec/blob/main/image-layout.md) format.

**Push to an OCI Layout directory:**

```java
LayoutRef ref = LayoutRef.parse("/tmp/my-layout:latest");
OCILayout ociLayout = OCILayout.Builder.builder().defaults(Path.of("/tmp/my-layout")).build();

Manifest manifest = ociLayout.pushArtifact(
        ref,
        ArtifactType.from("application/vnd.example.type"),
        Annotations.empty(),
        LocalPath.of(Path.of("my-file.txt"), "text/plain"));
```

**Pull from an OCI Layout directory:**

```java
LayoutRef ref = LayoutRef.parse("/tmp/my-layout:latest");
OCILayout ociLayout = OCILayout.Builder.builder().defaults(Path.of("/tmp/my-layout")).build();
ociLayout.pullArtifact(ref, Path.of("output-dir"), false);
```

**Tar-backed OCI Layout** (single-file, portable archive):

```java
LayoutRef ref = LayoutRef.parse("/tmp/my-layout.tar:latest");
OCILayout ociLayout = OCILayout.Builder.builder().defaults(Path.of("/tmp/my-layout.tar")).build();

ociLayout.pushArtifact(ref, ArtifactType.from("application/vnd.example.type"),
        Annotations.empty(), LocalPath.of(Path.of("my-file.txt"), "text/plain"));

// Pull from the same tar
ociLayout.pullArtifact(ref, Path.of("output-dir"), false);
```

**Copy from OCI Layout to a registry:**

```java
LayoutRef layoutRef = LayoutRef.parse("/tmp/my-layout:latest");
OCILayout ociLayout = OCILayout.Builder.builder().defaults(Path.of("/tmp/my-layout")).build();

Registry registry = Registry.builder().defaults("user", "pass").build();
ContainerRef target = ContainerRef.parse("registry.example.com/my-app:v1");

CopyUtils.copy(ociLayout, layoutRef, registry, target, CopyUtils.CopyOptions.shallow());
```

## Registries configuration

Since version `0.7.0` the ORAS Java SDK supports the `registries.conf` format
(see the [containers/image documentation](https://github.com/containers/image/blob/main/docs/containers-registries.conf.5.md)).

The SDK reads configuration from the following locations, in order (later entries override earlier ones):

1. `/etc/containers/registries.conf`
2. `/etc/containers/registries.conf.d/*.conf` (alphabetical)
3. `$HOME/.config/containers/registries.conf`
4. `$HOME/.config/containers/registries.conf.d/*.conf` (alphabetical)

### Supported features

```toml
# Short-name resolution mode (enforcing is the default)
short-name-mode = "enforcing"
unqualified-search-registries = ["docker.io"]

# Rewrite a location via a prefix
[[registry]]
prefix = "docker.io/bitnami"
location = "docker.io/bitnamilegacy"

# Block a registry
[[registry]]
prefix = "gcr.io"
blocked = true

# Mark a registry as insecure
[[registry]]
location = "localhost:5000"
insecure = true

# Mirrors — tried in order before falling back to the upstream registry
[[registry]]
prefix = "registry.example.com"
location = "registry.example.com"
mirror-by-digest-only = false   # set to true to restrict all mirrors to digest-only pulls

  [[registry.mirror]]
  location = "mirror1.example.com"
  insecure = false
  pull-from-mirror = "all"       # "all" (default) | "tag-only" | "digest-only"

  [[registry.mirror]]
  location = "mirror2.example.com"
  insecure = true
  pull-from-mirror = "digest-only"
```

### Resolution and evaluation order

Reference resolution always happens **before** any security decision, and every
security decision is evaluated against the **effective (resolved) reference** — not
the reference originally supplied. For each operation the SDK:

1. **Resolves** the reference: short-name / unqualified-search expansion (e.g.
   `nginx` -> `docker.io/library/nginx`), `prefix` -> `location` rewrites, and mirror
   selection.
2. **Evaluates** `blocked`, `insecure` (HTTP vs HTTPS) and the trust policy against
   that resolved reference.
3. **Connects** and transfers bytes.

This ordering is intentional: block-list and plaintext decisions must bind to the
host the request actually reaches. Evaluating them on the pre-rewrite reference
would let a mirror or alias redirect traffic to a blocked or plaintext host while
the check passed on the original name. `registries.conf` and `policy.json` are
trusted operator-controlled configuration; a threat model in which an attacker can
edit those files is out of scope (that host is already compromised).

Two consequences worth noting:

- **Policy scope is repository-level.** The `policy.json` format scopes rules to
  `registry[/namespace/repository]` only — tags and digests are stripped before
  matching. A policy rule *cannot* protect (or single out) a specific tag or digest.
- **The trust policy is a pull-time gate.** Manifest/index *pulls* are verified
  against the policy; *deletes* are only checked against `blocked`/`insecure`, never
  content-verified. Protecting a specific digest from deletion must be enforced by
  registry-side RBAC / tag immutability, not by the client trust policy.

## Trust policy

The ORAS Java SDK can enforce a containers trust policy when pulling, using the
[`policy.json`](https://man.archlinux.org/man/containers-policy.json.5.en) format used by Podman,
Skopeo and Buildah.

The policy is loaded from the following locations, in order (the first that exists wins):

1. `$HOME/.config/containers/policy.json`
2. `/etc/containers/policy.json`

If no policy file is found, an **accept-all** policy is used. You can also set it explicitly:

```java
// Load from a specific file
Registry registry = Registry.builder()
        .defaults()
        .withPolicy(Path.of("/etc/containers/policy.json"))
        .build();

// Or build one programmatically
Registry registry = Registry.builder()
        .defaults()
        .withPolicy(ContainersPolicy.rejectAll())
        .build();
```

When a policy is set, every manifest/index pull is evaluated against it and rejected
(`OrasException`) if it does not pass.

### Supported requirement types

| Type                     | Supported | Behaviour                                                          |
|--------------------------|-----------|--------------------------------------------------------------------|
| `insecureAcceptAnything` | Yes       | Accept the image without any verification (trust all).             |
| `reject`                 | Yes       | Reject the image unconditionally.                                  |
| `sigstoreSigned`         | Yes       | Accept only images with a valid keyed Sigstore (cosign) signature. |
| `signedBy` (GPG)         | No        | Not implemented. Legacy                                            |

### `sigstoreSigned`

Only **keyed** verification is supported for now. If `keyPath` or `keyData` is present it contains a single
Sigstore public key (the `cosign.pub` produced by `cosign generate-key-pair`), and only signatures
made by that key are accepted:

- `keyPath` — path to a PEM public key file.
- `keyData` — the same key, base64-encoded inline.

Multiple keys (`keyPaths`/`keyDatas`) and keyless (Fulcio/Rekor) verification are **not** supported.
Signatures are discovered through the OCI [referrers API](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#listing-referrers)
(the Sigstore bundle, `application/vnd.dev.sigstore.bundle.v0.3+json`, attached to the image); no
local signature store is consulted. Verification binds the signature to the pulled image by its
digest. The `signedIdentity` field is **not supported** and is ignored if present, because the
cosign bundle payload carries only the image digest and no claimed Docker reference to match against.

```json
{
    "default": [{"type": "insecureAcceptAnything"}],
    "transports": {
        "docker": {
            "example.com/my-image": [
                {"type": "sigstoreSigned", "keyPath": "/home/me/my-key.pub"}
            ]
        }
    }
}
```

## Attribution

This project is a fork of [oras-project/oras-java](https://github.com/oras-project/oras-java),
originally created by [Valentin Delaye](https://github.com/jonesbusy) and the ORAS Authors.
We are grateful to the original contributors for their work on the ORAS Java SDK.

## License

This code is licensed under the Apache 2.0 [LICENSE](LICENSE).
