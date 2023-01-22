---
layout: post
title: Stuff you should know about Python part 1
date: 2023-01-18 12:00:00
description: paths, pip and python installations
tags: [python, pip]

---

Due to various reasons, I've had some spare time to scroll thru and reply to plenty of Stackoverflow's [python](https://stackoverflow.com/questions/tagged/python?tab=Newest) questions. One of those questions that keep on getting asked again and again is "hey, I installed this package, why it is not working". While there seems to be plenty of similar FAQ type of questions, let me start by addressing this particular one.

## Talking about PATH's

When you type `command` into your terminal / shell prompt - the shell actually needs to look *where* that particular `command` resides in your filesystem. To assist with this process, pretty much all the shell's I've ever used has a setting - environment variable - to be exact called `PATH`. Bash, Zsh, Fish has it, cmd has it, pretty sure Powershell has it too.

So, lets say you want to run python interpreter in interactive mode, you should type `python` into your shell and hit enter and it should just work.  Except on windows it might not - Why ? Microsoft on at least windows 10 ships a binary called "python" in location that seems to be in PATH that is not "real" python but just a tool that prints out a message that "you can now install python from windows store". And depending on how/where you installed your python from, you might still have that MS wrapper in the path. Specifically so if something went wrong with the installation and PATH environment of the installed python was not placed before the path where the "fake" one is. That is because path searching will start from first directory mentioned and stops searching once first occurrence is found and then runs that. 

Depending on your operating system there are few options to verify the exact location of the python itself. In posix like environments like Linux or MacOS you can use  `which python` and similar functionality in windows is `where python`. These commands will print the absolute path where the first python command is found.

Now, as a side note, if you are not familiar with shells work and even after installation of python, you are not able to run it or which/where can't find it - keep in mind that modifying shell's environment variable at one shell will not affect other instances. For example, in windows, if you had "cmd" running and installed python via official installer - that existing shell will not be able to see it before it is restarted. On similar fashion on unixy land, certain shells can keep a cache of all the commands available in the cache. In bash you would update that cache with `hash -r` and `rehash` in zsh.

## Talking about multiple python interpreters

In previous chapter I mentioned that windows ships a binary called "python". But what if your OS does have that too ? At least macOS used to ship with python 2.7 which was in the path just as "python" along with also "pip". So, you where starting in that environment within recent years (before Apple removed old python version) - you where definitely going to have issues installing 3rd party dependencies due to compatibility issues. On same fashion, plenty of older linux distributions where relying on python2.7 due to installation scripts and internal tooling where written with it and updating to  python 3 would have caused these scripts/ libraries to not work. So, how was this handled ? Enter "python3". Instead of just running "python", newer version of the python was/is actually python3.

Now, each actually working python does have a default set of directories where it looks for anything that is imported into the running interpreter. If you want to take a deep dive into this topic, take a look at this RealPython.com about modole search path [here](https://realpython.com/lessons/module-search-path/).  However, lets keep things on "shell" level and lets take older macOS as example about using the correct interpreter.

Running `pip` on terminal actually executed "default" python which was python2.7.  Packages could be installed but once you run any other python version, those packages are not found. This would happen if you update the python *or* use any different version. similarly[D[D[D[D[D[D[D[S in Linux land, plenty of distributions had "python", "python3" executables and "python-pip" and "python3-pip" packages. If installed pip package was not matching the python version you are trying to use, you will end up having failures when running any code that tries to import 3rd party dependencies.

On posix like land, shebang mechanism could add extra layer of issues. Shebang is defined in a script, on first row of that script to define the "interpreter" that should be used to run the rest of the script file.

Here's few examples:

1. #!/usr/bin/env python
1. #!/usr/bin/python
1. #!/opt/homebrew/opt/python@3.10/bin/python3.10

Lets go thru the differences. First example will use utility called `env` to search the current PATH entries for executable called `python`. Now, what if you have multiple python executables in the path ? You will be blindly installing packages to what ever python version that happened to be first one in the path. Secondly, as mentioned earlier, in same cases `python` != `python3`

Second and third example use absolute path to exact python executable but that also means, you or your system administrator has to make sure that the explicit /usr/bin/python is the correct version needed to run your code.  However, third example uses absolute path to absolute python version and when ever you run this exact binary, you are making sure that all the other calls to the same interpreter will be that, same interpreter

## Here comes pip.

I've now covered few scenarios where executing pip can lead to packages not being available for your particular python version. pip can be for python2, or python3, or the actual command could be pip3. Shebangs could point either of those to a wrong python version (possibility but very minimal). So how to be 100% that you are using correct pip for your chosen python version ?  Use `python -mpip` or `python3 -mpip` instead of `pip` or `pip3`. For example, you want to install requests library, you would issue following command `python3 -mpip install requests`.  Now, if you happen to have multiple python interpreters by your own choice, you should call that particular version with **absolute path**  like `/opt/homebrew/opt/python@3.10/bin/python3.10 -mpip install requests`. 

## Final note

If possible, **always use virtual environments**

If you are not familiar with virtual environments - I've written few words regarding their use [here](/2020/02/01/benefits-of-venvs.html). 

Once you create and activate virtual environment, activation script modifies the active shell session in a way that tools like python interpreter and pip will always be first in the path. Also, at least in macOS, if you create the virtual environment with `python3` - after the activation of said environment, `pip3` and `python3` will become `pip` and `python` thus, can avoid other issues described above. 


---
