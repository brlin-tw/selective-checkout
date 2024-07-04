# The `selective-checkout` override-pull scriptlet

Check out the upstream revision according to the versions published on the Snap Store.

<https://gitlab.com/brlin/selective-checkout>  
[![The GitLab CI pipeline status badge of the project's `main` branch](https://gitlab.com/brlin/selective-checkout/badges/main/pipeline.svg?ignore_skipped=true "Click here to check out the comprehensive status of the GitLab CI pipelines")](https://gitlab.com/brlin/selective-checkout/-/pipelines) [![GitHub Actions workflow status badge](https://github.com/brlin-tw/selective-checkout/actions/workflows/check-potential-problems.yml/badge.svg "GitHub Actions workflow status")](https://github.com/brlin-tw/selective-checkout/actions/workflows/check-potential-problems.yml) [![pre-commit enabled badge](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white "This project uses pre-commit to check potential problems")](https://pre-commit.com/) [![REUSE Specification compliance badge](https://api.reuse.software/badge/gitlab.com/brlin/selective-checkout "This project complies to the REUSE specification to decrease software licensing costs")](https://api.reuse.software/info/gitlab.com/brlin/selective-checkout)

## The problem

Currently, due to the limitation of the Snap Store build service and Launchpad, it's not possible to ensure that a snap will be built on certain tagged release as a build is only triggered when:

* Someone pushes to the associated repository(which development tip (a.k.a. `HEAD`) not necessarily a tagged release)
* A Launchpad import occurs if the repo is set to be importing periodically from an external repository

## The workaround

The `selective-checkout` scriptlet enhances the pull step to only build development snapshots snaps if the latest tagged release has already been promoted to the stable channel.  This ensures that there's always a revision of the stable release snap available in the edge channel for the publisher to promote to the stable channel.

### Case: Tagged release published to the stable channel

```output
selective-checkout: Last tagged release(1.8.1) is in the stable channel, building development snapshot
selective-checkout: Snap version determined to be "1.8.1-26-gd6ddb74".
```

### Case: Tagged release hasn't been published to the stable channel

```output
selective-checkout: Last tagged release(1.8.2) hasn't promoted to the stable channel(1.8.1) on the Snap Store, building tagged release instead.
Note: checking out 'v1.8.2'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at dcd1bd3 release version 1.8.2
selective-checkout: Snap version determined to be "1.8.2".
```

This scriptlet is inspired by the snapcrafters' adopted snaps workflow, documented in the following forum topic:

[popey's reply(#2) - The automatic build and pubish process of snaps owned by the Snapcrafters - doc - snapcraft.io](https://forum.snapcraft.io/t/the-automatic-build-and-pubish-process-of-snaps-owned-by-the-snapcrafters/7954/2)

Refer to the following forum topic for a possible solution proposal to this problem:

[Proposal: Allow overriding the source-tag property for an one-time build in the build infrastructure - snapcraft - snapcraft.io](https://forum.snapcraft.io/t/proposal-allow-overriding-the-source-tag-property-for-an-one-time-build-in-the-build-infrastructure/10303)

## Prerequisites

* Your snap recipe is using the `base` keyword (i.e. not the legacy snapcraft syntax)
* Your main component part must be using one of the following version control systems:
    + Git
    + Mercurial
    + Subversion
* The upstream release tags must be version sorted and be either dot-separated or underscore-separated, optionally prefixed with a fixed string, e.g. `1.2`, `1.2.3`, `v1.2.3`, or `FOOBAR_3_0_0`.
* The build host must have internet access (for querying snap status on the snap store)
* Your snap must not be private on the snap store (`unlisted` is fine)

## How to use

1. In the top-level snap metadata, DO NOT SET the `version` key, instead, set the `adopt-info` key with the main component part's name as the value.

    ```yaml
    #version:
    adopt-info: _part_name_
    ```

1. Merge the following part definition:

   ```yaml
   parts:
     # Check out the tagged release revision if it isn't promoted to the stable channel
     # https://forum.snapcraft.io/t/selective-checkout-check-out-the-tagged-release-revision-if-it-isnt-promoted-to-the-stable-channel/10617
     selective-checkout:
       source: https://github.com/brlin-tw/selective-checkout.git
       source-tag: v3.0.0
       plugin: dump
       build-packages:
         # Scriptlet dependencies
         - curl
         - jq
         - sed

         # Uncomment the VCS your main part is using
         #- git
         #- mercurial
         #- subversion
       stage:
         - scriptlets/selective-checkout
       prime:
         - -*
   ```

1. Merge the following keys to the main component part:

   ```yaml
   parts:
     _part_name_:
       after:
         - selective-checkout
       # Either the root source tree with a .git repository
       #source: .
       # or an external repository
       #source: https://example.com/project/.git
       # If you specify the `source-depth` property should be deep enough to include a tagged release
       #source-depth: 500
       override-pull: |
         snapcraftctl pull

         "${SNAPCRAFT_STAGE}"/scriptlets/selective-checkout

         # Do your additional regular stuff
   ```

## The logic

* The scriptlet will **checkout the latest tagged stable release** and build tagged release snap if the latest tagged release hasn't been promoted to the stable channel (check is based on version-sort)
* The scriptlet will **checkout the latest tagged beta/candidate releases** if they were found and it is version-sortwise more recent than the version shipped in the more stabler channels (for releasing the snap to beta/candidate channels)
* The scriptlet will **not touch the source tree** and build as-is if
    + A VCS repository is not found (will set the snap version to `unknown`)
    + Working tree is dirty(building test snaps)

## Supported command-line options

### `--append-packaging-revision`

Appending the packaging revision string after the snap version string, in the format: `_upstream_version_+pkg-_packaging_revision_`.  Useful for out-of-upstream-source-tree snap build.

### `--beta-tag-pattern=_pattern_`

Customize the extended regular expression pattern of the beta release tags.

**Default:** `-beta[[:digit:]]+$`  
**Example:** `b[[:digit:]]+$`

### `--debug` (DEPRECATED)

Enable debugging messages.  This command option is only supported for compatibility reasons, see the `SELECTIVE_CHECKOUT_DEBUG` environment variable for the recommended method.

### `--debug-tracing`

Enable execution tracing, just in case you've encountered a bug.

### `--dry-run`

Don't run `snapcraftctl` so that you can run it out of snap build(after changing the working directory to `SNAPCRAFT_PART_SRC` and exporting the `SNAPCRAFT_PROJECT_NAME` environment variable, of course!)

### `--force-snapshot`

Force building development snapshot regardless of the status of the snap release, for whatever reason.

### `--force-stable`

Force building stable release regardless of the status of the snap's releases, for whatever reason.

### `--packaging-dirty-marker-postfix=_postfix_`

The postfix string to be appended to the packaging revision as the dirty marker.

**Default:** `-d`

### `--packaging-revision-minimal-length=_string_length_`

The minimal length of the packaging revision string.

**Default:** 4

### `--release-candidate-tag-pattern=_pattern_`

Set the extended regular expression pattern to match the beta release tags.

**Default:** `-rc[[:digit:]]+$`

### `--release-tag-pattern=_pattern_`

Set the extended regular expression pattern of all to release tags to filter from all tags, useful when non-release tags are used.

**Default:** `.*[._].*` (any string that contain a dot(`.`) or an underscore(`_`)  
**Example:** `foobar_[[:digit:]]+(\.[[:digit:]]+){2}`

### `--release-tag-prefix=_prefix_`

Set the prefix for all release tags, the prefix will be stripped from the resulting snap's version string.

**Default:** `v`

### `--snap-postfix-seperator=_seperator_`

Set the separator for the postfixed string in the snap version string, the postfixed string will be stripped from the snap version string before comparing with the stripped upstream release version.

**Default:** `+`

### `--stable-tag-pattern=_pattern_`

Customize the extended regular expression pattern of the stable release tags.

**Default:** (unset) Derived from the exclusion of all the release candidates and beta release tags.

### `--upstream-dirty-marker-postfix=_postfix_`

The postfix string to be appended to the upstream version string as the dirty marker.

**Default:** `-dirty`

### `--upstream-revision-minimal-length=_string_length_`
The minimal length of the upstream revision string.

**Default:** `7`

## Supported environment variables

### `SELECTIVE_CHECKOUT_DEBUG`

Enable debugging messages, supported value: `true` / `false`

**Default:** `false`

## The implementation

[林博仁 Buo-ren, Lin / The selective-checkout override-pull scriptlet · GitLab](https://gitlab.com/brlin/selective-checkout)

## Snaps that are using this stage snap

* [mikf/gallery-dl: Command-line program to download image-galleries and -collections from several image hosting sites](https://github.com/mikf/gallery-dl) - [patch](https://github.com/mikf/gallery-dl/commit/81d4d49234b04ac72dbf2e531a5d3b0bafc1e47e)
* [Git : Code : Unofficial Snap Packaging for LÖVE : 林博仁(Buo-ren, Lin)](https://code.launchpad.net/~buo-ren-lin/love-snap) - [patch](https://git.launchpad.net/love-snap/commit/?id=a73c7cc092e77c7eb23656e4e5e21a9e37deae35)
* [ImageOptim/gifski: GIF encoder based on libimagequant (pngquant). Squeezes maximum possible quality from the awful GIF format.](https://github.com/ImageOptim/gifski) - [patch](https://github.com/ImageOptim/gifski/commit/f975d220b003af2faf8d5f93f100d303d08f93ab)
* [nicolargo/glances: Glances an Eye on your system. A top/htop alternative for GNU/Linux, BSD, Mac OS and Windows operating systems.](https://github.com/nicolargo/glances) - [patch](https://github.com/nicolargo/glances/commit/6fdf37ef8973fb7afbc90fafdfeb56cd1ff114b8)

## TODO

Patches welcome!

* Support other SCMs(Bazaar, etc.)
* Don't require user exporting `SNAPCRAFT_PROJECT_NAME` before running `--dry-run`

## Reference

To be addressed.

## Licensing

Unless otherwise noted(individual file's header/[REUSE DEP5](.reuse/dep5)), this product is licensed under [the WTFPL license](http://www.wtfpl.net/), or any of its recent versions you would prefer.

This work complies to [the REUSE Specification](https://reuse.software/spec/), refer the [REUSE - Make licensing easy for everyone](https://reuse.software/) website for info regarding the licensing of this product.
