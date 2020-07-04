---
layout: post
title: How I use Invoke on few of my projects
date: 2020-07-04 12:00:00
description: to help with the repetitive tasks during coding/test/deployment cycle.
tags: [python, inv, make, build]

---

I was listening to latest [Python Bytes](https://pythonbytes.fm/episodes/show/188/will-the-be-a-switch-in-python-the-language) podcast and Brian talked about [Invoke](https://www.pyinvoke.org/) project. Inspired by it i thought I'd make a post about how I've been using it on my own projects.

I have few open-source software projects that i develop and maintain, [SeleniumTestability](https://github.com/rasjani/robotframework-seleniumtestability) being the major one. Landing a new piece of code for general consumption there typically involves any combination of following steps:

 * Write code
 * Write tests for the code
 * Install python dependencies
 * Install webdrivers required during acceptance tests
 * Create a working build.[^1]
 * Run acceptance tests against multiple browsers
 * Collect coverage information from tests
 * Format code
 * Generate documentation
 * Run static analysis against the python codebase.
 * Run type checker against the python code
 * Run static analysis against the acceptance tests
 * Clean the source directory of unwanted cruft

Once I'm happy how things look and code has been merged into repository and its time to make a release:

 * Run acceptance tests again
 * Review and generate technical documentation and project readme files
 * Generate changelog
 * Add all modified files so far into release commit and make a nice commit message
 * Tag the release in the repository with new version number
 * Generate pip package with assets of current python code and other build artifacts.
 * Publish the package to pypi.org
 * Publish the release data to projects GitHub page.

Lots of these tasks gets repetitive really fast. Different tasks require a slightly different parameters at different stages and what not. This adds up things you have to memorise if you wish to make all these steps manually every time.  Enter Invoke!

### Invoke

Invoke is in short: command line tool and framework to write and run predefined tasks that you can write as simple python methods. Something like ruby's rake tool, or even plain old make[^2].

Pretty much all of my tasks are calls to external tools. They might require some somewhat dynamic parametrisation. Some of the tools might not do exactly what so i have ability to also  modify the behaviour and rest are just of same old stuff.

I start by installing invoke into a [virtual environment]({% post_url 2020-02-01-benefits-of-venvs %}), create a file called tasks.py and add a simple task:

```python
from invoke import task
@task
def staticanalysis(ctx):
  ctx.run("flake8")

```

At this point, i do ofcourse have flake8 configured in my project's `setup.cfg` so i get the warnings and errors i care about. Then in order to run this, all i need to do in the shell is execute a command `inv staticanalysis`. Quite a few of my tasks are simple calls in similar fashion. I can call mypy, black, pylint to give my feedback and format code and/or chain them into a single task. I can call [webdrivermanager](github.com/rasjani/webdrivermanager), npm to install what ever dependencies i need. I can add tasks to run robot for acceptance tests and so forth.

This example was  pretty simple so lets move to a bit more complicated use-case to give some further ideas.

### Make invoke work for you

I want to have some sort of changelogs for every release to inform my users what has changed since previous release. I did a lot of research to find a suitable tool to generate a changelog that includes the information and  the aesthetics that i prefer.

I evaluated quite a few tools to do this, none of them really fit my needs perfectly. I did end up using a python project [gcg](https://github.com/nokia/git-changelog-generator), git-changelog-generator which has been open-sourced by Nokia. It fit almost perfectly my bill but it lacked one feature:  Sometimes there are commits in the repository which do not affect the endusers of the project at all. Lets say, i need to update details in my CI pipeline of what webdriver version is needed in the CI. That hardly is of any interest to anyone except the system where my tests are executed. gcg lacked a feature to filter out these sort of cases.

In scenario where i know i don't need to add the commit into next releases changelog, i append a string !nocl into git commit's subject line. I call gcg to generate a new changelog, it has all these git commit subject lines listed in it so it because easy to filter out these lines before the changelog actually gets committed into the repo.  Also, in order to generate a proper release changelog, i need to pass the exact version to gcg so that knows what will be the latest release.  This way, i can pre-generate the changelog into a commit that will be later on tagged with that given version.  So, let's see what i got:

```python
CHANGELOG = "CHANGELOG"
@task
def changelog(ctx, version=None):
    if version is not None:
        version = f"-c {version}"
    else:
        version = ""
    ctx.run(f"gcg -x -o {CHANGELOG} -O rpm {version}")
    filter_entries(CHANGELOG)
```

Here i have "changelog" task that takes optional parameter called version. If it's provided (eg, not None), i know i need to append `-c` flag to gcg with that version string. Then i call the actual gcg tool, i pass it the name of the file it should generate and optional versio string which could also be just an empty string .. And finally, i call python function  `filter_entries` and pass the filename of changelog gcg just generated

```python
filters = ["poc", "new release", "wip", "cleanup", "!nocl"]
def filter_entries(filename):
    buffer = []
    with open(filename) as old_file:
        buffer = old_file.read().split("\n")

    with open(filename, "w") as new_file:
        for line in buffer:
            if not any(bad_word in line.lower() for bad_word in filters):
                new_file.write(line + "\n")
```

This function essentially reads my changelog into a list, iterates over all of those lines and filters out all lines that contain any of the strings defined in `filters`-variable and finally writes everything back to the original file.

And voilã! Now i have a task in my environment that generates a changelog as i want it to be. I can call this task anytime and see how things look like. Or i could call this changelog task from another yet an another task and pass a version string along. This can be seen in a `release` task for example:

```python
@task
def release(ctx, version=None):
    assert version != None
    changelog(ctx, version)
    docs(ctx)
    ctx.run(f"git add docs{os.path.sep}* {CHANGELOG}")
    ctx.run(f"git commit -m {QUOTE}New Release {version}{QUOTE}")
    ctx.run(f"git tag {version}")
    build(ctx)
```

Yet again, another task. First it verifies that version was indeed passed in into it with `assert`. Then generate a changelog and docs, add all the files I'm expecting to change, commit and tag the release with given version string and generate a publishable python package.

As added bonus, once you have your "release" tasks defined and working, when you want to add continuous integration, testing and delivery, you already have all the tasks "scripted" so your pipeline steps can be very simple and standardized into predefined steps that then should working across different devices and operating systems without extra work. That is, if you took care of writing your tasks in cross platform manner. Lets take an example of how a single task can be helpful in CI and on local environment while writing code:

```python
@task
def test(ctx,coverage=False,xunit=None, skipci=False,outputdir=f"output{os.path.sep}",tests=None):
    """Runs robot acceptance tests"""
    extras = ""
    if coverage:
        ctx.run("coverage erase")
    cmd = "python"
    extra = "--non-critical skipheadless"
    if xunit:
        xunit = f"--xunit {xunit}"
    else:
        xunit = ""
    if coverage:
        cmd = "coverage run"
    if tests is None:
        tests = "atest"
    if skipci:
        extras = f"{extras} --exclude skipci"

    ctx.run(
        f"{cmd} -m robot --pythonpath src --outputdir {outputdir} --loglevel TRACE:TRACE {extras} {xunit} {tests}"
    )
```

My test tasks takes in quite a few parameters that are then used to construct a call to run my acceptance tests.

* coverage - should this run generate test coverage data or not. If its provided, i first clean up the previous coverage data and change the command from `python` to `coverage run`.
* xunit - should the tests generate report in xunit format. If it is defined, i add `--xunit` with the parameter file which indicates the filename where report should be written to. This is handy as pretty much none of the cloud based ci services do not support robot framework's native reports.
* tests - if defined, it contains a list of robot framework test suite filenames. Handy way to scope down what is actually executed when working on only small area of the code.  At the moment, i think SeleniumTestability's full acceptance run takes something like 10 minutes.
* outputdir - defines a location where robot framework should store all of its reports and logs. While developing locally, this doesn't give so much benefit but in CI it does. In my use case, i run my scripts with combination of different python interpreter versions and operating systems. All of those "parameters" are defined in the ci pipeline declaration and are accessible during execution and their combination is unique to that give build.  Easy way to ensure that i can gather all the reports from all the runs and that there are no conflicting files/directories betwe
* skipci - provides a flag that can toggle what tests should be executed. For example, in SeleniumTestability there's a feature that scrolls view-port up and done and the plugin should react to those scrolling events. However, it seems that both Firefox and Chrome when executed as "headless" do trigger scroll start and scroll end events but scrolling is way faster in headless mode than when running a full screen browser. Thus, the testcase might become "flaky" when running in headless-mode, which is what I have in my pipeline. I tag the tests in my acceptance tests so that its easy exclude those flaky tests CI and which do not and thus.

Finally, i call robot with combination of given parameters and it caters to my needs locally and within the ci pipeline.



[^1]:SeleeniumTestability consists of python code that that relies on javascript code running inside a browser. In order to avoid "vendoring" 3rd party javascript code, i download my dependencies with npm and build the final js bundle with webpack.
[^2]:Make however does have a benefit that it can detect if the stop should be run or not which invoke does not
