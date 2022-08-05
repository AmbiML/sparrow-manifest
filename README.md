# Project Sparrow

Sparrow is a project to build a low-power secure embeded platform
for Ambient ML applications. The target platform leverages
[RISC-V](https://riscv.org/) and [OpenTitan](https://opentitan.org/).
The Sparrow
software includes a home-grown operating system named KataOS, that runs
on top of [seL4](https://github.com/seL4) and (ignoring the seL4 kernel)
is written almost entirely in [Rust](https://www.rust-lang.org/).

Sparrow (and KataOS) are definitely a work in progress. The KataOS
components are based on an augmented version of seL4's
[CAmkES framework](https://docs.sel4.systems/projects/camkes/).
Critical system services
are CAmkES components that are statically configured. Applications are
developed using an AmbiML-focused SDK and dynamically loaded by the
system services.

[Initially the Sparrow repository only includes the Rust frameworks
for developing seL4 / CAmkES components. A later release will share
the KataOS services including the MemoryManager and ProcessManager that
support dynamic operation of applications.]

## Sparrow software repositories (what's included here).

Sparrow consists of multiple git repositories stitched together with the [repo
tool](https://gerrit.googlesource.com/git-repo/+/refs/heads/master/README.md).
The following git repositories are currently available:

- *camkes-tool*:
    seL4's camkes-tool repository with additions to support KataOS services
- *capdl*:
    seL4's capdl repository with addition for KataOS services and the
     KataOS rootserver (a replacement for capdl-loader-app that is written
     in Rust and supports hand-off of system resources to the KataOS
     MemoryManager service)
- *kernel*:
    seL4's kernel with drivers for Sparrow's RISC-V platform and support
    for reclaiming the memory used by the KataOS rootserver
- *kata*
    frameworks for developing in Rust, and (eventually) the KataOS system services
- *scripts*:
    support scripts including build-sparrow.sh

[More software will be published as we deem it ready for sharing until eventually
all of Sparrow (software and hardware designs) will be available.]

Most KataOS Rust crates are in the *kata/apps/system/components* directory.
Common/shared code is in *kata-os-common*:

- *allocator*: a heap allocator built on the linked-list-allocator crate
- *camkes*: support for writing CAmkES components in Rust
- *capdl*: support for reading the capDL specification generated by capDL-tool
- *copyregion*: a helper for temporarily mapping physical pages into a thread's VSpace
- *cspace-slot*: an RAII helper for the *slot-allocator*
- *logger*: seL4 integration with the Rust logger crate
- *model*: support for processing capDL; used by the kata-os-rootserver
- *panic*: an seL4-specific panic handler
- *sel4-config*: build glue for seL4 kernel configuration
- *sel4-sys*: seL4 system interfaces & glue
- *slot-allocator*: an allocator for slots in the top-level CNode

The other main Rust piece is the rootserver application that is located in
*projects/capdl/kata-os-rootserver*. This depends on the *capdl* and *model*
submodules of *kata-os-common*. It is possible to select either kata-os-rootserver
or the C-based capdl-loader-app with a CMake setting in the CAmkES project's
easy-settings.cmake file; e.g. `projects/kata/easy-settings.cmake` has:

```
#set(CAPDL_LOADER_APP "capdl-loader-app" CACHE STRING "")
set(CAPDL_LOADER_APP "kata-os-rootserver" CACHE STRING "")
```

Beware that kata-os-rootserver has very limited testing and likely does not
support and/or implement all the features of capdl-loader-app.

## How we do Sparrow software development.

Our primary development environment uses Renode for simulation of our Sparrow
hardware design. Renode allows us to do rapid software/hardware co-design of
our multi-core RISC-V target platform. The software environment is derived
from those provided by seL4 & OpenTitan. Debugging leverages Renode facilites
and [gdb talking to Renode](https://antmicro.com/blog/2022/06/sel4-userspace-debugging-with-gdb-extensions-in-renode/)
for system softare and application developement.

[For this initial roll-out we are targetting aarch64 running on qemu.]

## Getting started with repo & build-sparrow.

This repository contains scripts for use with the Sparrow open source
project. These script are meant to help new users get familiar with
the project, how it is built, and how to make use of Sparrow in your
own projects.

1. Clone this project from GitHub using the
   [repo tool](https://gerrit.googlesource.com/git-repo/+/refs/heads/master/README.md)
   We assume below this lands in a top-level directory named "sparrow".
2. Download, build, and run a sample CAmkES application for a specified
   target architecture. For now the only target that works is "aarch64"
   (for a raspi3b machine running in simulation on qemu).

``` shell
mkdir sparrow
cd sparrow
repo init -u https://github.com/google/AmbiML/sparrow-manifest -m camkes-manifest.xml
repo sync -j$(nproc)
sh scripts/build-sparrow.sh aarch64
(cd build-aarch64; ./simulate -M raspi3b)
```

Note the above assumes you have the follow prerequisites installed on your system
and **in your shell's search path**:
1. Gcc (or clang) for the target architecture
2. Rust; at the moment this must be nightly-2021-11-05-x86_64-unknown-linux-gnu (or be
   prepared to edit at least build-sparrow.sh). Beware that we override the default TLS
   model to match what CAmkES uses and this override is not supported by stable versions of Rust.
3. Whichever simulator seL4 expects for your target architecture; e.g. for aarch64 this
   is qemu-system-aarch64.

Sparrow uses [repo](https://gerrit.googlesource.com/git-repo/+/refs/heads/master/README.md)
to download and piece together Sparrow git repositories as well as dependent projects /
repositories such as [seL4](https://github.com/seL4).

``` shell
$ repo init -u https://github.com/google/AmbiML/sparrow-manifest -m camkes-manifest.xml
Downloading Repo source from https://gerrit.googlesource.com/git-repo

repo has been initialized in <your-directory>/sparrow/
If this is not the directory in which you want to initialize repo, please run:
   rm -r <your-directory>/sparrow//.repo
and try again.
$ repo sync -j12
Fetching: 100% (23/23), done in 9.909s
Garbage collecting: 100% (23/23), done in 0.218s
Checking out: 100% (23/23), done in 0.874s
repo sync has finished successfully.
$ sh scripts/build-sparrow.sh aarch64
info: component 'rust-std' for target 'aarch64-unknown-none' is up to date
loading initial cache file <your-directory>/sparrow/projects/camkes/settings.cmake
-- Set platform details from PLATFORM=rpi3
--   KernelPlatform: bcm2837
--   KernelARMPlatform: rpi3
-- Setting from flags KernelSel4Arch: aarch64
-- Found seL4: <your-directory>/sparrow/kernel
-- The C compiler identification is GNU 11.2.1
...
[291/291] Generating images/capdl-loader-image-arm-bcm2837
$ cd build-aarch64; ./simulate -M raspi3b
./simulate: qemu-system-aarch64 -machine raspi3b  -nographic -serial null -serial mon:stdio -m size=1024M  -kernel images/capdl-loader-image-arm-bcm2837
ELF-loader started on CPU: ARM Ltd. Cortex-A53 r0p4
  paddr=[8bd000..fed0ff]
No DTB passed in from boot loader.
Looking for DTB in CPIO archive...found at 9b3ef8.
Loaded DTB from 9b3ef8.
   paddr=[23c000..23ffff]
ELF-loading image 'kernel' to 0
  paddr=[0..23bfff]
  vaddr=[ffffff8000000000..ffffff800023bfff]
  virt_entry=ffffff8000000000
ELF-loading image 'capdl-loader' to 240000
  paddr=[240000..4c0fff]
  vaddr=[400000..680fff]
  virt_entry=4009e8
Enabling MMU and paging
Jumping to kernel-image entry point...

Warning:  gpt_cntfrq 62500000, expected 19200000
Bootstrapping kernel
Booting all finished, dropped to user space
kata_os_rootserver::Bootinfo: (725, 131072) empty slots 1 nodes (14, 74) untyped 131072 cnode slots
kata_os_rootserver::Model: 676 objects 1 irqs 0 untypeds 1 asids
kata_os_rootserver::capDL spec: 0.13 Mbytes
kata_os_rootserver::CAmkES components: 1.07 Mbytes
kata_os_rootserver::Rootserver executable: 1.31 Mbytes
<<seL4(CPU 0) [decodeARMFrameInvocation/2137 T0xffffff80004c7400 "rootserver" @44373c]: ARMPageMap: Attempting to remap a frame that does not belong to the passed address space>>
client: what's the answer to 342 + 74 + 283 + 37 + 534 ?
adder: Adding 342
adder: Adding 74
adder: Adding 283
adder: Adding 37
adder: Adding 534
client: result was 1270
```

The build-sparrow.sh script can be run repeatedly.  If you need to reset
your setup just remove the build tree and re-run build-sparrow.sh; e.g.

``` shell
cd sparrow
rm -rf build-aarch64
sh scripts/build-sparrow.sh aarch64
```

## Setting up a local project

The camkes-manifest.xml file used to initialize the sparrow repository
is specially constructed for building sample CAmkES tests from upstream
seL4. To build your own seL4 / CAmkES projects you can follow the seL4
[documentation](https://docs.sel4.systems/projects/camkes/).

### Depending on Rust crates

To use crates from Sparrow you can reference them from a local repository or
directly from GitHub using git; e.g. in a Config.toml:
```
kata-os-common = { path = "../system/components/kata-os-common" }
kata-os-common = { git = "https://github.com/google/AmbiML/sparrow/kata" }
```
NB: the git usage depends on cargo's support for searching for a crate
named "kata-os-common" in the kata repo.
When using a git dependency a git tag can be used to lock the crate version.

Note that many Sparrow crates need the seL4 kernel configuration
(e.g. to know whether MCS is configured). This is handled by the
kata-os-common/sel4-config crate that is used by a build.rs to import
kernel configuration parameters as Cargo features. In a Cargo.toml create
a features manifest with the kernel parameters you need e.g.

```
[features]
default = []
# Used by sel4-config to extract kernel config
CONFIG_PRINTING = []
```

then specify build-dependencies:

```
[build-dependencies]
# build.rs depends on SEL4_OUT_DIR = "${ROOTDIR}/out/kata/kernel"
sel4-config = { path = "../../kata/apps/system/components/kata-os-common/src/sel4-config" }
```

and use a build.rs that includes at least:

```
extern crate sel4_config;
use std::env;

fn main() {
    // If SEL4_OUT_DIR is not set we expect the kernel build at a fixed
    // location relative to the ROOTDIR env variable.
    println!("SEL4_OUT_DIR {:?}", env::var("SEL4_OUT_DIR"));
    let sel4_out_dir = env::var("SEL4_OUT_DIR")
        .unwrap_or_else(|_| format!("{}/out/kata/kernel", env::var("ROOTDIR").unwrap()));
    println!("sel4_out_dir {}", sel4_out_dir);

    // Dredge seL4 kernel config for settings we need as features to generate
    // correct code: e.g. CONFIG_KERNEL_MCS enables MCS support which changes
    // the system call numbering.
    let features = sel4_config::get_sel4_features(&sel4_out_dir);
    println!("features={:?}", features);
    for feature in features {
        println!("cargo:rustc-cfg=feature=\"{}\"", feature);
    }
}
```

Note how build.rs expects an SEL4_OUT_DIR environment variable that has the path to
the top of the kernel build area. The build-sparrow.sh script sets this for you but, for
example, if you choose to run ninja directly you will need it set in your environment.

Similar to SEL4_OUT_DIR the kata-os-common/src/sel4-sys crate that has the seL4 system
call wrappers for Rust programs requires an SEL4_DIR envronment variable that has the
path to the top of the kernel sources. This also is set by build-sparrow.sh.

## Source Code Headers

Every file containing source code includes copyright and license
information. For dependent / non-Google code these are inherited from
the upstream repositories. If there are Google modifications you may find
the Google Apache license found below.

Apache header:

    Copyright 2022 Google LLC

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

        https://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
