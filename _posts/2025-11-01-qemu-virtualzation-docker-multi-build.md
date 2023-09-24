---
title: "Inside Multi-Platform Docker Builds with QEMU"
author: cef
date: 2025-10-31
categories: [Technical Writing, Open Source]
tags: [QEMU, Docker, Go, Linux]
render_with_liquid: false
description: "A deep dive into how Docker uses QEMU and binfmt-misc to build and run multi-architecture container images, enabling cross-platform support for x86, ARM, and beyond."
---

One intriguing feature of containers and images is their multi-platform support. Docker uses Buildx, which is based on BuildKit, to enable multi-platform builds.

The old way (build for the current host's platform, if you're on an ARM CPU, you build an ARM image that won't run on x86, and vice versa):

```sh
docker build .
```

The new way: without a single care, build a multi-platform image that supports both x86 and ARM (and others):

```sh
docker buildx build --platform linux/amd64,linux/arm64 .
```

How is this sorcery possible? Let's take a look.

## But First, What Are Containers?

Containers, under the hood, are simply processes isolated thanks to [Linux's namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html). The executables and files of these processes, packaged in layers, are compiled for a specific architecture.

Put differently, a container is a bundled runtime. This is what the [OCI runtime bundle](https://specs.opencontainers.org/runtime-spec/runtime/?v=v1.0.2) defines:

```sh
coolcontainer/
├── config.json
└── rootfs/
    ├── bin/
    ├── lib/
    └── ...
```

An OCI runtime bundle (used to start a container) is obtained from an OCI image (Docker images are OCI-compliant). 

```sh
apt install -y umoci skopeo runc

# pull an OCI image from Docker Hub into a local dir
skopeo copy docker://alpine:latest oci:alpine:latest

# contents of the image
ls alpine/
# blobs  index.json  oci-layout

# unpack the image into a runtime bundle
umoci unpack --image alpine:latest alpine-runtime-bundle
ls alpine-runtime-bundle/
# config.json
# rootfs
# sha256_24bb3511a0db7b5114a4aee033c65a8a4148f39b7b80a398e548546db967a36f.mtree
# umoci.json

# run the container
runc run -b alpine-runtime-bundle mycoco
    # inside container
    $ echo yo > /home/greeting
    $ exit
  
# host
cat alpine-runtime-bundle/rootfs/home/greeting
# yo
```

This spec defines what's needed to run a container. All OCI-compliant container solutions adhere to it (Docker, Podman, etc.). These files are then used to create a container process. The isolation is achieved through Linux namespaces. To the container, it feels like it's running on its own filesystem, network, PID space, and so on, but in reality, it's just a process, albeit a well-isolated one.

The reference implementation that takes an OCI runtime bundle and starts a container is `runc` (Docker uses it under the hood). It takes all the information and layers in the bundle and creates the container process. Mounts, environment variables, and all kinds of options that can be specified when running a container are handled by `runc`.

This means that the executables and binaries are destined for a specific OS and architecture:

```sh
file -L alpine-runtime-bundle/rootfs/bin/ls
alpine-runtime-bundle/rootfs/bin/ls: ELF 64-bit LSB executable, ARM aarch64
```

So the `ls` command inside the container layers is simply a regular executable built for ARM64.

You can't just run an image built for an x86 CPU on an ARM CPU (out of the box). That's where multi-platform images come into the picture.


## Multi-Platform Builds

An image that supports multiple architectures? Say what? 

![Alpine Multi OS/Arch](/assets/alpine-docker-img-multi-arch-os.png)

Or via the CLI:

```sh
skopeo inspect --raw docker://docker.io/library/ubuntu:latest | jq | grep "architecture"
        # "architecture": "amd64",
        # "architecture": "arm",
        # "architecture": "arm64",
        ...
```

```sh
# I'm running on ARM, let's bring in those x86/amd64 bad boys
skopeo copy \
  --override-arch amd64 \
  --override-os linux \
  docker://docker.io/library/nginx:latest \
  oci:nginx-amd64:latest

umoci unpack --image nginx-amd64:latest nginx-amd64-runtime-bundle

# check the type of executable file for 'ls'
file -L nginx-amd64-runtime-bundle/rootfs/bin/ls
# nginx-amd64-runtime-bundle/rootfs/bin/ls: ELF 64-bit LSB pie executable, x86-64
```

J'accuse! Intruder! An x86-64 binary on an ARM machine!

```sh
./alpine-runtime-bundle/rootfs/bin/ls
# works!
./nginx-amd64-runtime-bundle/rootfs/bin/ls
# bash: ./nginx-amd64-runtime-bundle/rootfs/bin/ls: cannot execute binary file: Exec format error
# OR
runc run -b nginx-amd64-runtime-bundle my-nginx-container
# exec /docker-entrypoint.sh: exec format error
```

And that's the heart of the problem when it comes to running containers across different platforms. What to do?


## QEMU and binfmt-misc to the Rescue

QEMU (Quick EMUlator) is quite the remarkable piece of software. It's both an emulator and a virtualizer, and it also provides user-level emulation.
Ehhh, what? Well, that's what you get when you look up QEMU. Let's put it in simpler terms:

* Emulator: It emulates hardware. It simulates entire systems (CPU, memory, disk, network, etc.) in software, meaning it exposes an interface to a guest program similar to actual hardware. Think about it: for an OS, all it sees is a bunch of CPU machine code that interacts with hardware and registers. If those registers and hardware behaviors are simulated in software, the OS is none the wiser and that's exactly what QEMU does. You can simulate different CPUs (ARM, x86, RISC-V, etc.), run machine code instructions, and update state (registers, flags, program counter, etc.) as if you were running on real hardware, it's just slower. By emulating CPUs in software, QEMU can run an OS built for the same or a different CPU architecture.

* Virtualizer: Some CPUs offer hardware-assisted virtualization, basically, the CPU can differentiate between a guest and a host OS. This is a lot faster than using an emulator, but since you're using the same CPU, you can only run a guest OS built for that CPU (for example, an x86 Linux guest on an x86 Linux host). This is supported in Linux through KVM. QEMU can make use of KVM, so when available, it's better to use it for faster guest execution.

* User-space emulation: This allows us to run a binary built for an architecture different from our machine's by translating machine code and system calls on the fly. For instance, `qemu-arm ./arm-binary` works on an `x86_64` CPU as if it were native. It's truly magical, QEMU decodes ARM instructions and translates them into x86-64 ones, roughly:

```
ARM code:          ADD R0, R1, R2
QEMU intermediate: tcg_gen_add_i32(result, R1, R2)
x86-64 host code:  mov eax,[R1]; add eax,[R2]; mov [R0],eax
```

So QEMU user-space emulation is the first piece of the cross-platform image puzzle.

The second piece is `binfmt-misc`. It stands for *Binary Format Miscellaneous* (quite the name). The basic idea is that your Linux kernel knows how to run executables built for its own architecture. If you're on an x86-64 CPU, your kernel can run x86-64 ELF files by default. It can't run executables built for other architectures (like ARM) or other file types (like Windows `.exe` files or scripts).

`binfmt-misc` is a kernel feature that allows us to specify an interpreter or program to handle certain files, based on their extension or on a magic byte sequence contained within the file. ARM Linux executables, for instance, have a distinguishable magic sequence:
`0x7F 'E' 'L' 'F' ...`
We can configure `binfmt-misc` to use `qemu-arm` whenever it encounters a file with that magic sequence. Similarly, we can configure it to use `/usr/bin/java` when encountering files with a `.jar` extension.

```sh
# ARM CPU
uname -m
# arm64

# use qemu-x86_64 when you encounter the x86_64 executable 'magic' bytes
echo ':x86_64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x3e::/usr/bin/qemu-x86_64:' | sudo tee /proc/sys/fs/binfmt_misc/register

# Now x86-64 executables just work!


# Java example (using `.jar` extension)
./HelloWorld.jar 
# bash: ./HelloWorld.jar: cannot execute binary file: Exec format error

cat <<'EOF' >/usr/local/bin/java-wrapper
#!/bin/sh
exec /usr/bin/java -jar "$@"
EOF
chmod +x /usr/local/bin/java-wrapper

# run java -jar when you encounter a file with extension '.jar'
echo ':Java:E::jar::/usr/local/bin/java-wrapper:' | sudo tee /proc/sys/fs/binfmt_misc/register

./HelloWorld.jar
# Hello World!
```

So, `binfmt-misc` lets the kernel specify a wrapper, interpreter, or command to run certain files based on their magic bytes or file extensions.

To recap: **QEMU** allows us to execute binaries from other architectures, and **binfmt-misc** is the mechanism that maps those binaries (based on their magic bytes) to the appropriate QEMU user-space command.



## Docker Build Using QEMU

In Docker's [documentation](https://docs.docker.com/build/building/multi-platform/#qemu) about multi-platform builds, they explain that Docker Desktop supports multi-platform images with QEMU out of the box. (Docker Desktop is essentially a Linux VM tailored to run Docker, so it already has this configured.)

For Docker engine in Linux, we need to run:

```sh
docker run --privileged --rm tonistiigi/binfmt --install all
```

This registers the `binfmt-misc` mappings (like we did above for Java and x86) but for all architectures.

The image [tonistiigi/binfmt](https://github.com/tonistiigi/binfmt/blob/2062d3e3b27656ff1b19d762994567155b6fbdb2/cmd/binfmt/config.go#L22) contains a Go binary that basically does what we demonstrated earlier, setting up mappings from ELF magic bytes to the appropriate QEMU binary for multiple architectures:

```go
var configs = map[string]config{
	"amd64": {
		binary: "qemu-x86_64",
		magic:  `\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00`,
		mask:   `\xff\xff\xff\xff\xff\xfe\xfe\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff`,
	},
	"arm64": {
		binary: "qemu-aarch64",
		magic:  `\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00`,
		mask:   `\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff`,
	},
	"arm": {
		binary: "qemu-arm",
		magic:  `\x7fELF\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00`,
		mask:   `\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff`,
	},
	"s390x": {
		binary: "qemu-s390x",
		magic:  `\x7fELF\x02\x02\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x16`,
		mask:   `\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff`,
	},
  // ...
}
```

In [`main.go`](https://github.com/tonistiigi/binfmt/blob/2062d3e3b27656ff1b19d762994567155b6fbdb2/cmd/binfmt/main.go#L98), we find this delightful snippet:

```go
file, err := os.OpenFile(register, os.O_WRONLY, 0) // 'register' is /proc/sys/fs/binfmt_misc/register
// ...
line := fmt.Sprintf(":%s:M:0:%s:%s:%s:%s", binaryBasename, cfg.magic, cfg.mask, binaryFullpath, flags)
_, err = file.Write([]byte(line))
```

Pretty sweet!




## The Final Piece of the Puzzle

Ok, we know how foreign binaries are run. But how does it all tie together? How are we actually building these multi-platform images?

The Docker docs have a nice example:

```sh
# Dockerfile
FROM alpine
# `uname -m` outputs the architecture and we redirect it to the /arch file
RUN uname -m > /arch
```

Let's build and test:

```sh
docker buildx build --platform=linux/amd64,linux/arm64 -t letsgo:1.0 .

docker run --rm -it --platform=linux/arm64 letsgo:1.0 cat /arch
# aarch64

docker run --rm -it --platform=linux/amd64 letsgo:1.0 cat /arch
# x86_64
```

It's beautiful!
So what happened exactly? By specifying `--platform=linux/amd64,linux/arm64`, we're asking Docker to build two images, one for each platform. The pulled base layer (`alpine`) is platform-specific, and the binaries within it are built for each architecture. Let's verify that:

```sh
docker run --rm -it --platform=linux/arm64 letsgo:1.0 sh
apk add file 
# check 'uname' file type
file -L /bin/uname
# /bin/uname: ELF 64-bit LSB pie executable, ARM aarch64

# same thing with `linux/amd64`:
# /bin/uname: ELF 64-bit LSB pie executable, x86-64, version 1
```

Nice!
The `Dockerfile` runs `uname -m > /arch`, and that's where the QEMU magic occurs. Under the hood, each binary in the image layers, compiled for its target architecture, is executed. Docker's `RUN` instruction spawns a new process on the host (isolated within namespaces, but still just a process). Depending on the file's magic bytes, the appropriate QEMU interpreter is invoked automatically via `binfmt_misc` and that RUN command works. If it was not for `binfmt_misc` and QEMU, we'd get a polite `Exec format error`. 


## Multi-platform without QEMU
QEMU is amazing and easy to use, but it can be very slow for heavy workloads. After all, we're emulating hardware (a different CPU architecture) in software, and there's an overhead since each instruction needs to be translated.
QEMU isn't the only way to build multi-platform images. Docker Buildx supports using multiple builder nodes (a cluster), and you can use nodes with different architectures to build images natively for their respective platforms. There's even a cloud offering built around this approach.

```sh
# add local
docker buildx create --append --name multiarch-builder unix:///var/run/docker.sock
# add remote different arch
docker buildx create --append --name multiarch-builder ssh://user@arm64-host
docker buildx use multiarch-builder
```

For compilers that support cross-compilation (compiling code on one platform, the host, to create an executable for a different platform ,the target), like Go, which does so natively by specifying `GOOS` and `GOARCH`, you can build directly for each target architecture without relying on emulation. For example, in a Dockerfile build stage you might run:

```sh
RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -o server .
```

Then, you can simply copy the resulting binary into the runtime stage. Since the Go compiler supports cross-compilation, there's no need to use QEMU here. Instead, we rely on the `TARGETOS` and `TARGETARCH` environment variables provided automatically by Docker Buildx.

## Container Internals Video
Here's a video where I explore containers internals. Let me know what you think!

<iframe
  class="embed-video"
  loading="lazy"
  src="https://www.youtube.com/embed/cXhr5e58fio"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>