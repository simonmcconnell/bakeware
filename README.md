# Bakeware

[![CircleCI](https://circleci.com/gh/bake-bake-bake/bakeware.svg?style=svg)](https://circleci.com/gh/bake-bake-bake/bakeware)
[![Hex version](https://img.shields.io/hexpm/v/bakeware.svg "Hex version")](https://hex.pm/packages/bakeware)
[![Hex Docs](https://img.shields.io/badge/hex-docs-lightgreen.svg)](https://hexdocs.pm/bakeware/)
[![Total Download](https://img.shields.io/hexpm/dt/bakeware.svg)](https://hex.pm/packages/bakeware)
[![License](https://img.shields.io/hexpm/l/bakeware.svg)](https://hex.pm/packages/bakeware)
[![Last updated](https://img.shields.io/github/last-commit/bake-bake-bake/bakeware.svg)](https://github.com/bake-bake-bake/bakeware/commits/main)

Compile Elixir applications into single, easily distributed executable binaries

![The Bakeware oven](https://raw.githubusercontent.com/bake-bake-bake/bakeware/main/assets/bakeware_logo200.png)

> Bakeware was made over a weekend at [SpawnFest 2020](https://spawnfest.github.io/)
> and thanks to the response from the community, we're working on finishing it
> off. While it's not ready for production, it's definitely ready for
> experimentation - just expect APIs to change in the near-term. If you'd like
> to help, please let us know and stay tuned!

Bakeware extends [Mix
releases](https://hexdocs.pm/mix/1.10.4/Mix.Tasks.Release.html) with the ability
to turn Elixir projects into single binaries that can be copied and directly
run. No need to install Erlang or untar files. The binaries look and feel like
the build-products from other languages.

Here's a quick list of features:

* Simple - add the `bakeware` dependency and the Bakeware assembler to your Mix
  release settings
* Supports OSX and Linux (We wrote the code with Windows and the BSDs in mind,
  so support for those platforms may not be far off)
* [Zstandard compression](https://en.wikipedia.org/wiki/Zstandard) for smaller
  binaries
* Optional support for automatic software updates (work in progress)
* Command-line argument passing conveniences
* Lots of examples

This README contains the basics of making your applications work with `Bakeware`
and reference material for when you need to dig into how it works.

Since everything was written quickly and the integration is fairly
straightforward, we recommend that you take a look at the examples. The examples
are bare bones Elixir scripts, OTP applications, Phoenix applications and more
with small changes to their `mix.exs` files and instructions for running that
you can try out for yourself.

## Using

### Mix release

Bakeware supports tieing in executable binary assembly into a Mix release
as a step by using the `Bakeware.assemble/1` function.

<!-- ASSEMBLE !-->
This will assemble the necessary components to create a Bakeware executable
that can be distributed across machines to run the script/application without
extra environment setup (such as installing Elixir/Erlang, etc)

To use, add this to your release as a step after assembly:

```elixir
def release do
  [
    demo: [
      steps: [:assemble, &Bakeware.assemble/1]
    ]
  ]
end
```

Bakeware adds the following options in the release scoped to `:bakeware` key:

* `:compression_level` - Zstandard compression level (1 to 19) where higher
  numbers generally result in better compression, but are slower to build
* `:start_command` - The start script command to run when invoked. This defaults
  to `"start"`, but can be changed to `"start_iex"`, for example, if you want a
  prompt. See [`Mix.Release`](https://hexdocs.pm/mix/Mix.Tasks.Release.html#module-bin-release_name-commands)
  for supported commands.

```elixir
def release do
  [
    demo: [
      bakeware: [
        compression_level: 1,
        start_command: "daemon"
      ]
    ]
  ]
end
```

<!-- ASSEMBLE !-->

### Scripting

<!-- SCRIPT !-->
Bakeware supports an API similar to Erlang's escript for implementing a `main`
function. Here's an example module:

```elixir
defmodule MyApp.Main do
  use Bakeware.Script

  @impl Bakeware.Script
  def main(_args) do
    IO.puts "Hello, World!"
    0
  end
end
```

The return value sets the scripts exit status (0 for success and other values
for errors). Other value types are supported. See
[`:erlang.halt/2`](https://erlang.org/doc/man/erlang.html#halt-2) for how these
work.

Next, add this module to your `mix.exs`'s application description. This usually
looks something like this:

```elixir
  def application do
    [
      extra_applications: [:logger],
      mod: {Myapp.Main, []}
    ]
  end
```

Why does the module get added to `:mod`? Everything with Bakeware operates on
OTP Releases. The macros in `Bakeware.Script` add the scaffolding to invoke your
`main/1` function from the release.
<!-- SCRIPT !-->

## Tips

### Minimizing executable size

Bakeware binaries appear to have a lower bound of about 12 MB in size. We expect
that they can be made smaller out-of-the-box, but here are a few things you can
do:

1. Make sure `zstd` is installed to enable compression during assembly:
  * **MacOS**: `brew install zstd`
  * **Ubuntu**: `apt-get install zstd`
  * **Windows**: `choco install zstandard`
2. Build using `MIX_ENV=prod`. The default is `MIX_ENV=dev`, so be sure that the
   environment variable is set.
3. Run `rm -fr _build` and then `mix release`. During development cruft builds
   up in the release directory. Bakeware can't tell the difference between the
   important files and the cruft, so executables will slowly grow in size if you
   don't do a clean build.
4. Inspect your `_build/prod/rel/<name>` directory and especially under `lib`
   for files or dependencies that you might be including on accident.
5. Make sure that compile-time dependencies are marked as `runtime: false` in
   your `mix.exs` so that they're not included
6. Try raising the compression Zstandard compression level by setting
  `:compression_level` in the `mix.exs` release config

### Erlang distribution

Bakeware uses [Mix releases](https://hexdocs.pm/mix/Mix.Tasks.Release.html) and
inherits the default of starting of Erlang distribution. If you're using
Bakeware for commandline or other short-lived applications, this unnecessarily
starts Erlang distribution servers running and prevents two application
instances from running at a time.

To disable, run `mix release.init` to create starter `env.sh.eex` and
`env.bat.eex` files in the `rel` directory. Then edit the files to set
`RELEASE_DISTRIBUTION=none`.

### Creating cross-platform binaries

Bakeware binaries include the Erlang runtime but there are still dependencies on
the host system. These include the C runtime and other libraries referenced by
the Erlang runtime and any NIFs and ports in your application. Luckily, the
binary ABIs of many libraries are very stable, but if distributing to a wide
audience, it's useful to build on a system with older library versions. Python
has a useful pointers in their [packaging
guides](https://packaging.python.org/guides/packaging-binary-extensions/#building-binary-extensions).

### Static Compiling OpenSSL into Erlang Distribution

Sometimes wierd SSL state bugs arise with a release when openssl is not statically compiled in.
You're affected by these ssl issues if you see things like an SSL connection being established but
after sending the first packet the remote end drops you. Attached is a Dockerfile that can be 
built with Podman and used to build your baked released.  
  
```
#Edit the versions of libraries in Dockerfile
ENV SSL_VERSION=1.1.1j
ENV OTP_VERSION=OTP-23.1.4
ENV ELIXIR_VERSION=v1.11.3

#Build erlang with static openssl
podman build --tag mybuilder DockerfileFolder/

#Bake your release
podman run -it --rm -v .:/root/myproject --entrypoint bash mybuilder -c "cd /root/myproject && ./build.sh"

#Build.sh
export MIX_ENV=prod
rm -rf _build
mix deps.get
mix release
cp _build/prod/rel/bakeware/myproject .
```

### Building on Windows

Bakeware is tested to work in mingw environment on Windows 8 and 10. In order to setup the environment follow these steps:

* Install [`chocolatey`](https://chocolatey.org/install)
* Install elixir, zstandard, make, and mingw using chocolatey: `choco install -y elixir zstandard make mingw`
* Change the default `MAKE` environment variable used by `elixir_make` from `nmake` to `make` (set it permanently to get rid of the errors in VSCode)
* Set the CC environment variable
* Build the release

#### PowerShell

```powershell
$env:MAKE="make"
$env:CC="gcc"
mix release
```

#### Command Prompt

```
set MAKE=make
set CC=gcc
mix release
```

#### MinGW

Note: after building the release in MinGW, you need to switch back to PowerShell/CMD to run the application

```bash
export MAKE=make
export CC=gcc
mix release
```

## Reference material

### Command-line arguments

In general, command-line arguments passed to Bakeware applications are passed through to Elixir. A few special command-line arguments can be passed to adjust the launchers behavior. Bakeware stops parsing command-line arguments when it encounters a `--`. Processed command-line arguments are not passed along to Elixir.

The following arguments may be passed:

* `--bw-info` - Print out information about the application and exit
* `--bw-gc` - This cleans up all unused entries in the cache (NOT IMPLEMENTED)
* `--bw-install` - Unpack the application to the cache only. Do not run.
* `--bw-system-install` - Install to a system-wide location (NOT IMPLEMENTED)
* `--bw-command` - use the specified [`Mix.Release` command](https://hexdocs.pm/mix/Mix.Tasks.Release.html#module-bin-release_name-commands) when running the executable

### Environment variables

The Bakeware launcher sets the following environment variables for use in Elixir:

Variable name                       | Description
 ---------------------------------- | --------------------------
`BAKEWARE_EXECUTABLE`               | The absolute path to the executable
`BAKEWARE_ARG1`                     | The first command-line argument
`BAKEWARE_ARGn`                     | The nth command-line argument
`BAKEWARE_ARGC`                     | The number of arguments

See the [Scripting](#scripting) section of this document for a more user friendly API.

### Binary format

Bakeware application binaries look like this:

* Bakeware application launcher
* A CPIO archive of an Erlang/OTP release
* Trailer

The CPIO archive can be compressed. This depends on the contents of the trailer.

Trailer format (multi-byte fields are big endian):

Offset from end | Field           | Type           | Description
 -------------- | --------------- | -------------- | -----------
-4              | Magic           | 4 byte string  | Set to "BAKE"
-5              | Trailer version | 8-bit integer  | Set to 1
-6              | Compression     | 8-bit integer  | 0 = No compression, 1 = Zstandard
-8              | Flags           | 16-bit integer | Set to 0 (no flags yet)
-12             | Contents offset | 32-bit integer | Offset of CPIO archive
-16             | Contents length | 32-bit integer | Length of CPIO archive
-48             | SHA1            | 20 bytes       | SHA-1 of the CPIO archive

## Cache directory

Bakeware maintains a cache of extracted binaries. This is needed to run the
OTP releases and it enables start-time optimizations.

The default cache directory location is system-specific:

* Windows - `"C:/Users/<USER>/AppData/Local/Bakeware/cache"`
* MacOS - `"~/Library/Caches/Bakeware"`
* Linux and other Unixes - `"~/.cache/bakeware"`

You can override it by setting the `$BAKEWARE_CACHE` environment variable.

Here's the layout of each cache entry:

Path                                | Created by | Description
 ---------------------------------- | ---------- | --------------------------
`$CACHE_DIR/$SHA1/bin`              | CPIO       | OTP release's `bin` directory
`$CACHE_DIR/$SHA1/erts-x.y.z`       | CPIO       | OTP release's ERTS
`$CACHE_DIR/$SHA1/lib`              | CPIO       | OTP release's `lib` directory
`$CACHE_DIR/$SHA1/releases`         | CPIO       | OTP release's `releases` directory
`$CACHE_DIR/$SHA1/start`            | CPIO       | Start script. E.g., `bin/my_otp_release start`

## LICENSE

All code is licensed under Apache-2.0 with the exception of [`zstd`](https://github.com/bake-bake-bake/bakeware/tree/main/src/zstd) which is dual licensed BSD/GPL. See it's [LICENSE](https://github.com/bake-bake-bake/bakeware/blob/main/src/zstd/LICENSE) and [COPYING](https://github.com/bake-bake-bake/bakeware/blob/main/src/zstd/COPYING) files for more details.
