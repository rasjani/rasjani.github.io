---
layout:	post
title:	"Debugging 3rd party NPM Module regression."
date:	2015-03-19
---

  I was recently tasked to help with upgrading our npm module dependencies. The reason for this was after upgrading jasmine from 2.1.3 to latest 2.2.0, a small portion of our test cases started to fail. Which, if you have ever worked with npm, is quite common. So, I’ll be using this scenario as an example in how to debug 3rd party npm module regressions to help demonstrate the approach so it can be used generally.

So, we have a regression somewhere, what now? There’s few obvious things that might be the root cause.

* APIs have changed from one version to another
* Genuine regression in the dependency
The first scenario could be due to an error that is between your own code and it’s dependency, or it could also be between your dependencies. This is too often the case because people are lazy or ignorant when they [declare versions in package.json](http://blog.nodejitsu.com/package-dependencies-done-right/).

In our case, we were using [jasmine](https://github.com/jasmine/jasmine/), [sinon](https://github.com/cjohansen/Sinon.JS/) and [jasmine-sinon](https://github.com/froots/jasmine-sinon) matchers. Our test cases where failing in steps where we were using jasmine.any() in toHaveBeenCalledWith() jasmine-sinon matcher.

There are few ways to isolate whats up with the problems at this stage. What has changed ? Check out the release notes if there’s anything that could someone relate to the issue at hand. But that requires a lot of guess work. How about brute forcing your way into the root cause. You know, just open the damn code and do some trace debugging to try to pinpoint what could be the cause by running it and seeing what happens and where. For this, we need “use the source, luke!”

git clone git@github.com:jasmine/jasmine.gitAfter going through the checkout, it’s painfully obvious: if you know the codebase of the 3rd party lib well, this might be good solution. But most often this is not the case

Let’s stop here and think. We have established that we have a version that works, and newer version that does not. If we know what the piece of code was that caused the failures, we have a good starting point to actually fixing these errors.

We have the source but how can we use this particular git repo as part of the debugging process as it’s just a repo and we need it as proper dependency inside our main project’s node\_modules? The answer relies on npm itself; with npm link you can mark any git repository as a provider for the package defined in the original repo’s package.json file.

cd jasmine  
sudo npm linkThen, in my main project’s code which has the failing tests, I can do following:

npm install jasmine-coreIf the jasmine-core is linked as I did earlier on, node\_modules/jasmine-core is actually now a symlink to my git clone.

Almost there!

If I now do a checkout against specific commit in this particular repository, any changes to the tree will be mirrored automatically to node\_modules/jasmine-core of my main project. If there’s only few commits, iterating through all of them can be done manually but if you are facing hundreds of commits, we are going to need some automation.

Enter [git bisect](http://git-scm.com/docs/git-bisect "Git Bisect")!

Git bisect is a tool that uses binary search against commit history to pinpoint what the actual commit was that introduced the bug into codebase.

So far, I know that the version 2.1.3 was working and HEAD^ of the master wasn’t. Let’s start bisecting the froggie:

git bisect start  
git bisect good v2.1.3  
git bisect bad HEAD^After this, my git clone ended up in a random checkout between v2.1.3 and HEAD commits and it’s now time to run our tests in the main repository to see if we have a failure or not.

grunt karma:unit And depending on the outcome of the unit tests i was running, i kept marking commits either

git bisect goodor

git bisect badAnd finally, after a few iterations, the truth was revealed:

1c6f4ef0e69cd6cf8a8620e8acd336b880c494d8 is the first bad commit commit 1c6f4ef0e69cd6cf8a8620e8acd336b880c494d8 Author: Gregg Van Hove and Molly Trombley-McCann <pair+gvanhove+molly@pivotal.io> Date: Wed Mar 4 11:58:47 2015 -0800Build distribution for previous changes:040000 040000 c9b2dd92b9ff9da933fb14f372508865f0de8f15 2553f19e9f9f2ed8574e29522d1fa8439d180914 M libAnd since I now knew the root cause, i can safely stop the bisecting:

git bisect resetand quick diff to show what what was changed in the commit:

git diff 1c6f4e^ 1c6f4eAt this point, since I know what had been changed and what I was calling, it was really easy to make the changes to fix my test cases.

### Automating the bisect process.

In the above walk through example, I was marking each commit to either good or bad manually. This could be cumbersome in some cases and a lazy person just wants to fire a process and wait for the results. That’s why there’s a [git bisect run](http://git-scm.com/docs/git-bisect#_bisect_run)

Run allows us to provide a command or a script that gets executed against every commit between what you have marked as good and bad starting points.

In my case, I could have come up with something like this:

janimikkonen@RASJani ~/src/js/jasmine (master)$ cat ./runMyTests.sh  
cd ../myapp/  
grunt karma:unit  
STATUS\_FROM\_UNITTESTS=$?  
cd ../jasmine/  
exit $STATUS\_FROM\_UNITTESTS  
janimikkonen@RASJani ~/src/js/jasmine (master)$Quick explanation what is happening here. If I execute:

git bisect run ./runMyTests.sh My script will first change the working directory to ../myapp/, then it will run my karma tests (where we had the regression). Shell exitcode is then saved to STATUS\_FROM\_UNITTESTS right after grunt has executed the tests, return back to original directory and exit the script with same status as grunt exited from testrun.

  