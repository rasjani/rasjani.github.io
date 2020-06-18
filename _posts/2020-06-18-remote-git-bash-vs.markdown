---
layout: post
title: Using Windows Git as remote visual studio environment
date: 2020-06-18 12:00:00
description: with openssh and bash as my default shell.
tags: [python,visualstudio,msvc,git,windows,ssh]

---

My professional & hobby background has always been flavoured with heavy use of posix based operating systems. While I'm not unfamiliar with Windows either, having a familiar familiar developer
experience in between different gigs is positive for my productivity.And these days, I guess even Microsoft acknowledges this as they are heavily backing up works like WSL, new
[Terminal](https://github.com/microsoft/terminal), Docker support and so forth.

But while things like WSL is a really good solution for some, for me it's overkill. My usecase for having access to Windows is 100%  related to compiling, related tooling and tasks around automating those into CI. In a nutshell, i need msvc, [git](https://gitforwindows.org/), [cmake/ctest](https://cmake.org/), [ninja](https://ninja-build.org/), [conan](https://conan.io/), [doxygen](https://www.doxygen.nl/), [python](https://www.python.org/) and what not. All these work just fine in command prompt in windows. Just open a new cmd window, run your required `vcvarsall` and `VsDevCmd` batch files and you are set. Using git on windows, you do have few options, you can use the cli on regular windows shell but there's also a fully working bash shell available that runs on MiTTY terminal.

As old \*nix geezer, I've accumulated quite a lot of little pieces helpful scripts, tools and configs that set's up the environment to suite my workflows. I can  set everything up as fast as git can
clone my files. My Vim configs along with all plugins are there, scripts to setup needed language servers, scripts to auto generate ctags, ssh configs and so forth.

### But something was still missing.

While I was already using git-bash as my main environment to edit files, i was still switching between it and cmd prompt because using msvc from terminal do not provide scripts for bash, only
PowerShell/Batch so i had to come up with a way to apply those changes to environment variables of bash instead of cmd. This part i didn't need to come up from scratch. In our CI, our job triggers
will inject properties file into spesific job's environment to set up msvc for different build type and architectures. This happens via something like this:

```batch
set | grep -i -E "(^lib=|^libpath=|^include=|^path=|^Windows(Sdk|Lib)|^Framework|^NetFxSdk|^DevEnvDir|^Platform=|^UniversalCRTSdk|^VC(IDE|INSTALL|Tools))|^VS(15|CMD|INSTALL)|VisualStudio|^SIGN" > build.env
:: replace \ with \\, Jenkins EnvInject plugin does not like unquoted \
sed -i s/\\/\\\\/g build.env
```

Since we have git-bash installed, we have access to helpful tools like `sed` and `grep`. First we get a list of all environment variables with `set` and grep the result to strip out only the
environment variables we need and pipe those into a new file. And just for jenkins, there's a need to replace `\\` with `\\\\` so that EnvInject plugin handles that file properly.

Instead of just grepping for a list of variables we already know are being set up by vcvars and vsdevcmd, we could also use a diff approach. Get before and after states of environment variables for
those 2 batch files and list only the changed lines. This would be better approach actually as then you don't need to maintain a list of variables like my example here does.


After we have acquired our `build.env` properties file, we need to process it a bit for using it in bash. File itself at this point looks something like this:

```
DevEnvDir=C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Professional\\Common7\\IDE\\
Framework40Version=v4.0
FrameworkDir=C:\\Windows\\Microsoft.NET\\Framework64\\
FrameworkDIR64=C:\\Windows\\Microsoft.NET\\Framework64
FrameworkVersion=v4.0.30319
FrameworkVersion64=v4.0.30319
INCLUDE= ... uninteresting list of directories ...
LIB= ... uninteresting list of directories ...
LIBPATH= ... uninteresting list of directories ...
NETFXSDKDir=C:\\Program Files (x86)\\Windows Kits\\NETFXSDK\\4.6.1\\
PATH= ... very interesting list of directories ...
Platform=x64
UniversalCRTSdkDir=C:\\Program Files (x86)\\Windows Kits\\10\\
VCIDEInstallDir=C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Professional\\Common7\\IDE\\VC\\
VCINSTALLDIR=C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Professional\\VC\\
VCToolsInstallDir=C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Professional\\VC\\Tools\\MSVC\\14.16.27023\\
VCToolsRedistDir=C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Professional\\VC\\Redist\\MSVC\\14.16.27012\\
VCToolsVersion=14.16.27023
VisualStudioVersion=15.0
VS150COMNTOOLS=C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Professional\\Common7\\Tools\\
VSCMD_ARG_app_plat=Desktop
VSCMD_ARG_HOST_ARCH=x64
VSCMD_ARG_TGT_ARCH=x64
VSCMD_VER=15.9.17
VSINSTALLDIR=C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Professional\\
WindowsLibPath=C:\\Program Files (x86)\\Windows Kits\\10\\UnionMetadata\\10.0.17763.0;C:\\Program Files (x86)\\Windows Kits\\10\\References\\10.0.17763.0
WindowsSdkBinPath=C:\\Program Files (x86)\\Windows Kits\\10\\bin\\
WindowsSdkDir=C:\\Program Files (x86)\\Windows Kits\\10\\
WindowsSDKLibVersion=10.0.17763.0\\
WindowsSdkVerBinPath=C:\\Program Files (x86)\\Windows Kits\\10\\bin\\10.0.17763.0\\
WindowsSDKVersion=10.0.17763.0\\
WindowsSDK_ExecutablePath_x64=C:\\Program Files (x86)\\Microsoft SDKs\\Windows\\v10.0A\\bin\\NETFX 4.6.1 Tools\\x64\\
WindowsSDK_ExecutablePath_x86=C:\\Program Files (x86)\\Microsoft SDKs\\Windows\\v10.0A\\bin\\NETFX 4.6.1 Tools\\
```

In order to get most of these to work in bash would just require throwing an `export ` in from of the line but there are few corner cases. First, values that have windows path separators, spaces need to be
properly quoted and second, PATH cannot be directly exported because paths need to be in format that bash will understand them. And obviously we dont want to overwrite the existing PATH in bash,
just add the missing entries.


### Tiny Piece Of Python

With help of few standard libraries, converting this properties file can be converted to a syntax that bash supports. Full program could look something like this:

```python
#!/usr/bin/env python
import argparse
import subprocess
import os
import shlex
import sys


def convert_to_posix_path(item):
    return subprocess.check_output(["cygpath", "-u", "-p", item]).decode("utf-8").strip()


def parse_args():
    parser = argparse.ArgumentParser()
    group = parser.add_mutually_exclusive_group()
    parser.add_argument("--file", "-f", type=str, default="build.env")
    return parser.parse_args()


def read_env(filename="build.env"):
    with open(filename, "r") as f:
        return list(filter(None, f.read().split("\n")))

params = parse_args()
current_path = os.environ["PATH"]
try:
    items = read_env(params.file)
except:
    print(f"Cant read {params.file}")
    sys.exit(1)


extra_paths = []
for item in items:
    key, val = item.split("=", 2)
    val = val.replace("\\\\", "\\")
    if key.upper() == "PATH":
        for d in val.split(";"):
            if d not in current_path:
                extra_paths.append(shlex.quote(convert_to_posix_path(d)))

        for foo in current_path.split(";"):
            extra_paths.append(shlex.quote(convert_to_posix_path(foo)))

        new_path = convert_to_posix_path(f"{';'.join(extra_paths)}")
        print(f"export PATH={new_path}")
    else:
        print(f"export {key}={shlex.quote(val)}")
```

This does the following:

* Parse the arguments passed to the app
* Read the current environment variables
* Read provided argument  `file` into items
* Split line for key/value pairs
* if property KEY name is PATH:
  * split the value with list separator `;` and iterate over each element.
  * If item from value is not currently in PATH, store it into `extra_paths` variable in posix format. Converting to posix is handled by cli tool called `cygpath` which is provided by git-bash itself.
  * Iterate over existing PATH environment and add each entry in posix format once again to `extra_paths` variable.
  * Finally, convert the final array again into posix format to make sure things really should work and write appropriate export line.
* else,  print properly quoted key, value pair.

This process will print out a set of "bash" commands that then will modify the current environment to be able to run msvc once those lines have been executed in bash. What I typically do is that once
i have `build.env`file, i run the above python script against it and store the result into separate file which i can just then load via:

```bash
python build_env.py  --file build.env > ~/.vs_settings.sh
source ~/.vs_settings.sh
```

### How about the remote part ?

Way before Microsoft wasn't embracing open source, getting ssh daemon up and running in windows wasn't that easy. But now, OpenSSH can be installed automatically if you are running on new enough
version. I think I followed the instructions [here](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)

Once you get it up and running, ssh session would still default to cmd or powershell but this can be easily fixed via following powershell script:

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Program Files\Git\bin\bash.exe" -PropertyType String -Force
```

And, lets try it out by ssh'ing into our windows desktop:

```bash
janimikkonen@Janis-MacBook-Pro ~$ ssh varjodesktop
Jani Mikkonen@DESKTOP-P3TPBJF ~$ uname -a
MINGW64_NT-10.0-18362 DESKTOP-P3TPBJF 3.0.7-338.x86_64 2019-11-21 23:07 UTC x86_64 Msys
Jani Mikkonen@DESKTOP-P3TPBJF ~$ source ~/.vs_settings.sh
Jani Mikkonen@DESKTOP-P3TPBJF ~$ which cmake && which ninja && which cl
/c/Program Files (x86)/Microsoft Visual Studio/2017/Professional/Common7/IDE/CommonExtensions/Microsoft/CMake/CMake/bin/cmake
/c/Users/Jani Mikkonen/bin/ninja
/c/Program Files (x86)/Microsoft Visual Studio/2017/Professional/VC/Tools/MSVC/14.16.27023/bin/HostX64/x64/cl
Jani Mikkonen@DESKTOP-P3TPBJF ~$ exit
```

Now, i can just ssh into my work machine, share the same environment (dot files) as my mac and linux boxes and feel at home without adding an overhead of running virtual machines.. Pure bash experience in
Windows!


### Few Gotches and pointers ..
* This approach would also work with cygwin - I use cygwin in CI, mainly because we can also then install extra tools with cygwin's setup, as git's msys based distro doesn't ship with package manager.
* With that in mind, if you want to run python, having python installed inside cygwin is not best option if you want proper windows support.
* Git's vim, while new enough doesnt work with (mac's?) cursor keys. Cygwin's does - but thats why there's H,J,K,L keys ;)
* If you need to run repl's (like python) on the shell and you wish to use readline capabilities, repl has to be executed with tool called "winpty". It ships with git-bash and can be
  compiled/downloadaed from [github](https://github.com/rprichard/winpty/releases)
* Don't expect very responsive screen updates if you are running heavy cpu & i/o loads.
* It should be easy to extend the python script to also have "deactivate" feature if so needed. Its however easy to start another bash from the main, load the settings with `source` and deactivation
  is just running "exit".
