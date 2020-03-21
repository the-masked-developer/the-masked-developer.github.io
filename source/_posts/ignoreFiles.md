---
title: Configure gitignore and npmignore files
toc: true
date: 2020-03-24 19:04:47
tags:
	- Framework7
categories:
	- Framework7
---

# Configure .gitignore and .npmignore files

> This page is part of the [App Framework Documentation](../DOCUMENTATION.md)

<br />

The *.gitignore* and *.npmignore* files are updated automatically, based on a default configuration.

You can overwrite this configuration in `gitignore` and `npmignore` objects in the *app/config.json* file.

Example: Add a custom folder to the *.npmignore* file and remove the *.editorconfig* file

```
"npmignore": {
  "customFolder/": true,
  ".editorconfig": false
}
```

`true` means, the path will be listed in the ignore file.
