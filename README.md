# Managarm: Distribution Build Scripts

![Sanity Checks](https://github.com/managarm/bootstrap-managarm/workflows/Sanity%20Checks/badge.svg)

This repository contains patches and build script to build a [Managarm](https://github.com/managarm/managarm) kernel and userspace.

## Building a Managarm Distribution from Source

Since it is almost impossible to make sure all build environments will play nice with
the build system, **we strongly recommended to set up a build environment in a container**,
using the provided Dockerfile.

Make sure that you have enough disk space. As managarm builds a lot of large external packages
(like GCC, binutils, coreutils and the Wayland stack), about 20 - 30 GiB are required.

### Preparations

1.  First and foremost we will create a directory (`$MANAGARM_DIR`) which will be used for storing the source
    and build directories.
    ```bash
    export MANAGARM_DIR="$HOME/managarm" # In this example we are using $HOME/managarm, but it can be any directory
    mkdir -p "$MANAGARM_DIR" && cd "$MANAGARM_DIR"
    ```
1.  The `xbstrap` build system is required on the host (whether you build in a container or not).
    `xbstrap` can be installed via pip3: `pip3 install xbstrap`.
1.  Clone this repository into a `src` directory and create a `build` directory:
    ```bash
    git clone https://github.com/managarm/bootstrap-managarm.git src
    mkdir build
    ```

### Alternative I: setting up a container (recommended)

*Note: this step is not needed if you don't want to use a Docker container, if so skip to the next paragraph.*

1.  A working `docker` installation is required to perform a containerized build.
1.  Build a Docker image from the provided Dockerfile:
    ```bash
    docker build -t managarm-buildenv --build-arg=USER=$(id -u) src/docker
    ```
1.  Create a `bootstrap-site.yml` file inside the `build` directory containing:
    ```yml
    container:
        runtime: docker
        image: managarm-buildenv
        src_mount: /var/bootstrap-managarm/src
        build_mount: /var/bootstrap-managarm/build
        allow_containerless: true
    ```
    This `bootstrap-site.yml` will instruct our build system to invoke the build scripts within your container image.

Now proceed to the Building paragraph.

### Alternative II: installing dependencies on the host system

*Note: if you built a Docker image in the previous step, skip this paragraph.*

1.  Certain programs are required to build managarm;
    here we list the corresponding Debian packages:
    `build-essential`, `pkg-config`, `autopoint`, `bison`, `curl`, `flex`, `gettext`, `git`, `gperf`, `help2man`, `m4`, `mercurial`, `ninja-build`, `python3-mako`, `python3-yaml`, `texinfo`, `unzip`, `wget`, `xsltproc`, `xz-utils`, `libexpat1-dev`, `rsync`, `python3-pip`, `python3-libxml2`, `netpbm`, `itstool`, `zlib1g-dev`, `libgmp-dev`, `libmpfr-dev`, `libmpc-dev`, `subversion`, `gawk`, `libwayland-bin`, `libpng-dev`, `gtk-doc-tools`, `groff`, `libglib2.0-dev-bin`, `ragel`, `libtasn1-bin`, `libssl-dev`.
1.  `meson` is required. There is a Debian package, but as of Debian Stretch, a newer version is required.
    Install it from pip: `pip3 install meson`
1.  `protobuf` is also required. There is a Debian package, but a newer version is required.
    Install it from pip: `pip3 install protobuf`
1.  [bragi](https://github.com/managarm/bragi) is required to build managarm. It can be installed via pip: `pip3 install bragi`.
1.  For managarm kernel documentation you may also want `mdbook`. This requires `rust` & `cargo` to be installed.
    Install it using cargo: `cargo install --git https://github.com/rust-lang/mdBook.git mdbook`

### Building

1.  Initialize the build directory with
    ```bash
    cd build
    xbstrap init ../src
    ```
1.  There are several meta-packages available which control what software is build, this means that the build can be started using one of the following commands:

    If you want the full managarm experience with a selection of terminal and gui software available to try out use
    ```bash
    xbstrap install weston-desktop
    ```
    If you want to boot into `kmscon` and have some functional commands to play around with use
    ```bash
    xbstrap install base
    ```
    If you want some development tools in addition to the functionality of `base` use
    ```bash
    xbstrap install base-devel
    ```
    If you only want to build the bare minimum to boot into `kmscon` use
    ```bash
    xbstrap install --all
    ```
    Note that this command can take multiple hours, depending on your machine.

## Creating Images

*Note: these commands are meant to be ran in the `build` directory.*

After managarm's packages have been built, building a HDD image of the system
is straightforward. The [image_create.sh](https://github.com/qookei/image_create) script
can be used to create an empty HDD image.

Download the `image_create.sh` script and mark it executable:
```bash
wget 'https://raw.githubusercontent.com/qookei/image_create/master/image_create.sh'
chmod +x image_create.sh
```

This repository contains an `update-image.py` script to mount the image and copy the system onto it.

This script can use 3 different methods to copy files onto the image:
1. Using libguestfs (the default method).
2. Using a classic loopback and mount (requires root privileges). This is the safe fallback method most guaranteed to work.
3. Via Docker container. This is handy in case the user is in the `docker` group since it does not require additional root authentication.

Going with method 1 will require `libguestfs` to be installed on the host.
After installing `libguestfs` it might be necessary to run the following:
```bash
sudo install -d /usr/lib/guestfs
sudo update-libguestfs-appliance
```

If you're using xbstrap to invoke the script, selecting the desired method can be done by adding the following to the `bootstrap-site.yml` file:
```
define_options:
    mount-using: 'loopback' # or guestfs/docker
```

When invoking the script directly, selecting the method can be done with the `--mount-using` argument.

Furthermore, all methods require `rsync` installed on the host as well.

Hence, running the following commands in the build directory
should produce a working image and launch it using QEMU:
```bash
# Create a HDD image file called 'image'.
# Check "./image_create.sh -h" for more options.
./image_create.sh -o image -s 4GiB -t ext2 -p gpt -l limine -be

# Copy the system onto it.
xbstrap run make-image

# Alternatively, you can call the necessary scripts manually:
# Check their help messages for more information.
../src/managarm/tools/gen-initrd.py
../src/scripts/update-image.py

# To launch the image using QEMU, you can run:
../src/scripts/run-qemu
```

