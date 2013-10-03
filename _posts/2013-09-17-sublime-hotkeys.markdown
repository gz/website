---
layout: post
title:  "Sublime Shortcuts 101"
date:   2013-09-07 16:41:30
categories: sublime2
---

I recently switched to [Sublime Text 2][0] as my default editor. Here are a couple of shortcuts that I will hopefully have memorized soon. Until then, this post will serve as a reminder. Most of them are from the Youtube video [Perfect Workflow in Sublime Text 2][1].

### Command Palette

* <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>p</kbd> ```fuzzy terms```: Executes command
* <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>p</kbd> ```toggle```: Toggle anything
* <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>p</kbd> ```language```: Change language syntax

### GUI Layout

* <kbd>Ctrl</kbd>+KB: Toggle the sidebar
* <kbd>F5</kbd>: Reveal file in side-bar
* <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>2</kbd>: Move file to second view
* <kbd>Ctrl</kbd>+<kbd>2</kbd>: Switch cursor to second view
* <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>`</kbd>: Make views equal
* <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>1</kbd>: Make view 1 bigger
* <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>2</kbd>: Make view 2 bigger

For the last three you're going to need some additional [key bindings][2].

### Text editing

* <kbd>Ctrl</kbd>+<kbd>d</kbd>: Select next occurence of a word
* <kbd>Ctrl</kbd>+<kbd>u</kbd>: Deselect previous occurence of a word
* <kbd>Alt</kbd>+<kbd>F3</kbd>: Select all occurences of a word
* <kbd>Shift</kbd>+<kbd>RMB</kbd>: Select Rectangular Text area
* <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>L</kbd>: Edit all selected lines
* <kbd>Ctrl</kbd>+<kbd>i</kbd>: Interactive search, combine with <kbd>Ctrl</kbd>+<kbd>d</kbd> to search and edit!
* <kbd>Ctrl</kbd>+<kbd>j</kbd>: Join lines on one line
* <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>v</kbd>: Paste and keep indentation

### Vintage Mode

Remove Vintage plug-in from ```ignore_packages``` in your settings to get VIM mode!

### Navigating Files

* <kbd>Ctrl</kbd>+<kbd>p</kbd> ```folder/filename```: Fuzzy open files
* <kbd>Ctrl</kbd>+<kbd>p</kbd> ```filename>@<symbol```: Open filename and go to symbol
* <kbd>Ctrl</kbd>+<kbd>R</kbd> or <kbd>Ctrl</kbd>+<kbd>p</kbd> ```@```: Symbol view of current open file

### Regular Expressions

Make sure regular expression engine is enabled!

* <kbd>Ctrl</kbd>+<kbd>i</kbd> ```<h[0-9]>.+</h[0-9]>``` <kbd>Alt</kbd>+<kbd>Enter</kbd>: Match example
* <kbd>Ctrl</kbd>+<kbd>i</kbd> ```(?<=<h[0-9]>).+(?=</h[0-9]>)``` <kbd>Alt</kbd>+<kbd>Enter</kbd>: Negative Match example

### Project Settings

A ```<project>.sublime-project``` file in the top level directory of your project lets you add project specific settings:

```
"file_excluse_patterns": ["*.css"] // Remove specific files from project view
"folder_exclude_patterns:" ['js', 'css']  // Remove specific folders
"settings": [] // Specific project settings
```

### Useful User Settings
```
"trim_trailing_white_space_on_save": true, // Remove whitespaces at the end of lines
```

### Plugins

* Nettuts-Fetch: Fetchs online version of remote files
* Advanced New File: Specify file path while creating a new file
* SublimeLinter: Static analysis of various programming languages
* SyncedSideBar: Reveal file in side-bar
* Docblockr: Easier documenting for source code
* SublimeClang: On-the-fly compilation of C/C++ code, somewhat buggy and no longer developed :-(.
* ctags: C code indexing
* sublime-cscope: C code indexing
* GitGutter: Display edited lines on the side of the file
* LaTeXTools: On-the-fly compilation
* Python PEP8 Autoformat: Formatting for Python sources
* LaTeX Word Count: Counts words in a LaTeX project
* SublimeAStyleFormatter: Formats various programming language source files
* ZenCoding: Less typing for HTML
* Emmet-sublime: Less typing for CSS


[0]: http://www.sublimetext.com/ "Sublime Text official Website"
[1]: https://www.youtube.com/watch?v=TZ-bgcJ6fQo "Screencast"
[2]: https://gist.github.com/gz/6594118 "Gist with Shortcuts"
