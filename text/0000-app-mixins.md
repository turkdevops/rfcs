# Meta
[meta]: #meta
- Name: Application Mixins
- Start Date: 2020-06-22
- Author(s): [Joe Kutner](https://github.com/jkutner/)
- RFC Pull Request: (leave blank)
- CNB Pull Request: (leave blank)
- CNB Issue: (leave blank)
- Supersedes: N/A

# Summary
[summary]: #summary

This is a proposal for allowing application developers (buildpack users) to specify mixins that will be dynamically included with their build and run images.

# Motivation
[motivation]: #motivation

Mixins already allow buildpack authors to create buildpacks that depend on an extended set of OS packages without affecting build time, but it's common for application code to depend on OS packages too. But allowing buildpacks to arbitrarily install OS packages during build would drastically increase build time, especially when CNB tooling is used to build or rebase many apps with similar package requirements.

For that reason, this proposal defines a mechanism to implement dynamic installation of mixins at build time while minimizing the performance impact. This is accomplished by allowing stack images to satisfy mixin requires statically and/or reject dynamic mixin installation early in the build process.

# What it is
[what-it-is]: #what-it-is

- *application developer* - a person who is using buildpacks to transform their application code into an OCI image
- *root buildpack* - a new type of buildpack that runs as the root user
- *stack buildpack* - a type of root buildpack that runs against the stack image(s) instead of an app. It is distinguished by a static list of mixins it can provide.

An application developer may specify a list of mixins in their application's `project.toml` file like this:

```toml
[build]
mixins = [ "<mixin name>" ]
```

When a command like `pack build` is run, the list of mixins will be processed before buildpacks are run. For each mixin name, the following will happen:

* If the mixin name is prefixed with `build:` (as per [RFC-0006](https://github.com/buildpacks/rfcs/blob/main/text/0006-stage-specific-mixins.md)) the mixin will be dynamically added to the build image.
* If the mixin name is prefixed with `run:` (as per [RFC-0006](https://github.com/buildpacks/rfcs/blob/main/text/0006-stage-specific-mixins.md)) the mixin will be dynamically added to the run image.
* If the mixin name does not have a prefix it will be dynamically added to both the build and run stack images.

# How it Works
[how-it-works]: #how-it-works

When a list of mixins are required by buildpacks via the build plan and the build phase starts:

1. The lifecycle will compare the list of mixins to those provided by the stack. If all mixin names are provided by the stack, no further action is required.
1. If any requested mixin is not provided by the stack, the lifecycle will run the detect phase for all stackpacks defined in the builder.
1. The lifecycle will compare the mixins added to the build plan to see if they match they required mixins.
1. If at least one stackpack passes detect and provides the required mixin(s), the lifecycle will execute the stackpack build phase for all passing stackpack(s). If no stackpacks pass detect, or no stackpacks provide the required mixin, the build will fail with a message that describes the mixins that could not be provided.
1. During the lifecycle's build phase, the stackpacks that passed detection will run against the build and run images accordingly (see details below). All stackpacks will be run before the user's buildpacks.

## Stack Buildpacks

 A stack buildpacks (a.k.a. stackpacks) is a special case of buildpack that has the following properties:

* Is run as the `root` user
* Configured with `privileged = true` in the `buildpack.toml`
* Can only create one layer
* Does not have access to the application source code
* Is run before all regular buildpacks
* Is run against both the build and run images
* Is distributed with the stack run and/or build image
* May not write to the `/layers` directory

The stackpack interface is identical to the buildpack interface (i.e. the same `bin/detect` and `bin/build` scripts are required). However, some of the context it is run in is different from regular buildpacks.

For each stackpack, the lifecycle will use [snapshotting](https://github.com/GoogleContainerTools/kaniko/blob/master/docs/designdoc.md#snapshotting-snapshotting) to capture changes made during the stackpack's build phase (excluding `/tmp`, `/cnb`, and `/layers`). Alternatively, a platform may store a new stack image to cache the changes. All of the captured changes will be included in a single layer produced as output from the stackpack. The `/layers` dir MAY NOT be used to create arbitrary layers.

A stack can provide stackpacks by including them in the `/cnb/stack/buildpacks` directory. By default, an implicit order will be created for all stackpacks included in a stack. The order can be overridden in the `builder.toml` with the following configuration:

```
[[stack.order]]
  [[stack.order.group]]
    id = "<stackpack id>"
```

A stackpack will only execute if it passes detection.

The stackpack's snapshot layer may be modified by writing a `launch.toml` file. The `[[processes]]` and `[[slices]]` tables may be used as normal, and a new `[[excludes]]` table will be introduced with the following schema:

```
[[excludes]]
paths = ["<sub-path glob>"]
cache = false
```

Where:

* `paths` = a list of paths to exclude from the layer
* `cache` = if true, the paths will be excluded from the launch image layer, but will be included in the cache layer.

## Rebasing an App

Before a launch image is rebased, the platform must re-run the any stackpacks that were used to build the launch image against the new run-image. Then, the rebase operation can be performed as normal, while including the stackpack layers as part of the stack. This will be made possible by including the stackpack in the run-image, but because the stackpack detect phase is not run, the operation does not need access to the application source.

## Example: Apt Buildpack

A buildpack that can install an arbitrary list of mixins would have a `buildpack.toml` like this:

 ```toml
[buildpack]
id = "example/apt"
privileged = true

[buildpack.mixins]
any = true

[[stacks]]
id = "io.buildpacks.stacks.bionic"
```

Its `bin/detect` would have the following contents:

 ```bash
#!/usr/bin/env bash

exit 0
```

Its `bin/build` would have the following contents:

 ```bash
#!/usr/bin/env bash

apt update -y

for package in $(cat $3 | yj -t | jq -r ".entries | .[] | .name"); do
  apt install $package
done

cat << EOF > launch.toml
[[excludes]]
paths = [ "/var/cache" ]
cache = true
EOF
```

## Example: CA Certificate Buildpack

Support for custom CA Certificates can be accomplished with two buildpacks: a stackpack that can install the cert, and a normal buildpack that can provide a cert in the build plan.

### CA Cert Installer Stackpack

A buildpack that installs custom CA Certificates would have a `buildpack.toml` that looks like this:

```toml
[buildpack]
id = "example/cacerts"
privileged = true

[[stacks]]
id = "io.buildpacks.stacks.bionic"
```

Its `bin/detect` would have the following contents:

 ```bash
#!/usr/bin/env bash

cat << EOF > $2
[[provides]]
name = "cacert"
EOF

exit 0
```

Its `bin/build` would would install each `cacert` in the build plan. It would have the following contents:

 ```bash
#!/usr/bin/env bash

# filter to the cert
for file in $(cat $3 | yj -t | jq -r ".entries | .[] | select(.name==\"cacert\") | .metadata | .file"); do
  cert="$(cat $3 | yj -t | jq -r ".entries | .[] | select(.name==\"cacert\") | .metadata | select(.file==\"$file\") | .content")"
  echo $cert > /usr/share/ca-certificates/$file
  chmod 644 /usr/share/ca-certificates/$file
done

update-ca-certificates
```

### CA Cert Provider Buildpack

The stackpack must be used with a buildpack that provides a certificate(s) for it to install. That buildpack would have the following `buildpack.toml`:

```toml
[buildpack]
id = "my/db-cert"

[[stacks]]
id = "io.buildpacks.stacks.bionic"
```

Its `bin/detect` would require a certificate with the following contents:

 ```bash
#!/usr/bin/env bash

cat << EOF > $2
[[requires]]
name = "cacert"

[requires.metadata]
file = "database.crt"
content = """
$(cat $CNB_BUILDPACK_DIR/database.crt)
"""

[[requires]]
name = "cacert"

[requires.metadata]
file = "server.crt"
content = """
$(cat $CNB_BUILDPACK_DIR/server.crt)
"""
EOF

exit 0
```

Its `bin/build` would do nothing, and `have the following contents:

 ```bash
#!/usr/bin/env bash

exit 0
```

# Drawbacks
[drawbacks]: #drawbacks

- Mixins are less flexible than a generic [root buildpack](https://github.com/buildpacks/rfcs/blob/app-image-ext-buildpacks/text/0000-app-image-extensions.md).

# Alternatives
[alternatives]: #alternatives

- [Add root buildpack as interface for app image extension RFC](https://github.com/buildpacks/rfcs/pull/77)
- [App Image Extensions (OS packages)](https://github.com/buildpacks/rfcs/pull/23)
- [Root Buildpacks](https://github.com/buildpacks/rfcs/blob/root-buildpacks/text/0000-root-buildpacks.md): Allow application developers to use root buildpacks in `project.toml`. This would have significant performance implications, and creates a "foot gun" in which end users could build images they are not able to maintain. For this reason, we are holding off on a generic root buildpack feature.

# Prior Art
[prior-art]: #prior-art

- [RFC-0006: Stage specific mixins](https://github.com/buildpacks/rfcs/blob/main/text/0006-stage-specific-mixins.md)
- The term "non-idempotent" is used in section 9.1.2 of [Hypertext Transfer Protocol -- HTTP/1.1 (RFC 2616)](https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html)

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

- Should the stackpack's detect have read-only access to the app?
    - Does a stackpack even need a detect phase?
    - This would likely be driven by a stackpack that does not provide mixins, but instead dynamically contributes to the build plan based on the contents of the app source code. I don't know if we have a use case for this, but I can imaging a buildpack that reads environment variables as input to some function.
- Should stackpacks be able to define per-stack mixins?
    - We could support a top-level `mixins` array, and allow refinements under `[[stacks]] mixins`. If we do this, we need to distinguish between provided and required mixins (at least under `[[stacks]]`).
    - If buildpacks can require mixins from `bin/detect`, the stackpack could use this mechanism to require per-stack mixins.
- Should we use regex to match mixins?

# Spec. Changes (OPTIONAL)
[spec-changes]: #spec-changes

## Stackpacks

Stackpacks are identical to other buildpacks, with the following exceptions:

1. The `<layers>` directory is NOT writable.
1. The working directory WILL NOT contain application source code during the build phase.
1. All changes made to the filesystem (with the exception of `/tmp`) during the execution of the stackpack's `bin/build` will be snapshotted and stored as a single layer.

## launch.toml (TOML)

```
[[excludes]]
paths = ["<sub-path glob>"]
cache = false
```

Where:

* `paths` = a list of paths to exclude from the layer
* `cache` = if true, the paths will be excluded from the launch image layer, but will be included in the cache layer.

## buildpack.toml  (TOML)

 This proposal adds new keys to the `[buildpack]` table in `buildpack.toml`, and a new `[[mixins]]` array of tables:

 ```
[buildpack]
privileged = <boolean (default=false)>
non-idempotent = <boolean (default=false)>

[buildpack.mixins]
any = "<boolean (default=false)>"
names = [ "<mixin name>" ]
 ```

* `privileged` - when set to `true`, the lifecycle will run this buildpack as the `root` user.
* `non-idempotent` - when set to `true`, indicates that the buildpack is not idempotent. The lifecycle will provide a clean filesystem from the stack image(s) before each run (i.e. no cache).

Under the `[buildpack.mixins]` table:

* `any` - a boolean that, when true, indicates that the buildpack can provide all mixins
* `names` - a list of names that match mixins provided by this buildpack