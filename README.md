# App definition format

Each `{app_name}.toml` in this directory defines one containerised CLI app. The file stem
is the app name: `ffmpeg.toml` is run with `climate run ffmpeg`.

A definition has three tables: `[app]`, `[image]`, and `[run]`.
Only `[app]` and `[image]` are required.

## `[app]`

| Key           | Type   | Default  | Description                              |
| ------------- | ------ | -------- | ---------------------------------------- |
| `name`        | string | required | App name; must match the file stem.      |
| `description` | string | `""`     | One-line description for `climate list`. |
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
| `cwd`         | bool           | `true`        | Bind-mount the current working directory at the same path; `false` mounts nothing.                               |
| `network`     | enum           | `"none"`      | Network access: `"full"` (host network), `"none"` (isolated, no connectivity), or `"localhost"` (loopback only). |

`entrypoint` accepts a single string or a list of strings: a string overrides the entrypoint
verbatim (run directly), while a list is encoded as a JSON array,
e.g. `entrypoint = ["/bin/sh", "-c", "ffmpeg -version"]`.

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
