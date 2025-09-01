---
title: Auto formatting extensionless Bash scripts in Neovim
description: Here's what I did to make the combination of the Bash language server and shfmt work with Editorconfig settings for Bash script files that don't have extensions.
date: 2025-09-01
tags:
  - bash
  - editorconfig
  - neovim
  - lsp
  - shfmt
---
## Background

In my continuing quest to [build out a cleaner and leaner Neovim configuration](/blog/posts/2025/06/10/a-modern-and-clean-neovim-setup-for-cap-node.js-configuration-and-diagnostics/) I turned my attention recently to improving my Bash script editing experience. For Bash there are plenty of facilities that fit into my Personal Development Environment, including `shellcheck` and `shfmt`, both of which I have been using for quite a while (see [Improving my shell scripting](/blog/posts/2020/10/05/improving-my-shell-scripting/)).

I decided to give the [Bash language server](https://github.com/bash-lsp/bash-language-server) a try. It's a super easy setup [for Neovim 0.11+](https://github.com/bash-lsp/bash-language-server?tab=readme-ov-file#neovim) and the [integration with shfmt](https://github.com/bash-lsp/bash-language-server?tab=readme-ov-file#shfmt-integration) was a feature that caught my eye in particular.

Add to this a desire to also more consciously embrace [Editorconfig](https://editorconfig.org/), especially given [shfmt's support for that](https://github.com/mvdan/sh/blob/master/cmd/shfmt/shfmt.1.scd#description), and I was set for a great experience.

## The perfect storm

I am of the opinion that shell scripts that are written to be executed on the command line should not have extensions. This follows a [specific guideline](https://google.github.io/styleguide/shellguide.html#s2.1-file-extensions) in the [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html). Interpretation of shell script file\[type\]s is based on the [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)).

This approach, combined with the Bash language server, its support for `shfmt`, and a custom part of `shfmt`'s support for Editorconfig, turned out to be a small but perfect storm:

- Editorconfig's approach and configuration ([specification](https://spec.editorconfig.org/)) is based on file extensions only and there is no provision for specifying editor settings for files without extensions
- As the extensionless shell script is such a common phenomenon, support for Editorconfig within `shfmt` has been extended to allow for special [section names](https://spec.editorconfig.org/#glob-expressions) in Editorconfig configuration files so that such shell scripts can be matched more generically; this support is described in the [Examples](https://github.com/mvdan/sh/blob/master/cmd/shfmt/shfmt.1.scd#examples) section in `shfmt`'s documentation thus:
  > "EditorConfig sections may also use `[[shell]]` or `[[bash]]` to match any shell or bash scripts, which is particularly useful when scripts use a shebang but no extension. Note that this feature is outside of the EditorConfig spec and may be changed in the future."
- While it's great to have [support for shfmt](https://github.com/bash-lsp/bash-language-server?tab=readme-ov-file#shfmt-integration) in the Bash language server, this support does not include this non-standard (but insanely useful) `[[bash]]` style facility for Editorconfig configuration.

Consequently, when editing shell scripts with extensionless filenames using Neovim with the Bash language server configured and `shfmt` installed[<sup>1</sup>](#footnotes)
I got a very odd and confusing experience.

## Example of the issue

Here's an example, based on these (reduced) Editorconfig settings in my `.editorconfig` file:

```toml
root = true

[*]
indent_style = tab

[[bash]]
indent_size = 2
indent_style = space
switch_case_indent = true
binary_next_line = true
space_redirects = true
```

and shell script files called `a`, `b` and `fixup`.

*On the command line*, running `shfmt`[<sup>2</sup>](#footnotes) on each of these three files produced the expected results, in that all were formatted according to the `shfmt`-specific settings in that `[[bash]]` section plus of course indented with 2 spaces.

*Within Neovim*, via the Bash language server[<sup>3</sup>](#footnotes), with shell script files `a` and `b` everything seemed to work fine and my Bash scripts were indented appropriately with 2 spaces. When I then started to work on real shell scripts such as one called `fixup`, things didn't work the same, and the indentation was done with tabs, not spaces (the horror!).

Yes, you guessed it, this was because in standard Editorconfig procedure, the `[bash]` part within the `[[bash]]` section heading is considered a sequence of characters `b`, `a`, `s` and `h`, matching files of those single-character names; and as `fixup` was not matched with that sequence, that section (containing `indent_style = space`) didn't apply, and so `indent_style = tab` prevailed.

This is unfortunate, but is not really anyone's fault.

## Bash language server, shfmt and Editorconfig

In preparing to call `shfmt`, the Bash language server determines properties (appropriate for the file in question) from the Editorconfig configuration. It uses the [editorconfig](https://www.npmjs.com/package/editorconfig) NPM package for this determination, and the custom `shfmt` extension to allow for `[[bash]]` and `[[shell]]` does not exist in that package.

There are generic configuration properties (such as `indent_style`), and `shfmt`-specific ones (such as `switch_case_indent`). If there are any `shfmt`-specific ones then the Editorconfig configuration is used. Otherwise, the language server configuration is used. This reflects the approach taken by `shfmt` itself in using either Editorconfig or its own command line flags, but not both.

The configuration is then turned into a series of explicit `shfmt` command line options, in particular (but not exclusively) from the [Printer flags](https://github.com/mvdan/sh/blob/master/cmd/shfmt/shfmt.1.scd#printer-flags) group. For example, on surfacing a `binary_next_line = true` setting in `.editorconfig` it will generate an equivalent `-bn` flag to pass in the call to `shfmt`.

Then the Bash language server invokes `shfmt` passing these generated options plus the `--filename` option to specify the full pathname of the file being formatted.

### Example for script called 'fixup'

Here's an example of that for `fixup` for which the `editorconfig` package does _not_ match the `[[bash]]` section:

```shell
shfmt --filename=/home/dj/test/fixup -i0 -ln=auto
```

- the `-i0` comes from a default driven by the basic details sent in the request from Neovim's LSP client, which includes `insertSpaces = false`
- the `-ln=auto` comes from [a default in the Bash language server itself] for `languageDialect` (see also [Appendix A](#appendix-a-mason-info-for-bash-language-server))

### Example for script called 'a'

Here's another example for `a` which (by chance) _was_ matched by the `[[bash]]` section:

```shell
shfmt --filename=/home/dj/test/a -i=2 -bn -ci -sr
```

- these options come from the `indent_size`, `binary_next_line`, `switch_case_indent` and `space_redirects` Editorconfig properties in the `[[bash]]` section

## Useful debugging approaches

In working out what was going on, I used various tools and settings to help me. Here's a quick list of those, mostly to remind my future self.

### The editorconfig CLI

The [editorconfig](https://www.npmjs.com/package/editorconfig) package, which I installed globally for these investigations with `npm i -g editorconfig`, also sports a CLI which can be used to show the "effective configuration", given a filename. Taking the `.editorconfig` file contents in the [example of the issue](#example-of-the-issue) shown earlier, here's what is output ...

for the shell script file named `a`:

```shell
$ editorconfig a
indent_style=space
indent_size=2
switch_case_indent=true
binary_next_line=true
space_redirects=true
tab_width=2
```

and for the shell script file named `fixup`:

```shell
$ editorconfig fixup
indent_style=tab
indent_size=tab
```

## Appendix A - Mason info for bash-language-server

```text
Language Filter: press <C-f> to apply filter

Installed (1)
  ◍ bash-language-server
    A language server for Bash.

    installed version 5.6.0
    installed purl    pkg:npm/bash-language-server@5.6.0
    homepage          https://github.com/bash-lsp/bash-language-server
    languages         Bash, Csh, Ksh, Sh, Zsh
    categories        LSP
    executables       bash-language-server

    ↓ LSP server configuration schema (press enter to collapse)
      This is a read-only overview of the settings this server accepts.
      Note that some settings might not apply to neovim.

      → bashIde.backgroundAnalysisMaxFiles    default: 500
      → bashIde.enableSourceErrorDiagnostics  default: false
      → bashIde.explainshellEndpoint          default: ""
      → bashIde.globPattern                   default: "**/*@(.sh|.inc|.bash|.command)"
      → bashIde.includeAllWorkspaceSymbols    default: false
      → bashIde.logLevel                      default: "info"
      → bashIde.shellcheckArguments           default: ""
      → bashIde.shellcheckPath                default: "shellcheck"
      → bashIde.shfmt.binaryNextLine          default: false
      → bashIde.shfmt.caseIndent              default: false
      → bashIde.shfmt.funcNextLine            default: false
      → bashIde.shfmt.ignoreEditorconfig      default: false
      → bashIde.shfmt.keepPadding             default: false
      → bashIde.shfmt.languageDialect         default: "auto"
      → bashIde.shfmt.path                    default: "shfmt"
      → bashIde.shfmt.simplifyCode            default: false
      → bashIde.shfmt.spaceRedirects          default: false
```

## Footnotes

1. as well as `shellcheck` which [is also supported](https://github.com/bash-lsp/bash-language-server?tab=readme-ov-file#dependencies)
1. version [3.8.0](https://github.com/mvdan/sh/releases/tag/v3.8.0) or higher, as that is when support for Editorconfig sections such as `[[shell]]` [was introduced](https://github.com/mvdan/sh/issues/664)
1. [configured](https://github.com/qmacro/dotfiles/blob/57c7f38e64ef1f59b9c41f6155b0fa350eb030b7/config/nvim/init.lua#L14-L23) through a `BufWritePre` event based call to `vim.lsp.buf.format`

