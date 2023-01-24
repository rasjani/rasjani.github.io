---
layout: post
title: (More) Stuff you should know about Python part 2
date: 2023-01-24 12:00:00
description: pip, packages and python
tags: [python, pip, wheel, packages]

---
In previous [post](/2023/01/18/stuff_you_should_know_about_python.html) I was talking about python and "paths" related to how to avoid problems when installing python packages. On this one, I'm take a stab on installing packages and what sort of problems one might face when installing packages with standard python.

## Packages in Python.

There are few main ways to package python code for others to use/install; source distribution and build distributions aka "wheel". Source distribution. Source distribution is essentially compressed archive of the source tree with some instructions on what files go where when its being installed. If the package depends on some code that is not "pure python" eg, package relies on libraries written with for example in C++, package may include instructions on how to compile those external dependencies into format that is supposed to work with the particular python version. With source distributions, this compilation steps happens in the device where the package is being installed **to**. This makes it a "hard dependency" to have working compiler in the system that can produce working binaries for any non-python code that's shipped within the package.

Now, wheel packages on the other hand will not have this compilation step *if* they use external libraries because maintainer of the package is responsible to make that compilation step before submitting the package to package registry like [PyPI.org](https://pypi.org).

### Source Distribution

By default of pip's functionality, installing a package with it will use source distribution if certain pre-conditions are not met. If the package contains non-python code, user **must** have working compiler environment installed. On Linux, gcc is installed most of the time, MacOS does provide XCode.  In both of these environments, installing the compiler shouldn't take more than few mouse clicks (and accepting the EULA on mac via command line). However, on windows one needs Microsoft Visual Studio - which is a paid product but Microsoft does provide free versions. For more details on how to install MSVC, wiki at python.org has [instructions](https://wiki.python.org/moin/WindowsCompilers).

Sometimes package maintainers will not provide all the dependencies that are **required** to build the particular package for the platform. Typically these external dependencies are documented within project's README and/or projects pypi page. This means;  careless installation of the package can lead to issues if the system where package is being installed to does not meet the requirements.

### Binary Distribution

Since "wheels" are already compiled, user won't need a fully working compiler. However, wheels might have other issues. As said earlier,  `pin install $package` might still default to installing a source distribution. For example, if virtual environment is used, one has to make sure a package called `wheel` is installed before attemting to install other dependencies. My personal take on this is to create a venv with shell alias or small script that does following:

```
python3 -mvenv venv && source venv/bin/activate && python -mpip install --upgrade setuptools pip wheel
```

This incation will create a virtual environment, activates it and updates following modules: pip, setuptools & wheel.

Next, as mentioned above, wheels are "already compiled". But there can be "pure python" wheels that do not contain any native code and then there's wheels that do. If the package does indeed contain native code, maintainer should provide binaries for each target platform and python version that he/she is willing to support. For example, at the time of writing this, python 3.7 is still being actively supported and there have been 4 other major versionsof python since then. Totals to 5. Next, a binary wheel needs to be compiled for the operating system **and** cpu architecture the OS is running on. Let's say a package maintainer wants the package to be working and  installable only in MacOS. MacOS now runs on Apple's own CPU architecture (arm) and older devices use Intel CPU. Supporting just MacOS with pre-packaged/compiled wheels would mean that the maintainer needs to provide wheel for each python version *and* architecture, resulting in 10 different versions.  Total of 10 wheels need to be provided. Adding linux to the mix, with various platforms it can run and where python is used can easily double or triple the amount of packages.  Ofcourse free CI/CD services like github actions and azure devops will make this possible but sometimes its just not worth the hassle. Let's take `numpy` as example. At the time of writing this post, latest numpy version is 1.24.1 and released files that pip can install are listed [here](https://pypi.org/project/numpy/#files). There's total of 27 wheels provided by numpy project.

Now, what if new major release of python appears - and you install the latest and greatest because you want to live on the edge - and rely on external packages ? there is *always* a time in between the release of new python version *and* release of the wheel *for* that particular version, if the project as direct dependency or dependency of dependency has not not provided a working wheel for your os/cpu architecture and python version, pip will fall back to using source distribution. Similar issue happened when Apple made the switch to its own cpu architecture, plenty of packages where in various states of despair because pypy didnt distinquish between Apple Silion and Intel binaries ...

So, never take it for granted that your environment will always work without considering what versions you really need and don't take it for granted that package maintainers will support each and every combination. And be prepared to actually be able to compile software if/when necessary. 


---
