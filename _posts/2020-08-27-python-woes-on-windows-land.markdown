---
layout: post
title: Python Woes On Windows Land
date: 2020-08-27 12:00:00
description: Using python for build engineer in Windows
tags: [python,visualstudio,msvc,windows,conan,pathlib,subprocess, jenkins]

---

## No I'm Not Doing Cross Compilation.

Let's start with the Jenkins side of things. my build and test nodes in windows far are currently set up to start via SSH that is running from within Cygwin. There are few reasons, first, during the
years I've noticed that its somewhat more reliable way to start Jenkins node via it compared to either via windows service or via JNLP. Also, because cygwin allows starting the daemon as user process,
not as a service, child processes are then able to run test applications as interacive applications where as services are not allowed to do that without somewhat cumbersome setup.

This sort of approach has been working well up until we wanted to use Ninja to build Conan recipes. What was weird that things worked fine on local developer machines but when triggering the same
exact build on Jenkins, conan failed miserable. Issue looked like that when using Ninja generator to CMake(), conan somehow thought the platform was not detected properly and the environment it is
running should do cross compilation. Spend too much time to try and debug this but nothing came up. Completely by bad, i should have had a peek into the standard libraries.

Que few months later, I'm in process of modifying our conan recipes so that i can build them on multiple architectures and operating systems and since standard library has this module, platform and
uname() method which returns all relevant info about the platform its running .. Write code, works, make a pull request and see it fail miserably in the CI, WTF?!

Code was essentially something like this:

```python
import platform
data = platform.uname()
config_key = f"{data.system}-{data.machine}"
```

And config_key always ended up being just "Windows-". WTF once again! So, lets take a peek what is actually happening. platform.uname() ends up eventually calling
https://github.com/python/cpython/blob/master/Lib/platform.py#L842 this piece of code, which turns out to be something along these lines:

```python
os.environ.get('PROCESSOR_ARCHITEW6432', '') or os.environ.get('PROCESSOR_ARCHITECTURE', '')
```

Oh? Im missing, these either one of these variables.. Fire up the kvm, open shell, run a small piece of code and it works! But not when jenkins triggers it, or in the bash shell invoked via ssh from
Cygwin. Bit of googling and ended up http://smithii.com/node/44 but that did not help with build running from jenkins, only the ssh connection directly to the box. Dammit!

Well, after a bit of fiddling around, I ended up just setting up a per slave environment variable in node configs that set up %PROCESSOR_ARCHITECTURE% env variable to AMD64 and fixed the Conan issue
with Ninja generator. As alternative, per node configuration does have option under "Prepare jobs environment" called "Unset System Environment Variables". That most likely would have helped also but
I was bit hesitant because that might inject other variables which might cause other issues..

## WindowsPath is not iterable.

During my time at current position, I've been refactoring a bunch of Robot Framework libraries and build tooling to use Pathlib instead of plain old os.path and alike standard api's. We have bunch of
tools and tests which are triggered from python code and in order to make those scripts work nicely, eg, construct dynamic paths so that they work on developer machines but also in CI where path
structure is completely different, using pathlib is really nice.

My usual work environment is MacOS and i do run same python version as our ci boxes so imagine my suprise when i had a code like this:

```python
from pathlib import Path
import os
import subprocess

dir_name = os.environ.get("WORKSPACE", None) or "."
output_directory = Path(dir_name) / "results"
res = subprocess.run(["mytest", "--output", output_directory])
```

And it throws an exception: TypeError: argument of type 'WindowsPath' is not iterable.

WTF?! Again?!

I try this on my Mac, on older and newer python releases that have pathlib support, 0 issues. Buddy of mine tries to run the same piece of code on Linux, with different versions of python. It just
works.

So, whats happening ? Stacktrace points to line 555 in my current python version but on the latest cpython up on github, its this line https://github.com/python/cpython/blob/master/Lib/subprocess.py#L568

That line checks if a single argument contains space or tab character and thus, it would then need to be quoted.

Lets check if this would work on Mac or Linux with following:

```python
from pathlib import Path
x = Path("/tmp")
" " in x
```

Nope, that throws same error except wit PosixPath as type of x. I didn't dig up further if linux/mac would even have similar "needquote" functionality but in both of those environments, it is still
safe to use pathlib types are command line parameters. Either they are automatically casted to str's or there is no check like this. This feels definitely like a bug in CPython implementation because
now i need to explicitly cast my path objects into strings before passing them to subprocess api's.

=(


