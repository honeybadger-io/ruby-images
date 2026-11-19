# ruby-images

Ruby base images for Honeybadger apps, published to `ghcr.io/honeybadger-io/ruby`.

We build these ourselves so we can ship a Ruby security release the day it comes out, without waiting for the official `ruby` images on Docker Hub. The Dockerfiles come from [docker-library/ruby](https://github.com/docker-library/ruby), the recipe behind the official images, with a few changes:

- Ruby is linked against **jemalloc** (`--with-jemalloc`), which cuts memory fragmentation and RSS for long-running Puma and Sidekiq processes.
- Ruby is built with **YJIT** on amd64 and arm64 (same as upstream).
- The Ruby version and checksum come from `versions.json` instead of being hard-coded.

Every change from upstream is marked with a `honeybadger:` comment in the Dockerfiles.

## Tags

| Tag | Base | Example |
|---|---|---|
| `<version>` | `buildpack-deps:trixie` | `ghcr.io/honeybadger-io/ruby:4.0.10` |
| `<version>-slim` | `debian:trixie-slim` | `ghcr.io/honeybadger-io/ruby:4.0.10-slim` |
| `<minor>` | `buildpack-deps:trixie` | `ghcr.io/honeybadger-io/ruby:4.0` |
| `<minor>-slim` | `debian:trixie-slim` | `ghcr.io/honeybadger-io/ruby:4.0-slim` |

The `<minor>` tags move to the newest patch release. Every tag is multi-arch (`linux/amd64` and `linux/arm64`) and is drop-in compatible with the matching official tag (`ruby:4.0.10`, `ruby:4.0.10-slim`).

All tags are rebuilt every Monday so they pick up Debian security fixes. A rebuild replaces the tag, so pin by digest (`ghcr.io/honeybadger-io/ruby:4.0.10-slim@sha256:...`) if you need a build to be byte-for-byte reproducible.

## Add a new Ruby version

1. Run `bin/add-version`, for example:

   ```sh
   bin/add-version 4.0.11
   ```

   This looks up the `.tar.xz` SHA-256 in [ruby-lang.org's release index](https://cache.ruby-lang.org/pub/ruby/index.txt) and writes it to `versions.json`. A patch release replaces the existing entry for its minor version (`4.0`). A new minor version (`4.1`) adds an entry, and both are built.
2. Check the checksum against the [ruby-lang.org releases page](https://www.ruby-lang.org/en/downloads/releases/).
3. Open a pull request. CI builds and tests every image on both CPU types but does not push.
4. Merge. CI pushes the new tags.
5. Bump the `FROM` lines in the app repos.

To stop building an old minor version, delete its entry from `versions.json`. Its tags stay in the registry but stop getting weekly rebuilds.

## Sync with upstream

`docker-library/ruby` changes now and then, for example to bump the pinned Rust toolchain used to build YJIT or to move to a new Debian release. To pull those changes in, compare our Dockerfiles with upstream's for the newest Ruby version and copy over anything that isn't a `honeybadger:` change:

```sh
git clone https://github.com/docker-library/ruby /tmp/docker-library-ruby
diff -u /tmp/docker-library-ruby/4.0/trixie/Dockerfile trixie/Dockerfile
diff -u /tmp/docker-library-ruby/4.0/slim-trixie/Dockerfile slim-trixie/Dockerfile
```

Both variants share one Dockerfile across all Ruby versions, so if we ever build a minor version whose upstream recipe differs, give it its own directory.

## Test an image locally

```sh
docker build -t ruby-test \
  --build-arg RUBY_VERSION=4.0.10 \
  --build-arg RUBY_DOWNLOAD_SHA256=1537f7a6003c436546ecab6b48b6e7725a9af2328f45a8af7f42f41c5df90ab8 \
  slim-trixie
bin/verify ruby-test 4.0.10
```

`bin/verify` checks the Ruby version, that jemalloc is linked and loaded, that YJIT turns on, and that OpenSSL, Psych, and zlib load. CI runs it on every image before pushing.

## How CI works

`.github/workflows/build.yml` runs on pull requests, pushes to `main`, a weekly schedule, and on demand.

1. **build**: one job per Ruby version, variant, and CPU type. Each job runs on a native runner (`ubuntu-latest` or `ubuntu-24.04-arm`), since compiling Ruby under QEMU is very slow. It builds the image without a layer cache (so base images and Debian packages are always fresh), runs `bin/verify`, and on `main` pushes the image by digest.
2. **publish** (`main` only): joins the amd64 and arm64 digests into one multi-arch manifest and tags it.

## License

The Dockerfiles are derived from [docker-library/ruby](https://github.com/docker-library/ruby), which is MIT-licensed. See `LICENSE.docker-library`.
