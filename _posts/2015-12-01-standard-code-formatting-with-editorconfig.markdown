---
layout: post
title: Standard Code Formatting with EditorConfig
date: '2015-12-01 18:27:00'
tags:
- quality
- tools
- editor
- style
- vim
---

[EditorConfig](http://editorconfig.org/) is a very simple configuration file that can be used in any code base to define formatting expectations like spaces or tabs, size of tab, file ending newlines, etc.

## Installation

First you will need the `editorconfig` binary. On a Mac with [Homebrew](http://brew.sh/), this is as simple as:

```
brew install editorconfig
```

## Creating the Configuration File

Now you will want to create the `.editorconfig` file in your project root. My standard config looks like this:

```
# EditorConfig is awesome: http://EditorConfig.org

# top-most EditorConfig file
root = true

# Unix-style newlines with a newline ending every file
[*]
end_of_line = lf
insert_final_newline = true

# PHP/JS Formatting
[*.{php,js,json}]
charset = utf-8
trim_trailing_whitespace = true
indent_style = space
indent_size = 4
```

This says that:

- All new lines are UNIX style line-feed
- All files end with a newline

And for PHP, JS, or JSON files:

- Character set is UTF-8
- All trailing whitespace is stripped
- Indentation is 4 spaces instead of tabs

## Using it with Vim

Install the [editorconfig-vim](https://github.com/editorconfig/editorconfig-vim) plugin according to the instructions. (You're already using [Pathogen](https://github.com/tpope/vim-pathogen) or [Vundle](https://github.com/VundleVim/Vundle.vim), right?)

Once the plugin is installed, it should Just Work. Open a file, add some trailing whitespace, save it and see the magic in action!
