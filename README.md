# Table of Contents
- [`update-ports`](#update-ports): Syncs and upgrades ports, merges configuration changes, and performs housekeeping.
  - [`portmaster.patch`](#portmasterpatch): Adds a `--packages-if-stock` switch to `portmaster(8)` which selectively builds from source or installs from packages depending on whether ports' options have been customized.
- [`update-system`](#update-system): Syncs and upgrades the kernel and world, merges configuration changes, and cleans up.

# `update-ports`
📢 Requires `ports-mgmt/portmaster`.

This script automates the process of syncing the ports tree, identifying changes in `UPDATING` that require action, upgrading ports, merging inbound changes to sample configuration files into their local counterparts, cleaning up obsolete configuration, distfiles and packages, and checking ports for inconsistencies.
## Features
- Syncs the ports tree using either `git(1)` or `svnlite(1)` as appropriate
- Displays new entries in `UPDATING` added since the last sync
- Selectively upgrades ports from packages or source depending on whether they have custom options*
- Automatically merges most changes to ports' sample configuration files into their local counterparts
- Cleans up redundant leaf ports, distfiles, packages, and configuration directories
- Checks ports for checksum, dependency, and shared library problems

(*) Selectively upgrading ports from binary packages vs. source code depending on whether their options have been modified requires [`portmaster.patch`](#portmasterpatch).
## Output
```
>>> Sync ports tree? [Y/n]
    💡 Updates source using either git(1) or svnlite(1) as appropriate.

>>> See changes in UPDATING? [Y/n]
    💡 Displays additions to the ports changelog since the last sync.

>>> Back up ports' sample configuration files for post-upgrade merging? [Y/n]

>>> Upgrade portmaster(8)? [Y/n]

>>> Patch portmaster(8)? This adds a --packages-if-stock switch which selectively
    installs ports with stock configuration from binary packages while building
    customized ports from source. [Y/n]

>>> Upgrade ports using patched portmaster(8) with --packages-if-stock? [Y/n]
    ===>>> Update mariadb105-client-10.5.11 to mariadb105-client-10.5.12? y/n [y]
    ===>>> Modified options found; building from source ✨
           GSSAPI_BASE: on => off (GSSAPI support via base system (needs Kerberos))
           GSSAPI_NONE: off => on (Disable GSSAPI support)
    ===>>> Update binutils-2.33.1_4,1 to binutils-2.37,1? y/n [y]
    ===>>> No modified options; installing from package ✨

>>>  Merge inbound changes to sample configuration into local files? [Y/n]
    >>> postfix/main.cf.sample differs from postfix/main.cf.sample.orig.
        Attempt to automatically merge the differences into postfix/main.cf? [Y/n]
        [√] Differences successfully merged. ✨
    >>> apache24/httpd.conf.sample differs from apache24/httpd.conf.sample.orig.
        Attempt to automatically merge the differences into apache24/httpd.conf? [Y/n]
        [X] Only some changes applied cleanly:
            …
            >>> Merge interactively? [Y/n]

>>> Delete backups of pre-upgrade sample configuration files? Answer
    affirmatively if you have merged all inbound changes. [Y/n]

>>> Check for dependency-installed ports that may no longer be needed? [Y/n]

>>> Delete any and all obsolete distfiles? [Y/n]

>>> Delete any and all obsolete binary packages? [Y/n]

>>> Check for orphaned configuration directories in /var/db/ports? [Y/n]

>>> Check ports for checksum, dependency, and shared library problems? [Y/n]
```

## `portmaster.patch`
This patch augments `portmaster(8)` with a new `--packages-if-stock` switch which selectively:
- builds from source ports whose options have been modified from the defaults, and
- installs from binary packages ports whose options are unmodified (i.e. stock).

Unlike `portmaster(8)`'s `-P`(`P`) switch, which if used causes _all_ ports to be unconditionally installed from binary packages (and if not used results in _all_ ports being unconditionally built from source), `--packages-if-stock` individually analyzes each port's options and, depending on whether they have been customized, determines the appropriate type of upgrade for each port.

If a port has stock configuration and can be installed from a binary package but no suitable package is available (e.g. the mirror is unreachable or the package is older than the source), `--packages-if-stock` gracefully falls back to building from source like `portmaster(8)`'s `-P`/`--packages` switch.

💡 This patch may be used either independently of in conjunction with the [`update-ports`](#update-ports) tool, which offers to automatically patch `portmaster(8)` and invoke it with `--packages-if-stock` if the user accepts the patch.
## Output
```
$ sudo portupgrade -i --packages-if-stock binutils mariadb105-server

===>>> Update binutils-2.33.1_4,1 to binutils-2.37,1? y/n [y]
===>>> No modified options; installing from package ✨
…
===>>> Update mariadb105-server-10.5.11 to mariadb105-server-10.5.12? y/n [y]
===>>> Modified options found; building from source ✨
       CONNECT_EXTRA: on => off (Enable ODBC and XML in CONNECT engine)
       DOCS: on => off (Build and/or install documentation)
       WSREP: on => off (Build wsrep clustering)
       ZSTD: off => on (Zstandard compression support (RocksDB only))
       INNOBASE: on => off (InnoDB default engine)
       ROCKSDB: off => on (RocksDB LSM engine)
       SPHINX: on => off (SphinxSE engine)
       SPIDER: on => off (Partitioning and XA-transactions engine)
       GSSAPI_BASE: on => off (GSSAPI support via base system (needs Kerberos))
       GSSAPI_NONE: off => on (Disable GSSAPI support)
…
===>>> The following actions will be taken if you choose to proceed:
       Upgrade binutils-2.33.1_4,1 to binutils-2.37,1 from package ✨
       Upgrade mariadb105-server-10.5.11 to mariadb105-server-10.5.12 from source ✨
```

# `update-system`
This script automates the process of syncing the source tree, identifying changes in `UPDATING` that require action, building and installing the kernel and world, merging inbound changes to configuration files (including `GENERIC`) into their local counterparts, and cleaning up obsolete files after a build.
## Features
- Allows easy switching of branches (e.g. `RELEASE` → `STABLE`)
- Syncs the source tree using either `git(1)` or `svnlite(1)` as appropriate
- Offers to install package `devel/git-tiny` if no Git client is present
- Displays new entries in `UPDATING` added since the last sync
- Supports both `etcupdate(8)` and `mergemaster(8)` for configuration merging
- Offers to bootstrap `etcupdate(8)` before its first use
- Automatically merges most changes to `GENERIC` into the local kernel configuration
- Cleans up obsolete files, directories, and shared libraries
## Usage
```
Usage: update-system --stage1 [kernconf]
                     --stage2
                     --stage3
Stages:
    --stage1: Updates the source tree, shows changes in UPDATING, merges
              inbound changes to GENERIC into `kernconf` (if one is given),
              builds the userland, and builds and installs the kernel.
      [kernconf]: File name (not path) of the kernel configuration to build.
              Defaults to GENERIC if unspecified.
    --stage2: Installs the userland and merges changes in configuration files.
    --stage3: Upgrades pkg(8), cleans build output, and removes obsolete files.
```
## Output
### `--stage1 MYKERNEL`
```
Kernel configuration: MYKERNEL

>>> Keep the same branch, releng/12.2? [Y/n] n
    >>> New branch (e.g. releng/xx.y): stable/13
    💡 Updates source using either git(1) or svnlite(1) as appropriate.

>>> See changes in UPDATING? [Y/n]
    💡 Displays additions to the system's changelog since the last sync.

>>> Build the userland (`make -j buildworld`)? [Y/n]

>>> Clean the build directory before building? [Y/n]

>>> Build the MYKERNEL kernel (`make -j4 buildkernel KERNCONF=MYKERNEL`)? [Y/n]

>>> Attempt automatic merge of changes from GENERIC into MYKERNEL? [Y/n]
    💡 Merges inbound changes to the default configuration into the custom MYKERNEL.
    >>> Interactively merge GENERIC and MYKERNEL? [Y/n]
        💡 If automatic merging is declined or fails.

>>> Install the MYKERNEL kernel (`make installkernel KERNCONF=MYKERNEL`)?

>>> Reboot and activate the new MYKERNEL kernel? [Y/n]

=======================================================================================
After the reboot, run `update-system --stage2` to continue.
=======================================================================================
```
### `--stage2`
```
etcupdate(8) is set as the preferred configuration merging tool, but it needs
to be bootstrapped before its first use. Bootstrapping consists of making a copy
of the stock configuration in /usr/src (which should match the currently running
world) and establishing it as the baseline for detecting local changes.

>>> [B]ootstrap and use etcupdate(8), or [f]all back to mergemaster(8)? b
    Bootstrapping etcupdate(8); please wait... done.

>>> Merge pre-installworld configuration with `etcupdate -p`? [Y/n]

>>> Install the userland (`make installworld`)? [Y/n]

>>> Merge configuration files with `etcupdate -B`? [Y/n]

>>> Reboot into the new userland? [Y/n]

=======================================================================================
After the reboot, run `update-system --stage3` to continue.
=======================================================================================
```
### `--stage3`
```
>>> Upgrade pkg(8)? [Y/n]

>>> Clean up obsolete files and directories (`make delete-old`)? [Y/n]

>>> Checking for obsolete shared libraries...
    …
    >>> Delete ALL obsolete shared libraries listed above?
        WARNING: There may be ports still using some of these libraries. [y/N]
    >>> Interactively delete redundant shared libraries one at a time? [Y/n]

>>> Clean build directory (`make cleanworld`)? Answer affirmatively if
    you are satisfied with the upgrade and have no further use for the build
    output (e.g. upgrading other hosts from the built binaries). [y/N]
```
