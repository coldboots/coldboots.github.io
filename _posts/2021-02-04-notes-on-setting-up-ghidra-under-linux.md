---
layout: post
title: Notes on setting up Ghidra under Linux
date: 2021-02-04 13:06 +0100
tags: ghidra re tools
author: larsw
---

## Background

I've used Ghidra for a couple of months, but it is just recently I discovered a couple of things that you want to know
about if you're using it under Linux.

## PUBLIC or DEV?

Your mileage my vary - it might be wise to choose the PUBLIC version if you're not interested in checking out bleeding edge features,
like the upcoming debugger.

### Installing PUBLIC

1. Pull the latest .zip file from the official site, https://ghidra-sre.org/ (v9.2.2 as of time of writing)
2. Unzip and move to `/opt/ghidra`

### Building DEV

The following are basically just an compressed version of the README in the repo. You don't need to run `docker` with `sudo`
if you're a member of the `docker` group.

0. Have Docker installed.
1. Clone [https://github.com/dukebarman/ghidra-builder.git](https://github.com/dukebarman/ghidra-builder.git) and cd into it.
2. `sudo docker-tpl/build`
3. `cd workdir && sudo ../docker-tpl/run ./build_ghidra.sh`
4. Upon completion, the `out` folder will contain a .zip of the installation.
5. See 1. and 2. under _Installing PUBLIC_

If you want to build another branch, like `debugger`, `cd` into the `workdir/ghidra` and check out the appropriate branch,
before repeating step 3-5 above.

### Creating a proper desktop item/icon

It is possible to run Ghidra from a terminal window by invoking `/opt/ghidra/ghidraRun.sh`, but note that this has
a shortcoming that isn't intuitive at all; keyboard shortcuts won't work!

Because of this, you will probably want to create a `Ghidra.desktop` file and assign an icon.

#### Download a proper .ico file

See this comment on the following GitHub issue:
https://github.com/NationalSecurityAgency/ghidra/issues/463#issuecomment-554611410

Download the attached file, and place the icon from the zip-file into `/opt/ghidra/support`.

#### Author a .desktop file

Create a file named `Ghidra.desktop` in:

```
[Desktop Entry]
Categories=Application;Development;
Comment[en_US]=Ghidra Software Reverse Engineering Suite
Comment=Ghidra Software Reverse Engineering Suite
Exec=/opt/ghidra/ghidraRun
GenericName[en_US]=Ghidra Software Reverse Engineering Suite
GenericName=Ghidra Software Reverse Engineering Suite
Icon=/opt/ghidra/support/ghidra.ico
MimeType=
Name[en_US]=Ghidra
Name=Ghidra
Path=/opt/ghidra
StartupNotify=false
Terminal=false
TerminalOptions=
Type=Application
Version=1.0
X-DBUS-ServiceName=
X-DBUS-StartupType=none
X-KDE-SubstituteUID=false
X-KDE-Username=
```

You can now find Ghidra under _Show Applications_ and add it to the task bar/Favorites.

### Applying a dark theme

If you're like me, you probably prefer that your developer tools comes with a dark theme.
Out of the box, Ghidra does not :-/

Luckily, someone made a dark theme that covers about 97% of the UX, which is good enough.

0. It might be a good idea to do this before you move everything to `/opt/ghidra`.
1. Clone the https://github.com/zackelia/ghidra-dark.git and cd into it.
2. Be sure to close all running Ghidra instances.
3. If you've built Ghidra yourself, you probably need to patch the `install.py` before running it:

```diff
@@ -92,7 +92,7 @@ if not flatlaf_set:
 # _PUBLIC was appended to the name after 9.0.4
 # The "-" after .ghidra was changed to "_" after 9.0.4
 if tuple(map(int, (version.split(".")))) > (9, 0, 4):
-    version_path = f".ghidra_{version}_PUBLIC"
+    version_path = f".ghidra_{version}_DEV"
 else:
     version_path = f".ghidra-{version}"
```

Change the `version_path` according to the name of your installation.

4. Invoke `python install.py --path=/path-to-ghidra`

Note that you will need to repeat this process if you download or build a new version.


Enjoy your reverse engineering with Ghidra!

@larsw
