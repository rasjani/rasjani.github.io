---
layout: post
title: Foray into GitHub actions
date: 2020-07-15 12:00:00
description: and how i set up automatically updating profile README
tags: [python, github, ci, flex]

---

During my professional career I've dealt with numerous CI systems. [Jenkins](https://www.jenkins.io), with all of its pitfalls and legacy reasons still is my go to tooling most of the time. I've worked with [BuildBot](https://buildbot.net/), [TeamCity](https://www.jetbrains.com/teamcity/),
[CruiseControl](http://cruisecontrol.sourceforge.net/) and various other ci/cd tools aimed for integration of deb or rpm packages. For my own opensource projects, I've toyed with [TravisCI](https://travis-ci.org/) & [AppVeyor](https://www.appveyor.com/) but ever sense Azure started to provided
[pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/) that could run on Windows, Linux and OSX, thats what I have been using.

About a week ago, I spotted a post on HN that described a setup he had used to create automatically updating github repo. In short, GitHub has "secret" feature where if you create a public repository
thats named as your github username, README from that repository will be shown in your Github profile page. Mine ended up looking something like [this](https://github.com/rasjani).

### Actions

Even thought GitHub actions have been publicly available, I haven't had huge interest or too much extra time to check them out. After seeing what what [simonw](https://github.com/simonw/simonw/) set
up, i thought this would be small enough "side project" to try out actions and write some code too.

While github actions are their own entity, it feels like some of the stuff is lifted off from Azure Pipelines. If not straight up copying, at least some of the ideas have been definitely lifted off
from Microsoft side. Configuration format using yaml also made me feel quite confortable.

### So what should I do ?

Since I do have a blog, maybe I could get one or two new readers if I list few recent articles in my profiles page?

Blog, like any of the thousands github pages is powered by jekyll. With few lines of configuration, it can also create a xml feed of entries. This provides a good entry point of getting the needed
data to generate links to the posts. And in order to get and parse that feed, there's always the handy FeedParser module. Point to the the feed and thats about it.  Initially I wrote just a small
python script that generates the README with fixed "layout" in markdown but after few runs, i thought that it would be nicer if i could make a template and fill it in with given set of values. Good
usecase for Jinja2 template engine. Barebones template i came up with looks like this:

```jinja
{% raw %}
### Latests posts
{% for post in posts -%}
* {{ post["date"] }} - [{{ post["title"] }}]({{ post["link"] }})
{% endfor %}

More posts @ [blog]({{ blogurl }})
{% endraw %}
```

Template iterates over a list of posts, shows a list of posts by date and clickable title to the url of the post and at the bottom, link to the blog itself.

And the code that uses this template ended up something like this:

```python
import feedparser
import jinja2
from pathlib import Path

BLOGURL = "https://rasjani.github.io"
FEED = f"{BLOGURL}/feed.xml"
MAX_POSTS = 5
template = None


def format_date(ts):
    return f"{ts.tm_mday:02}.{ts.tm_mon:02}.{ts.tm_year}"


with Path("README.template").open("r") as f:
    template = jinja2.Template(f.read())

feed = feedparser.parse(FEED)
posts = list(
    map(
        lambda post: {
            "date": format_date(post["published_parsed"]),
            "title": post["title"],
            "link": post["links"][0]["href"],
        },
        feed["entries"][0:MAX_POSTS],
    )
)


with Path("README.md").open("w") as f:
    f.write(template.render(blogurl=BLOGURL, posts=posts))
```

First, import the dependencies `feedparser` and `jinja2` and Path from pathlib.  Then I have few variables that define the base url and file name of the actual feed, how many posts should be shown and
variable where template is read into and finally rendered.

Single function that just takes care of formatting the `published_parsed` time entry from feedparser into a string.

Then, i read off the template file "README.template", download the whole feed and do some parsing on the results. I could have done the feeds postprocessing in the template but since I haven't toyed
that much with Jinja, i thought it would be easier to do that in python side. For this i used the trust `map()` function. It can be used to transform elements from any sort of iterables into something
else. This happens with the `lambda`. Lamba gets a single entry from `feed["entries"]`, limited to first with range selector.  Once lambda has a single post entry, i return a smaller dict with 3 keys:
  `date`, `title` and `link`. And once that map() is done, i save results into `posts` which gets passed to `template.render()` along with `BLOGURL`. And, that gets saved to `README.md`.

### Wiring that together

Only thing left to do at this point is making things run automatically and thats where I wanted to try out github actions.

Steps locally would be something like this:

* Install dependencies aka jinja & feedparser: `pip install -r requirements.txt`
* run my python script: `python build.py`
* Add and commit the changed README.md (if it was infact changed=): ```bash
        git config --global user.email "rasjani@users.noreply.github.com"
        git config --global user.name "BuildBot"
        git diff
        git add  README.md
        git commit -m "Updated content" || exit 0
        git push
```

Last step in short just set's up user name and email to use for commit, diff is shown to explicitly show what has changed in the build logs for better debugging. git add README.md will add the file
even if not changed and commit with message provided will either pass, and  doing a push if it does so, or just exit the pipeline with 0 if there was nothing to commit. You can see the full pipeline
[here](https://github.com/rasjani/rasjani/blob/master/.github/workflows/build.yml)

### Conclusion

Nice excersise to spend few hours one. GitHub actions looks and feels a lot like Azure Pipelines. Infact, I do think they are executed on same infra as Azure Devops.  Next steps, I think i'll convert
my main Azure Deveops pipeline to Github Action just for the heck of it!

Piece, out!


