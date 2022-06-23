---
Title: How to use container-suseconnect
slug: container-suseconnect
---

# What is container-suseconnect?

[`container-suseconnect`](https://github.com/SUSE/container-suseconnect) is a
Zypper plugin available in all Base Container Images that ship with Zypper. When
the plugin detects the host's SUSE Linux Enterprise Server, it automatically
registers the running container. This gives access the SUSE Linux
Enterprise repositories. This includes additional modules and previous package
versions that are not part of the free SLE_BCI repository.

## How to use container-suseconnect

If you are running a registered SLES system with Docker, `container-suseconnect`
automatically detects and uses the subscription, without requiring any action on
your part.

On openSUSE systems with Docker, you must copy the files `/etc/SUSEConnect` and
`/etc/zypp/credentials.d/SCCcredentials` from a registered SLES machine to your
local machine. Note that the `/etc/SUSEConnect` file is required only if you are
using RMT for managing your registration credentials.

## How to use container-suseconnect on non-SLE hosts or with Podman, Buildah or with nerdctl

To use `container-suseconnect` on non-SLE hosts or with Podman, Buildah or
with nerdctl, you need to have access to a registered SLES system. This can be a
physical machine, a virtual machine or the bci-base container into which you
install `SUSEConnect` and register the machine. Unless you are using RMT to
manage your subscription codes, you only have to copy
`/etc/zypp/credentials.d/SCCcredentials` to your development machine, otherwise
copy `/etc/SUSEConnect` as well. For example you can use the following one liner
to obtain `SCCcredentials` (it will be shown as the last two lines of the
output):

{{< tabs "obtain_scccredentials">}}
{{< tab "Docker">}}
```Shell
docker run --rm registry.suse.com/suse/sle15:latest bash -c \
    "zypper -n in SUSEConnect; SUSEConnect --regcode $YOUR_REG_CODE_HERE; \
     cat /etc/zypp/credentials.d/SCCcredentials"
```
{{< /tab >}}
{{< tab "Podman" >}}
```Shell
podman run --rm registry.suse.com/suse/sle15:latest bash -c \
    "zypper -n in SUSEConnect; SUSEConnect --regcode $YOUR_REG_CODE_HERE; \
     cat /etc/zypp/credentials.d/SCCcredentials"
```
{{< /tab >}}
{{< tab "nerdctl" >}}
```Shell
nerdctl run --rm registry.suse.com/suse/sle15:latest bash -c \
    "zypper -n in SUSEConnect; SUSEConnect --regcode $YOUR_REG_CODE_HERE; \
     cat /etc/zypp/credentials.d/SCCcredentials"
```
{{< /tab >}}
{{< /tabs >}}

If you are directly running a container based on a BCI, then you have to mount
`SCCcredentials` (and optionally `/etc/SUSEConnect` as well) in the correct
place. In the following example we put `SCCcredentials` into the current working
directory:

{{< tabs "use_scccredentials_as_secret">}}
{{< tab "Docker">}}
```Shell
docker run -v /path/to/SCCcredentials:/etc/zypp/credentials.d/SCCcredentials \
    -it --pull=always registry.suse.com/bci/bci-base:latest
```
{{< /tab >}}
{{< tab "Podman" >}}
```Shell
podman run -v /path/to/SCCcredentials:/etc/zypp/credentials.d/SCCcredentials \
    -it --pull=always registry.suse.com/bci/bci-base:latest
```
{{< /tab >}}
{{< tab "nerdctl" >}}
```Shell
nerdctl run -v /path/to/SCCcredentials:/etc/zypp/credentials.d/SCCcredentials \
    -it --pull=always registry.suse.com/bci/bci-base:latest
```
{{< /tab >}}
{{< /tabs >}}

It is highly recommended **not** to copy the `SCCcredentials` and `SUSEConnect`
files into your container image, as these files can easily end up in your final
image or one of its intermediate layers unless exceptional care is
taken. Instead, we recommend to rely on secrets, as these are only available to
a single layer and are not part of the built image. All you need is a copy of
`SCCcredentials` and (optionally `SUSEConnect`) somewhere on your file system
and to modify the `RUN` instructions which will invoke zypper as follows:

```Dockerfile
FROM registry.suse.com/bci/bci-base:latest

RUN --mount=type=secret,id=SUSEConnect \
    --mount=type=secret,id=SCCcredentials \
	zypper -n in fluxbox
```

Docker and buildah both support mounting secrets via the `--secret` flag as
follows:
{{< tabs "mount_secret">}}
{{< tab "Docker">}}
```Shell
docker build --secret=id=SCCcredentials,src=/path/to/SCCcredentials \
    --secret=id=SUSEConnect,src=/path/to/SUSEConnect .
```
{{< /tab >}}
{{< tab "Podman" >}}
```Shell
buildah bud --layers --secret=id=SCCcredentials,src=/path/to/SCCcredentials \
    --secret=id=SUSEConnect,src=/path/to/SUSEConnect .
```
{{< /tab >}}
{{< /tabs >}}


## Adding additional modules into the container or container Image

`container-suseconnect` allows you to automatically add SLE Modules into a
container or container image. Which modules are added is controlled via the
environment variable `ADDITIONAL_MODULES`. It should be set to a comma separated
list of the full module names. In a `Dockerfile`, this can be achieved using the
`ENV` directive as follows:

```Dockerfile
FROM registry.suse.com/bci/bci-base:latest
ENV ADDITIONAL_MODULES sle-module-desktop-applications,sle-module-development-tools

RUN --mount=type=secret,id=SCCcredentials zypper -n in fluxbox && zypper -n clean
```
