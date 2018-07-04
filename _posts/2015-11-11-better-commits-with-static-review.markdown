---
layout: post
title: Better Commits with Static Review
date: '2015-11-11 04:50:00'
tags:
- tutorial
- git
- quality
- tools
---

One of the best changes that I have ~~recently~~ ever made to my workflow has been reading Chris Beams' excellent [How to Write a Git Commit Message](http://chris.beams.io/posts/git-commit/). If you haven't read it, you should go do it right now and come back once you are done.

# The Problem

You're programming something and it seems to be working fine. You commit it and push it for QA to review. They come back with a problem that you didn't consider. You write a fix and push it again. Repeat this a few more times and you're starting to get frustrated and your commit messages start to look like this:

> Fixed for real

And then a four hours later, when all you want is a beer:

> I hate my life

Finally you get it working and the next day your code reviewer looks at your pull request and has a comment:

> This block violates [PSR-2 elseif rules](http://www.php-fig.org/psr/psr-2/#5-1-if-elseif-else)

![seriously?](https://dl.dropboxusercontent.com/u/7258484/Laughs/dafuq.gif)

All you wanted to do was be done with this code! Who cares, right?!

## The Real Problem

You are not a bad programmer. You are just moving too fast and you had developed some bad habits because of it. As my favorite Zen quote goes:

> You are perfect just as you are, and you could use a little improvement. *– Shunryu Suzuki*

What can you do to improve your commit habits?

# The Solution

One of the most powerful features of Git is [hooks](http://www.git-scm.com/book/en/v2/Customizing-Git-Git-Hooks). These little scripts can let us validate our changes using external tools, block us from committing, and check our commit messages for us. This can force us to slow down and be sure that what we are committing is what we really want.

Luckily for us [sjparkinson](https://github.com/sjparkinson) has built a fantastic tool called [Static Review](https://github.com/sjparkinson/static-review) that does all of the heavy lifting for us. It can do everything from running [composer validate](https://getcomposer.org/doc/03-cli.md#validate) to [phpcs checks](https://github.com/squizlabs/PHP_CodeSniffer) for style violations to commit message review of [the seven rules](http://chris.beams.io/posts/git-commit/#seven-rules).

## Installing Static Review

The best way to install static review is using a [Composer global](https://getcomposer.org/doc/03-cli.md#global):

    composer g req sjparkinson/static-review

Next you will want to allow globally installed Composer binaries to be used as normal commands. Edit your `.bashrc` or `.zshrc` or other shell configuration to contain the global `bin` path. It should look something like:

    export PATH="$PATH:$HOME/.composer/vendor/bin"

Now you should have `static-review.php` available in your shell at all times. The next step will be to add the hooks to any Git repository that you want to do reviews on. To install the default file checks:

    static-review.php hook:install ~/.composer/vendor/sjparkinson/static-review/hooks/static-review-pre-commit.php .git/hooks/pre-commit

To install the default commit message checks:

    static-review.php hook:install ~/.composer/vendor/sjparkinson/static-review/hooks/static-review-commit-msg.php .git/hooks/commit-msg

Now whenever you make a commit, and all goes well, you will see something like this in your shell:

```
Reviewing 1 of 1.
✔ Looking good.
Have you tested everything?
✔ That commit looks good!
```

Repeat the same `hook:install` commands for any repository you want to enable these hooks on.

### Optional Dependencies

If you want to enable the Composer [security checker](https://github.com/sensiolabs/security-checker) you will need to install it globally as well:

    composer g req sensiolabs/security-checker

And if you want to enable [PHP Code Sniffer](https://github.com/squizlabs/PHP_CodeSniffer):

    composer g req squizlabs/php_codesniffer

# Conclusion

The excellent [Static Review](https://github.com/sjparkinson/static-review) Git hooks help us have better habits by ensuring that we write good commit messages and follow best practices in our code. While it might seem annoying and slow at first, with time you will become aware of your mistakes as they happen, instead of being reminded by someone else later. By writing better code we help everyone that we collaborate with and maintain a high standard of quality throughout our projects.