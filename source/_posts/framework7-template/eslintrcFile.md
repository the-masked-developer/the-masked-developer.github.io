---
title: Configure .eslintrc file
toc: true
date: 2020-03-19 19:04:47
tags:
	- Framework7
categories:
	- Framework7
---

# Configure .eslintrc file

> This page is part of the [App Framework Documentation](../DOCUMENTATION.md)

<br />

The *.eslintrc* file will be updated automatically.

You can configure it in the `eslint` object in the *app/config.json* file.

Example:

```
"eslint": {
  "extends": "airbnb",
  "rules": {
    "semi": [
      "error",
      "never"
    ]
  }
}
```

Currently, `airbnb` and `standard` are available to extend - if you wish to extend another shared configuration, please ask for it in our [issue list](https://github.com/scriptPilot/app-framework/issues).
