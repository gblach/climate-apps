# App definition format

Each `{app_name}.toml` in this directory defines one containerised CLI app. The file stem
is the app name: `ffmpeg.toml` is run with `climate run ffmpeg`.

A definition has four tables: `[app]`, `[image]`, `[run]`, and `[limits]`.
Only `[app]` and `[image]` are required.

## `[app]`

| Key           | Type   | Default  | Description                              |
| ------------- | ------ | -------- | ---------------------------------------- |
| `name`        | string | required | App name; must match the file stem.      |
| `description` | string | required | One-line description for `climate list`. |
| `license`     | string | required | SPDX license identifier of the main app. |

## `[image]`

The image and how to fetch it.

| Key         | Type   | Default  | Description                                                                  |
| ----------- | ------ | -------- | ---------------------------------------------------------------------------- |
| `reference` | string | required | Fully qualified image reference, e.g. `docker.io/linuxserver/ffmpeg:latest`. |
| `pull`      | bool   | `true`   | Whether the image may be pulled from a registry.                             |

`pull = true` (the default) pulls newer images when available; `pull = false` never pulls,
meaning the image is built locally or provided out of band.

## `[run]`

How the container is launched. Every key is optional; by default the current working
directory is mounted in so the tool behaves like a native one.

| Key           | Type           | Default       | Description                                                                                                      |
| ------------- | -------------- | ------------- | ---------------------------------------------------------------------------------------------------------------- |
| `entrypoint`  | string or list | image default | Override the image entrypoint.                                                                                   |
| `args`        | list of string | `[]`          | Default arguments, placed before user-supplied arguments.                                                        |
| `env`         | list of string | `[]`          | Environment entries: `"NAME"` passes a host variable through, `"NAME=VALUE"` sets it explicitly.                 |
| `mount-cwd`   | bool           | `true`        | Bind-mount the current working directory at the same path; `false` mounts nothing.                               |
| `mount`       | list of string | `[]`          | Extra host paths to share, on top of the working directory; see below.                                           |
| `network`     | enum           | `"none"`      | Network access: `"full"` (host network), `"none"` (isolated, no connectivity), or `"localhost"` (loopback only). |

`entrypoint` accepts a single string or a list of strings: a string overrides the entrypoint
verbatim (run directly), while a list is encoded as a JSON array,
e.g. `entrypoint = ["/bin/sh", "-c", "ffmpeg -version"]`.

`mount` shares further host paths with the container, mainly a tool's own config directory or file,
so its settings survive between runs. Each entry is written the way `docker` and `podman` spell
a share, as up to three `:` separated fields `<host path>:<path inside the container>:<ro|rw>`:

```toml
[run]
env = ["HOME"]
mount = [
    "~/.config/gh",                       # shared read-write under its own name
    "/etc/ssl/certs:ro",                  # shared read-only under its own name
    "~/.config/gh:/root/.config/gh",      # shared read-write, at another path inside
    "/etc/ssl/certs:/certs:ro",           # shared read-only, at another path inside
]
```

Leaving the middle field out keeps the path's own name inside the container, which needs the tool
to look for it there - passing the host's `HOME` through with `env = ["HOME"]` usually does it.
Name the path inside explicitly when the image runs the tool with a different home directory.

Shares are read-write unless the flag says `ro`. A leading `~` is the user's home directory; every
other path must be absolute. Files can be shared as well as directories.

A path that exists is shared as whatever it is. A missing one is created, so a tool can be given
its config before its first run - as a directory if the host path ends with `/`, as an empty file
otherwise. The `/` belongs to the host path, so it stays where it is when a path inside
the container follows:

```toml
mount = [
    "~/.config/gh/",                   # created as a directory
    "~/.gitconfig",                    # created as an empty file
    "~/.config/gh/:/root/.config/gh",  # created as a directory, shared at another path inside
]
```

A missing path shared read-only is an error instead, as sharing an empty one is never what
the definition meant.

## `[limits]`

How much of the machine one run may take, enforced by the kernel through cgroup v2. Every key is
optional, and a key left out is not limited at all, so a definition without this table runs with
the whole machine available, the way a natively installed tool does.

| Key           | Type   | Default   | Description                                                                          |
| ------------- | ------ | --------- | ------------------------------------------------------------------------------------ |
| `memory`      | string | unlimited | Memory ceiling, as a byte count with an optional binary unit.                        |
| `swap`        | string | unlimited | Swap allowed on top of `memory`; `"0"` keeps the app out of swap entirely.           |
| `memory-high` | string | unset     | Lower mark at which the app is throttled rather than killed; must be below `memory`. |
| `cpu`         | number | unlimited | CPU time in cores' worth per second, so `0.5` is half a core.                        |
| `cpu-shares`  | number | `1024`    | Share of a contended CPU, relative to other programs.                                |
| `pids`        | number | unlimited | Processes and threads the app may have at once.                                      |

Sizes are a byte count with an optional binary unit - `K`, `M`, `G` or `T`, where `M` means
1024 * 1024. `MB` and `MiB` spell the same unit; no unit at all means plain bytes.

```toml
[limits]
memory = "2G"
swap = "0"
cpu = 1.5
pids = 512
```

`memory` on its own is not the ceiling it looks like: an app that grows past it is pushed into
swap rather than killed. `swap` closes that - it says how much swap the app gets on top of
`memory`, and needs a `memory` limit to sit on top of.

`memory-high` throttles instead of killing, but reclaim needs somewhere to put the pages it frees,
so pairing it with `swap = "0"` leaves it little to do when the app's memory is mostly files it
wrote under `/tmp`; such an app is killed at `memory` anyway.

`cpu` caps total CPU time, not how many cores the app spreads over. `cpu-shares` only matters
while something else wants the CPU, and youki rescales the number, so `512` does not mean half.

Containers are limited through the systemd user session, which delegates only the `cpu`, `memory`
and `pids` controllers, so there are no keys for CPU pinning or disk I/O.

## Example

```toml
[app]
name = "ffmpeg"
description = "Audio/video transcoding"
license = "LGPL-2.1-or-later"

[image]
reference = "docker.io/linuxserver/ffmpeg:latest"

[run]
entrypoint = "ffmpeg"
```
