# gprovision

Source code related to factory restore and provisioning (aka manufacture or imaging). Also includes buildroot files.

## target platform

x86 linux

This has only been tested on x86, but there shouldn't be much that is x86-specific. Buildroot is used to get a repeatable build of AMD64 utilities, as well as x86-32 libraries (one tool we use is only available in 32-bit versions).

## packages that may be of particular interest

* provisioning, aka manufacture `gprovision/pkg/mfg`
* factory restore `gprovision/pkg/recovery`
* data erase `gprovision/pkg/erase`
* linux /init binary `gprovision/cmd/init`, `gprovision/pkg/init`
    * determines whether to erase, factory restore, or do normal boot
    * or, with the appropriate build tag, provisions the unit
* integ tests for the above
* crystalfontz lcd code, including menus `gprovision/pkg/hw/cfa`
* partitioning/formatting/bootability code for uefi and legacy `gprovision/pkg/recovery/disk`
* flexible logging to multiple sinks (lcd, file, console) `gprovision/pkg/log`
* appliance package identifies the hardware variant it's running on `gprovision/pkg/appliance`
* config of ipmi account, password `gprovision/pkg/hw/ipmi`

## proprietary/oss

Some packages couldn't be open sourced, and a few provided functionality that is absolutely required; for those, replacements were written and live under `gprovision/pkg/oss`. These packages are related to remote logging, record keeping, and factory restore data persistence.

To facilitate open source release while still using proprietary packages within genesys, `gprovision/pkg/common` (and subpackages) was added. This uses interfaces to abstract away exactly what package is providing the functionality. In a few dirs, you will find an `oss.go` file; internally, we use a `proprietary.go` in place of `oss.go`, to similar effect. These files have the effect of initializing interface vars in the aforementioned common package.

If you wish to customize gprovision's behavior, start by looking at the interfaces in `common` - re-implementing one or more of these may provide all you need.

## security

The code we could open source does *not* include a mechanism for secure storage of passwords. Password storage is required for the integ tests, so some sort of implementation was required. To discourage use of the simplistic and weak mechanism included in this code, it deliberately uses annoying passwords and emits warnings.

## building

### GOPATH, PATH
With bash on linux, run
    . ./bash_env.sh
to set up your env. This also runs `dep ensure` if it looks like you haven't, and lists mage targets.

For other shells, you'll need to set GOPATH and update PATH to include the repo's `/bin` so that our `mage` is found before any version you may have installed.

### TEMPDIR

For integ tests, your /tmp must have plenty of free space. If it has < 8GB or so, set the TEMPDIR env var to point to a dir that does have that much free space.

Passing tests delete their temp dirs, but failing tests generally do not. Re-running a failed integ test will delete temp dirs from previous runs of that test. This delete is done with a  wildcard like `test-gprov-erase*`, so there is a slight but non-zero chance it would delete something it didn't create.

### mage
Using mage is not necessary for simple packages like the crystalfontz code, but it is needed for things like the kernels with embedded initramfs and for integ tests. These integ tests depend on env vars mage sets, and are skipped if the vars are not set.

Due to required dependencies, mage will not run until after you run `dep` - see its section below.

Mage targets (listed with `mage -l`) are case insensitive, and each corresponds to a public function in $GOPATH/src/mage. For example, `mage bins:embedded` runs the function with signature `func (Bins) Embedded(ctx context.Context)`.

### qemu
qemu is also needed for integ tests. See `qemu/` in the repo root for a Dockerfile that'll build qemu and copy the necessary files out. A pre-built version is available on github under releases, and is automatically downloaded by mage.

### dep
dep is needed. Once installed, `cd ./gopath/src/gprovision` and run `dep ensure` to download all dependencies.

### kernel
All tools required to build the kernel are needed - gcc, flex, bison, make, binutils, etc. See kernel documentation for details.

#### kernel version
The kernel source to download is determined from the name of the `linux-*.config` file.

### buildroot
Files in `brx/` are inputs to build a cpio of non-go binaries. A pre-built version is available on github under releases, and is automatically downloaded by mage.

### Known to work with...
Known to work with `go v1.12.5` or higher, and the latest `dep`.

### go packages
To pull in dependencies:

    cd $GOPATH/src/gprovision
    dep ensure

### verify
Once that is done, running `mage -l` from any dir should print a list of targets.

`mage tests:integ`, for example, will build prerequisites and then run integ tests.

## Known issues

The integ tests are occasionally flaky, as are the crystalfontz tests.

## Sequence

### Provision, AKA manufacture

    +---------+     +---------+
    | Legacy  |     | UEFI    |
    | PXEboot |     | PXEboot |
    +----+----+     +----+----+
         |               |
         +-------+-------+
                 |
                 v
          +------+------+
          |provision.pxe|
          +------+------+
                 |
                 v
       +---------+----------+
       | load json from url |
       |  passed in kernel  |
       | parameter 'mfgurl' |
       +---------+----------+
                 |
                 v
     +-----------+-------------+
     | verify hardware specs,  |
     | download image & write  |
     |   to recovery volume,   |
     | write credentials, etc  |
     +-----------+-------------+
                 |
                 v
        +--------+---------+
        |  configure boot  |
        | (legacy or uefi) |
        +--------+---------+
                 |
                 v
            +----+-----+
            |  reboot  |
            +----------+

### All boots after manufacture

      +--------+   +------+
      | Legacy |   | UEFI |
      |  boot  |   | boot |
      +----+---+   +---+--+
           |           |
           +-----+-----+
                 |
                 v
        +--------+--------+
        | "normal" kernel |
        +--------+--------+
                 |
                 v
         +-------+-------+
         | erase needed? |
         +--+----------+-+
            |          |
            v          v
          +-+--+    +--+--+
          | no |    | yes |
          +-+--+    +--+--+
            |          |
            |          v
            |      +---+---+     +--------+
            |      | erase +---->+ reboot |
            |      +-------+     +--------+
            |
            v
     +------+---------------+
     | locate primary drive |
     +----------+-----------+
                |
                v
    +-----------+-----------+
    | does flag file exist? |
    +--+--------------+-----+
       |              |
       v              v
    +--+-+         +--+--+
    | no |         | yes |
    +--+-+         +--+--+
       |              |
       |              v
       |       +------+--------+
       |       |   all good,   |     +----------+
       |       | start systemd +---->+ eventual |
       |       |   from disk   |     |  reboot  |
       |       +---------------+     +----------+
       v
     +-+---------------------+
     | start factory restore |
     +----------+------------+
                |
                v
     +----------+-------------------------+
     |                                    |       +--------+
     | find latest good image on recovery +------>+ reboot |
     | format primary & write image       |       +--------+
     | write network config               |
     | write flag file                    |
     | etc                                |
     |                                    |
     +------------------------------------+
