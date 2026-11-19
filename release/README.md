# Releasing azuracast 0.23.4-2

0.23.4-2 is built exactly like 0.23.4-1 below, from the same deployed 0.23.4
digest, with PHP's own files copied in from
`repo.cylo.net/php-artifacts:docker-library-php-8.5.13-fpm-trixie` instead of
8.5.12. PHP 8.5.13 carries the php-src security fixes of 19 November 2026.

- The artifact recipe is docker-library/php 8f7cce49 `8.5/trixie/fpm`, which
  differs from the 8.5.12 recipe only in the PHP version, URL and sha256
  lines; `FROM` stays pinned to the same debian:trixie-slim digest.
- It was built directly on the dedicated builder (builder.grant) from the
  pushed recipe commit, then turned into the artifact with
  `make-artifact.sh`, gated with `verify-artifact.sh` and pushed.
  `artifact.env` records the provenance.
- `release/build-on-builder.sh` now targets builder.grant (port 12246) and
  builds with `--builder default`, because the builder's selected buildx
  builder does not load images into the local store.
- PHP 8.5.13 turns `session.upload_progress.enabled` off by default. The
  image's own PHP and FPM configuration is unchanged; the in-place
  `zz-php-2026-11-19-mitigation.ini` added to running 0.23.4-1 containers on
  19 November is not part of the image.

Build, gate and push from a clean checkout of branch `php-8.5.13`:

    CYLO_BUILD_TMP=<scratch dir> release/build-on-builder.sh release

# Releasing azuracast 0.23.4-1 (previous release)

The image is the deployed 0.23.4 digest with PHP's own files copied in from
`repo.cylo.net/php-artifacts:docker-library-php-8.5.12-fpm-trixie` (see `Dockerfile`).

## 1. The patched PHP

On 21 October 2026 Docker Hub's `php:8.5.12-fpm-trixie` tag was an empty image index
(no linux/amd64 image), so PHP 8.5.12 was built here instead, from the
unmodified docker-library/php `8.5/trixie/fpm` recipe with `FROM` pinned by digest:

1. `release/php-artifact/node-build.sh` built `release/php-artifact/docker-library/`
   on a low-load Cylo node (legacy builder, `--cpu-shares 256`), saved the
   image as a gzip'd docker archive and recorded its sha256.
2. The archive was copied node -> builder with `scp -3` and its sha256 was
   checked again on the builder before `docker load`.
3. `release/php-artifact/make-artifact.sh` turned the loaded image into a
   FROM-scratch artifact (the whole /usr/local tree plus /php-artifact/
   README, SHA256SUMS, SYMLINKS, Dockerfile, runtime packages, php -v/-m).
4. `release/php-artifact/verify-artifact.sh` gated it (PHP version, libraries,
   php-fpm -t, multipart upload, artifact contents identical to the image,
   tag absent) and it was pushed, then pinned by digest in `Dockerfile`.

`release/php-artifact/artifact.env` records the provenance (recipe commit,
base digest, PHP tarball sha256 and signing key, node, image ID, archive
sha256).

## 2. Build, gate, push

From a clean checkout of branch `php-multipart-fix` on the Mac:

    git pull --ff-only origin php-multipart-fix
    CYLO_BUILD_TMP=<scratch dir> release/build-on-builder.sh release

`release` builds `repo.cylo.net/azuracast:0.23.4-1` on builder1 from the pushed
commit, runs `release/verify-image.sh` and pushes only if the gate passes,
then prints the pushed digest. The gate checks the deployed layers and image
config are unchanged apart from PHP_VERSION and the PHP files, the PHP files
match the pinned artifact, php and php-fpm report 8.5.12, no library is
missing, `php -m` and `php-fpm -m` equal the deployed image, `php-fpm -t`
and `nginx -t` pass, a PHP page served through the image's nginx and
php-fpm parses a multipart upload, PHP's built-in server parses one too, and
the tag is definitively absent from the registry (MANIFEST_UNKNOWN).
