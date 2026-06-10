<p align="center">
  <img src="assets/logo.png" alt="Harbor" width="150">
</p>

<h1 align="center">Harbor</h1>

<p align="center">A small, Docker-style container engine — written entirely in <a href="https://github.com/Juanshep1/vanta">Vanta</a>.</p>

---

Harbor builds images and runs them as isolated containers, the same way Docker
does. The twist: the whole engine is written in Vanta, a plain-English language
I built. `harbor.va` is the program; there's no Go, no C, no Rust under it. I
wanted to find out whether Vanta was a real enough language to write a real tool
in, and Harbor is the answer.

It runs the same on macOS, Windows, and Linux, because it doesn't depend on any
one operating system's container features.

```
$ ./harbor build harbor-examples/hello
Built image 'hello' (1 file(s))

$ ./harbor run hello
Running container harbor_481204 from image 'hello'
----------------------------------------
Hello from inside a Harbor container!
I am a Vanta program running in my own isolated sandbox.
My WHO environment variable is: Vanta
Proof I ran real code: 1+2+3+4+5 = 15
----------------------------------------
Container harbor_481204 exited with code 0
```

## How it relates to Docker

If you've used Docker, the mental model is identical:

- A **Harborfile** describes how to build an image (Docker has the Dockerfile).
- An **image** is a saved, reusable bundle of files plus a recipe.
- A **container** is a throwaway, isolated copy of an image that runs a command.
- Images live in `~/.harbor`, the way Docker keeps things in `/var/lib/docker`.

The one real difference is *how* containers are isolated. Docker uses Linux
kernel features (namespaces and cgroups), which is why Docker Desktop quietly
runs a Linux virtual machine on Mac and Windows. Harbor isolates at the folder
level instead: every container gets its own sandbox directory with its own copy
of the files, and the command runs inside it. That's portable and needs no VM,
at the cost of not being a hard security boundary. For learning how containers
actually work, it's the right trade.

## Requirements

Python 3.8+ (Harbor ships its own copy of the Vanta interpreter, `vanta.py`).

## Commands

```bash
./harbor build <dir>      # build an image from a folder with a Harborfile
./harbor images           # list images
./harbor inspect <image>  # show an image's recipe and metadata
./harbor run <image>      # create and run a container
./harbor ps               # list containers that have run
./harbor logs <id>        # show a container's captured output
./harbor push <image>     # push an image to the local registry
./harbor pull <image>     # pull an image from the local registry
./harbor rm <id>          # delete a container
./harbor rmi <image>      # delete an image
```

On Windows, use `harbor.bat` instead of `./harbor`.

## The Harborfile

A Harborfile is itself a Vanta program. You describe an image by calling a few
functions:

```
name("hello")
base("vanta")                    # "vanta" bundles the interpreter; "none" doesn't
include("app.va")                # files to copy into the image
setenv("WHO", "Vanta")           # environment variables for the container
expose(8080)                     # declare a port (recorded as metadata)
start("python3 vanta.py app.va") # the command the container runs
```

Because `base("vanta")` copies the Vanta interpreter into the image, a Vanta
container is fully self-contained — it carries its own runtime, just like a real
image carries its dependencies. Harbor can also run plain shell workloads; see
`harbor-examples/greeter`, which just runs `cat`.

## Try it

```bash
./harbor build harbor-examples/hello       # a container that runs a Vanta app
./harbor build harbor-examples/greeter     # a container that runs a shell command
./harbor images
./harbor run hello
./harbor inspect hello
./harbor ps

# push it to the local registry, delete it, pull it back
./harbor push hello
./harbor rmi hello
./harbor pull hello
```

## How it works

It's all in [`harbor.va`](harbor.va). The interesting parts:

- **build** runs the Harborfile through Vanta's `import`, which fills in a config
  map via the `name()` / `include()` / `start()` functions. It copies the listed
  files into `~/.harbor/images/<name>/`, optionally bundles the interpreter, and
  writes a `manifest.json`.
- **run** makes a fresh sandbox at `~/.harbor/containers/<id>/root`, copies the
  image's files in, and runs the start command *inside that folder* with
  `shell(command, directory, env)`. The output and exit code get saved so `ps`
  and `logs` can read them later.
- **push/pull** copy whole image folders to and from `~/.harbor/registry`.

If you want to see what the engine is built on, the language lives in its own
repo: [Vanta](https://github.com/Juanshep1/vanta).

## Limitations

Worth being honest about what this is and isn't:

- Folder-level isolation, not kernel-level. A container can't see other
  containers' files, but it isn't a security sandbox like a Linux namespace.
- No real networking. `expose(...)` records the port but nothing is mapped.
- No layered images or build caching — a build copies files straight in.
- No resource limits (CPU/memory).

## What's next

- Real port mapping for simple servers
- Layered images so shared files aren't copied twice
- A `stop`/`start` lifecycle for long-running containers
- Pushing to a remote registry over HTTP

## License

MIT. See [LICENSE](LICENSE).
