# Apple Notes Export Tools

This repository includes a few python export tools for Apple's "Notes.app".  The scripts require python3, but I tried to make the scripts self-contained and concise, with no additional dependencies. I may revisit this and break the common code into a library in the future.

This repository is in the public domain.

## `notes.md`

Description of how notes data is stored.

## `notes2bear`

Writes a `notes.bearbk` file in Bear's backup format (a zip file with their own markup flavor).  It doesn't handle tables because Bear doesn't do tables yet. This shaves about 100 lines from the script.

## `notes2html` 

`notes2html` takes an destination directory and writes a tree of html files and images.  One html file per note and any associated media in the media directory. If you specify `--title` the files with be named with the title guessed by Apple (with / replaced by _).  If you specify `--svg`, any drawings will be rendered as inline SVG; otherwise, the fallback jpg files provided by Apple will be used.

Usage is:

```
notes2html [--svg] [--title] dest
```










