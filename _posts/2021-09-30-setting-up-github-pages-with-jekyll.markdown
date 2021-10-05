---
layout: post
title:  "Setting up my Github Pages Site with Jekyll"
date:   2021-09-30 20:09:15 -0500
categories: blog
---

Growing as a software engineer means continually learning via projects, committing to open 
source, teaching, and mentoring. I've decided to start revamping my Github profile to document 
some of my learnings with some of the things mentioned above. The first step in this is creating 
a personal site to host my resume and start a blog.

<!--more-->
This repo is just that. I've decided to learn a new tool along the way by creating this static 
site with [Jekyll][jekyll]. As I update this repo, I'll continually update this README as well. 
The end goal is this will be my first blog article on my personal site. 

## Installing Jekyll

> I can't guarantee that this section will stay relevant. I am installing these tools on OS X Mojave 
> in October of 2021. As there is already a new version of OS X out, I encourage you to look at the 
> install instructions of Jekyll's site for any updates.

As I mentioned in the introduction, I have decided to learn how to create a static site with Jekyll to 
build my personal site. The first step is [installing][jekyll-install] the 
tools on my computer. I currently use a Mac for development so I'll be discussing the 
[Mac OS install instructions][jekyll-macinstall].

2 tools are required before you start - XCode and Homebrew. Install these first to make installation go 
faster.

Since I do not use Ruby normally, I opted to install Ruby with Homebrew and point my path to that Ruby 
install. This seemed to be a quicker option and I don't foresee any issues with it at this time. I did 
install the Jekyll gem locally as suggested.

Overall, the install process went smoothly. The documentation provided by Jekyll is sufficient.

## Creating the Site

Github has posted some good [documentation][github-create] for creating a site in a repo with Jekyll. 
For my site, I am going to serve Github pages from the root directory instead of the `docs` 
directory like in the tutorial. To install in the root directory, I had to add the `--force` flag to the `jekyll new` command.

One issue not mentioned in the Github documentation is a small issue when using Jekyll 4.2.1 and 
Ruby 3.0.0. The work around is simple and is mitigated by running the command `bundle add webrick` 
in the site directory you have chosen after running `jekyll new --skip-bundle .`.

Additionally, I am setting my github pages site to run from the main branch directly.

After setting up the initial site and testing basic functionality locally, I changed a few of the 
defaults so that I could personalize the site. I then tested locally once more before pushing to 
Github for testing.

In detail, I changed the `_config.yml` file to show my content:

{% highlight yml %}
title: Kevin Benton
email: krbenton@icloud.com
description: >- # this means to ignore newlines until "baseurl:"
  Personal site and blog.
baseurl: ""
domain: "kevin-benton.github.io"
url: "https://kevin-benton.github.io"
twitter_username: Kevin_R_Benton
github_username: kevin-benton
{% endhighlight %}

## Next Steps

I still have a lot I want to do with this site and I will link the next step of my improvements to this 
post. I don't know how many posts I will do on this. In my next post I want to change the theme, add a 
custom layout or 2, and add my resume to the about page. There may also be some other things so stay 
tuned.

[jekyll]: https://jekyllrb.com
[jekyll-install]: https://jekyllrb.com/docs/installation/
[jekyll-macinstall]: https://jekyllrb.com/docs/installation/macos/
[github-create]: https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll
