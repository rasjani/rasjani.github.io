---
layout: post
title: Benefits of using virtual environments
date: 2020-02-01 12:00:00
description: There are benefits that might not be obvious at first for using virtual environments with python. Here's some points that might convince you to use them too.
tags: [python, ci, venv, virtualenvironments]

---

First off, what's virtual environment in python land ? In a nutshell, it is a isolated "container" of all your python code
including all of your own code and external dependencies tied into a specific python interpreter. 

Why would one need then have a need to "contain" their code ? There's plenty of good reasons why you should and here's few reasons.

When setting up CI environment, it typically has tools like test libraries, runners, reporters and other tools that glue things together
or help up in test environment setups or validation. Initial thought would be to install all these dependencies directly to what ever python is
installed on the system so that everyone can benefit of not having the need to install everything. It's a working solution up until the point.

What happens if, you need a library to be installed in your test environment but you don't have access to write to default locations ? 
Need to support multiple versions of python ? What if you have tools or code that rely on the same dependency but for different versions?
What if you install a package as user A and that package can't be found when user B is using the same system? Or scenario where someone updates 
a package on the system and that causes regression for someone else.  Or how about when your project starts to grow and you suddenly have a need 
to scale up your test environment ...

Lets go thru these.

## Directory access

Typically when python is installed on any system, it is installed via root or admin account and into file system locations that are typically
locked down for write access for that account or group only. For example, in Linux or mac, you just can't install packages via pip without `sudo`
or `--user`. With `sudo`, that library is then found by every user account in that system if they are using the same python interpreter but 
`--user` flag installed packages are only available for that particular user only.  

On the other hand, when you use virtual environment, you decide where your code and dependencies gets installed so you don't need to worry about 
access.

## Multiple Python's 

One can have multiple python interpreters installed and working on a single system but using them requires either that you juggle with your
`PATH`, `PYTHONHOME` and `PYTHONPATH` or any combination of them in order to use different versions. Using absolute paths will help. You
could just call the python interpreter or by utilizing shebang. Both of these might cause issues later on, for example if you have a new system
where location of python is not the same as in other systems. Virtual environments provide that PATH juggling for you automatically: it 
essentially creates an isolated environment where "python" command points into exact python version that was used to create the environment.
Worth to mention that even in this case, you should be implicit about which python so using absolute paths might still be required but that
is required only when you create the environment. 

## Dependencies and more dependencies.

Dependencies can cause multiple issues. Package A might have pinned its dependency against C into a single version or a range of versions. When
you then need a package B, it might have also pinned the version of dependency C but to a different one. Package B then might end up in state that
C is there but it might cause issues due to compatibility. Or maybe package B might pull in newer C and A wasn't explicit what it requirees so A 
might start to malfunction. Of course if you only have one project, you will be vetting those dependencies and their interoperability when you
are developing the project but if you share your CI with other project, you can end up with problems like this. On the other hand, if and when
take a bit of time and pin all of your dependencies into something like `requirements.txt`, install those explicit versions and those versions 
only into virtual environment, you are not affecting any other environments or system level python libraries and you have a reproducible 
environment every time you create a new or activate existing one. 

## Scaling up

And the final point. Your test or CI  run starts to take long and you need to split things up or just run the same set on multiple slaves. You get
a new hardware or virtual machine and you setup it up. With VM's you actually might already have a tooling to set it up according to standard
requirements from the start but in more "general purpose" guests, it still makes sense to have the job to always enforce its own. This acts 
also acts as a general documentation but also helps out to offload image customization when you need to run same test assets on different
operating systems.

With this approach, either in your base image or installation checklist, you just need to make sure the basic stuff is available and projects 
take care of their own dependencies at runtime. Thus, adding a new slave online is just hooking it up to a network and configuring 
your CI to use it.

## Downsides ? 

Well, you don't share the "global state" anymore between all users in that particular system. It could be a drag that installation takes valueable
time during CI job execution but imho, the benefits outshine the downsides.

## Afterthoughts

One benefit that i didn't mention is about house keeping. When you are experimenting with what ever libraries, you install them and they most
likely will pull up their own set of dependencies and they again their own and so forth. Now, if you where to install all of those into global 
environment you really soon have hundreds of packages installed and you don't know why they are there or are they needed by anything. With 
virtual environments, cleaning up thing is just removing a single directory. 

If this raised any interest on using virtual envs, do check RealPython's [Virtual Environments Primer](https://realpython.com/python-virtual-environments-a-primer/) 
or this shorter introduction of python3's standard [venv](https://docs.python.org/3/library/venv.html) module [here](https://cewing.github.io/training.python_web/html/presentations/venv_intro.html)
