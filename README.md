# genie

[![ko-fi](https://www.ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/I3I1VA18)

A quick way into a systemd "bottle" for WSL

What does that even mean?

Well, this gives you a way to run systemd as pid 1, with all the trimmings, inside WSL 2. It does this by creating a pid namespace, the eponymous poor-man's-container "bottle", starting up systemd in there, and entering it, and providing some helpful shortcuts to do so.

If you want to try it, please read this entire document first, _especially_ the BUGS section.

## NOTE: WSL 2 ONLY

Note: it is only possible to run _systemd_ (and thus _genie_ ) under WSL 2; WSL 1 does not support the system calls required to do so. If you are running inside a distro configured as WSL 1, even if your system supports WSL 2, genie will fail to operate properly.

## INSTALLATION

If there is a package available for your distribution, this is the recommended method of installing genie.

### Debian

Dependent packages on Debian are _daemonize_, _dbus_, _dotnet-runtime-3.1_, _policykit-1_, _systemd_, and _util-linux_ . For the most part, these are either already installed or in the distro and able to be installed automatically.

The chief exception is _dotnet-runtime-3.1_ , for which you will need to follow the installation instructions here:

https://dotnet.microsoft.com/download/

To install, add the wsl-translinux repository here: https://packagecloud.io/arkane-systems/wsl-translinux , and install genie using the command:

_sudo apt install systemd-genie_

### Other Distros

Debian is the "native" distribution for _genie_ , for which read, "what the author uses". Said author, sadly, does not have the time required to track all the possible changes in other distros, or to maintain packages accordingly. Maintainers to manage the deviations files (see below) and provide packaging for other distros are *actively sought*. If, following the instructions below, you make it work on another distro for yourself, please consider stepping up.

#### Arch

For Arch Linux users, there are prebuilt packages available at:

https://aur.archlinux.org/packages/genie-systemd/ and

https://aur.archlinux.org/packages/genie-systemd-git/

The former of which is prebuilt and the latter of which compiles it from source. Both install all needed dependencies, save that for version 1.17-1.23, you will have to install .NET Core 3.0 runtime explicitly, and likewise .NET Core 3.1 runtime for 1.24 and above. A PKGBUILD file for this in Arch is available here for those who wish to use it: https://github.com/arkane-systems/genie/files/3827049/dotnet-PKGBUILD.tar.gz ; you can also make your own, and for security and good practice, please always check what a PKGBUILD does before you use it.

Thanks to Arley Henostroza for providing these.

#### Ubuntu

There is a package for Ubuntu in the same repository as the Debian package, which makes use of the deviations file functionality to compensate for a different file location. An Ubuntu maintainer who will track and PR any further changes required and/or take over maintaining the Ubuntu package is sought.

#### Other

Currently, there are no installation packages available for other distributions/package-management systems. It is possible to download the genie.tar.gz file from the Releases page (essentially, the unwrapped Debian package) and manually place the files within, other than the DEBIAN folder, in the matching places and with the correct provisions, the edit the deviations file as necessary. Note that this is a packaged build, rather than a local-install build, and thus expects itself and hostess to both be installed under _/usr/bin_.

If you are able to build an install package for another distribution/package-management system, please consider adding your build process to the genie makefile in the same manner as the _debian_ target, and submitting a pull request. While not able to maintain packages myself for every distro, keeping the build processes for as many as possible in this repo would be useful to future genie users.

### ...OR BUILD IT YOURSELF

It is possible to build your own version of genie and install it locally. To do so, you will require _build-essential_ and _dotnet-sdk-3.1_ in addition to the other dependencies, all of which must be installed manually.

After cloning the repository, run

```
make install
```

This will build genie and install it under _/usr/local_ .

### Deviations File?

That would be the file _/usr/lib/genie/deviated-preverts.conf_ (or, on a local-install version, _/usr/local/lib/genie/deviated-preverts.conf_). Normally, it is an empty JSON file, to wit:

```
{
}
```

However, if essential binaries required by _genie_ are in different places on your system than the default location under Debian, the deviations file can be used to inform _genie_ of those locations. For example, to support Ubuntu 20.04 "Focal Fossa", which keeps daemonize in _bin_ rather than _sbin_, the following deviations file is used:

```
{
  "daemonize": "/usr/bin/daemonize"
}
```

Valid keys which can be used in the deviations file are: _daemonize_, _env_, _mount_, _nsenter_, _systemd_, and _unshare_.

## USAGE

```
genie:
  Handles transitions to the "bottle" namespace for systemd under WSL.

Usage:
  genie [options] [command]

Options:
  -v, --verbose <VERBOSE>    Display verbose progress messages
  --version                  Display version information

Commands:
  -i, --initialize           Initialize the bottle (if necessary) only.
  -s, --shell                Initialize the bottle (if necessary), and run a shell in it.
  -c, --command <COMMAND>    Initialize the bottle (if necessary), and run the specified command in it.
  --shutdown, -u             Shut down systemd and exit the bottle.
```

So, it has three modes, all of which will set up the bottle and run systemd in it if it isn't already running for simplicity of use.

_genie -i_ will set up the bottle - including changing the WSL hostname by suffixing -wsl, to distinguish it from the Windows host -  run systemd, and then exit. This is intended for use if you want services running all the time in the background, or to preinitialize things so you needn't worry about startup time later on, and for this purpose is ideally run from Task Scheduler on logon.

_genie -s_ runs your login shell inside the bottle; basically, Windows-side, _wsl genie -s_ is your substitute for just _wsl_ to get started, or for the shortcut you get to start a shell in the distro. It follows login semantics, and as such does not preserve the current working directory.

_genie -c [command]_ runs _command_ inside the bottle, then exits. The return code is the return code of the command. It follows sudo semantics, and so does preserve the cwd.

Meanwhile, _genie -u_ , run from outside the bottle, will shut down systemd cleanly and exit the bottle. This uses the _systemctl poweroff_ command to simulate a normal Linux system shutting down. It is suggested that this be used before shutting down Windows or restarting the Linux distribution to ensure a clean shutdown of systemd services.

While not compulsory, it is recommended that you shut down and restart the WSL distro before using genie again after you have used _genie -u_.

## RECOMMENDATIONS

Once you have this up and running, I suggest disabling via systemctl the _getty@tty1_ service (since logging on and using WSL is done via ptsen, not ttys).

Further tips on usage from other genie users can be found on the wiki for this repo.

## BUGS

1. This breaks _pstree_ and other _/proc_-walking tools that count on everything being a child of pid 1, because entering the namespace with a shell or other process leaves that process with a ppid of 0. To the best of my knowledge, I can't set the ppid of a process, and if I'm wrong about that, please send edification and pull requests to be gratefully accepted.

2. It is considerably clunkier than I'd like it to be, inasmuch as you have to invoke genie every time to get inside the bottle, either manually (replacing, for example, _wsl [command]_ with _wsl genie -c [command]_), or by using your own shortcut in place of the one WSL gives you for the distro, using which will put you _outside_ the bottle. Pull requests, etc.
