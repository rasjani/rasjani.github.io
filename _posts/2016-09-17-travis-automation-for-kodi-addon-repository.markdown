---
layout:	post
title:	"Travis automation for KODI addon repository."
date:	2016-09-17
---

  For a while, i’ve been maintaining few KODI addons. In order for users to find out and use these addons w/ automatic updates, they should be installed from a addon repository. KODI ships with their own, official repo, but getting 3rd party addons integrated into it is not always a straightforward task. All hope is not lost! 3rd party developers can create their own repositories and host them pretty much everywhere where one can host static files. Even GitHub =)

Taking care of every steps from developing a addon itself and updating the repository with new releases is tedious, atleast for lazy sob like me.

I started out with the idea that “if I tag a new release in the addon repo, the repository should be automatically updated and published”. Work for this easy to split into into two phases. First, upload the newly created released addon into the repository. Second, update the repository metadata. I started with the second one as it seemed easier to archive.

#### Updating repository metadata

There are quite a lot 3rd party repositories already but there’s not really a de-facto standard set of tools to update one’s one. Repositories usually structured like this:

Root folder contains a file called addons.xml, which lists all available addons with their corresponding metadata. Root folder also contains a directory for each addon. Naming of that folder needs to match the addon’s unique id. That folder then contains the addon itself as a zip file and addon.xml from that is also archived into the zip.

I found some tools that do update the repository’s addons.xml by iterating over addon directories and adding contents of the adddon.xml into it. This left out the manual part of extracting the addon.xml from the release file and placing it into same folder with the zip. [Commit](https://github.com/rasjani/xbmc-rasjanisrepo/commit/82fafe8839c59bbf336b7eabfeee36850720152a#diff-769eb8a92333fe4a99b293a4c0eb940dR59) later, with python packages “semantic\_version” and “zipfile”, i had functionality put together that finds the newest zip file in the addon folder, extracts addon.xml and places it into the correct directory.

That part done it was time to concentrate on automating the process. Travis was the obvious choice. While simple Travis setup is pretty straight forward process, I had few hickups while getting things up smootly.

#### Updating GitHub repository from Travis job

I googled around for tools or examples of how to upload build artifacts back to GitHub from Travis job. Either my search keywords where bad or not many people have made any posts about this as all i could find was some scripts to deploy only to gh-pages branch. Ofcourse this provided some guidance on how step forward. I would need to call my scripts to update repository data, provide means to authenticate git client inside Travis to authenticate itself and then push the changed files back to GitHub.

First off, I created a ssh keys and added public key to the GitHub repo’s settings as a deploy key. Next step, encrypt the the private key with Travis’ commandline tool and commited that encrypted file into the repo. Do note that in order to do this, Travis needs to be enabled for the git repo already and it should contain (atleast minimal) .travis.yml file.

At this point I wasn’t so sure at which point i should decrypt that file. Some examples where looking at where doing that in the deployment scripts. After experimenting with this for a bit, I decided that it would make more sense to do that in .travis.yml file itself. Reason for this is that i might want to deploy updates to repo on my local checkout and if the deploy script would do that for me, it would by-pass my personal GitHub account for no reason. So i settled for doing the decryption in before\_install section of the .travis.yml — do remember to set proper umask or chmod the decrypted file afterwards to be readable only by the user the file was created as ssh doesn’t allow keys to be readable/writable by anyone else except the user.

Since Travisjob is pushing back to my git repo, I also needed to define user’s name and email to the git client — without them, git client would not push. I opted to provide these values within .travis.yml file but i could just have placed the settings into Travis job’s settings where one can define multiple environment variables.

And it was then time to actually implement the actual deployment script and this is where i started to get puzzled why things where not working as expected.

At this point, my .travis.yml file was configured to install python dependencies in install phase, and run my python code in script and I could see the log that everything was building fine — but no matter what, git didn’t see any changes. Gotcha #1. Travis cleans everything after script unless you explicitly tell it not to with “skip\_cleanup” in deploy section. After I noticed this and Travis made a first “succesful” push to my git repo, i still couldn’t see the the changes. Gotcha #2. Travis does checkout for the build in detached head so anything pushed back to repo ends up as detached too. So, before adding changed files to a commit, checkout a branch where you actually want your changes to land. Travis provides environment value TRAVIS\_BRANCH which points to the branch, so it was quick fix, just switch to that branch before add/commit/push. Gotcha #3 - When Travis does the checkout of your repo, it does it via http(s), not with git/ssh protocol so when you push, you have to define the url of the remote repository where you are pushing into (or modify the origin url in git configs). Gotcha #4 — As i did setup the encrypted private key as a file in the git repo itself, when it gets decrypted during before\_install, a new file is created into the git repo. You do not want this file to get added into the commit. In order to avoid this, add the filename into .gitignore and/or remove the file itself after you add it to ssh-agent.

#### Final Words

With all this work now done, when i have a release zip file ready, i can just add it to repo and commit it and all the repository metadata is updated automatically. Next step is to automate the release process. For example, if i make a release tag in the GitHub repository of the specific addon, i want that zip file uploaded automatically to this addon repository. I have not yet done this but when i get there, i’ll make another post about it. Stay tuned!

  