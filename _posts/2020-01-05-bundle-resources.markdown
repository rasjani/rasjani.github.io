---
layout: post
title: Bundle RobotFramework resource files into packages.
date: 2020-01-05 12:00:00
description: People have been talking about how to share your robot resource files via packages. I took a stab at it and made a proof of concept out of the idea.
tags: [robotframework, python]

---

So there has been people in rf slack channels asking about the possibility to ship resource files by some mechanism other than bundling them with the rest of the test data. Often that indeed is a good
alternative. But consider the scenario where you have a helpful keyword library written as `yourlib.resource`. This file may reside in another repository but it would be nice to get that available for
ci or development environment automatically ? If your test libraries where written in python, this would be rather easy. Just whip up pip, install the package and import it into your testsuite with 

```robotframework
Library   YourPackage
```

But when your package only consists only of robot resources, you can't just import those without some hurdles. Now, there is [Import
Resource](https://robotframework.org/robotframework/latest/libraries/BuiltIn.html#Import%20Resource) - keyword that you could use to load resource files like this: 

```robotframework
*** Test Cases ***
Importing mylib.resource 
  Import Resource   mylib.resource
  This Keyword Implemented In MyLib
```

Downside of this is that you clutter up your test code with stuff that is unrelated to the actual workflow. And this doesn't really solve the issue of distributing those resource files either. You would have to know the full path to the resource file to load it and that might change from one system to another - You might be writing the tests on your linux box and test environment is windows for example, Or you run Jenkins, workspace directory might not always be the same if same node runs same job in parallel. And many other possible scenarios.

### Where to go from here ? 

Most of the time, if you run python somewhere, you have access to pip. Adding arbituary files into packages doesnt take much effort after you can produce a barebones package.
Then uploading that to either your interal package archive or to public pypi. I'm not going into details on how to do this packaging but i made a small example to get the gist of it
[here](https://github.com/rasjani/robotframework-importresource-testdata) - now we just need to have a way to load these packaged files into robot when you run the tests.

We do have a mechanism to load resources from known locations with `Import Resource` keyword inside robot code or from python side, another mechanism to ship files within packages. The only thing we dont have is a mechanism to locate where those files end up in the filesystem and the "how" we can load those in the testsuite with minimal code clutter. 

Lets play out some ideas. Ways to extend RF is writing python libraries and if i would have a library like that, all it would need to know is what package names it should scan and import resources from. 

Python standard library has module [pkgutil](https://docs.python.org/3/library/pkgutil.html) and API [iter_modules()](https://docs.python.org/3/library/pkgutil.html#pkgutil.iter_modules) we can use to get a list of all packages in the system (works fine with virtual environments too). That API will give as the base location of a package and it's name. With those, its easy to construct a full path of the module and identify the module by its name. At this point, we can get a list of all files with Pathlib's [rglob](https://docs.python.org/3/library/pathlib.html#pathlib.Path.rglob) within that path and make educated guesses on which ones we might want to load in.

That would look something like this:

```python
    def __init__(self, resources):
        self.resources = []
        try:
            self.rf = BuiltIn()
        except RobotNotRunningError:
            pass
        self.modules = self._find_modules()
        for resource in resources.split(";"):
            if resource in self.modules:
                resource_files = self._find_resources(self.modules[resource])
                if resource_files:
                    for resource_file in resource_files:
                        try:
                            self.rf.import_resource(resource_file)
                        except RobotNotRunningError:
                            pass
                        self.resources.append(resource_file)
                else:
                    logger.warn(f"Module {resource} did't contain any resource files")
            else:
                logger.warn(f"Module {resource} doesn't contain resource directory: {self.RESOURCE_PATH}")
```

What we have here is a constructor of our library (that we can then use in robot side). It gets a single parameter "resources". For how, we treat it as a string that can have
semicolon as a separator for multiple names. Then, we try to acquire instance of robot by calling BuiltIn().  I'm catching RobotNotRunningError here because this code gets executed also during keyword
documenting process but robot itself is not running then. A bit outside of the scope here but worth to mention anyway. Next step, getting the list of modules. Which can be done via that iter_modules()
in private find modules and finally, we start iterating all the requests packanames from the passed-in variable, resources.

Each module mentioned in the resources is then checked if it exists in the modules we have, and a list of resource files is gathered from it and imported into running instance of robot via
import_resource().

Finally, lets see how this would look in the robot side of things:


```robotframework
*** Settings ***
Documentation   Verifies ImportResource Functionality
Library         ImportResource  resources=importresource-testdata

*** Test Cases ***
Test ImportResource
  ${returnvalue}=   keyword from package importresource-testdata
  Should Be Equal   ${returnvalue}    importresource-testdata

```

I've written the library already here and named it as ImportResource. I passed resources parameter to check a package called importresource-testdata and load all resource files from it. 

In the testcases, i make  call to `keyword from package importresource-testdata`, which is obviously a keyword [written in robot](https://github.com/rasjani/robotframework-importresource-testdata/blob/master/src/ImportResource-TestData/rf-resources/test.resource), packaged and installed via pip. By calling that keyword and storing its
return value, i'm able to verify that ImportResource has indeed loaded the expected resource file and and state in our testsuite is as expected. 


### Afterthoughts

First off, I made this only to learn and check out if i could actually pull this off. I think this sort of functionality should be backed into Robot Framework itself and i hope that making this sort
of proof of concept can convince core robot guys to pick up the idea. In fact, when i showed the project in RF slack @pekkaklarck was agreeing with the idea to have something similar in the core too. 

No matter how this sort of functionality ends up being implemented and used, i feel that if we as RF community get there, having the ability to share keyword libraries written in robot framework's own
syntax is enabling greater community resources and participation!

If you feel like that you have a need for this sort of functionality, for now you can use this proof of concept and maybe even contribute back. Project is installable via pip:

```bash
pip install robotframework-importresource
```

Links: 
 * [robotframework-importresource](https://github.com/rasjani/robotframework-importresource)
 * [robotframework-importresource-testdata](https://github.com/rasjani/robotframework-importresource-testdata)

### PS 

There are some bugs in the example code above and repository itself too. Some features might be added and better documents should be written.. 
