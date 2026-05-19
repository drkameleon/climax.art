<h1 align="center">
    Climax
</h1>

<p align="center">
     <i>Elegant command-line<br>argument parsing for Arturo</i>
     <br><br>
     <img src="https://img.shields.io/github/license/arturo-lang/grafito?style=for-the-badge">
    <a href="https://github.com/arturo-lang/arturo" style="text-decoration: none; display: inline-block;"><img src="https://img.shields.io/badge/language-Arturo-6A156B.svg?style=for-the-badge" alt="Language"/></a>
</p>

---

<!--ts-->

- [What does this package do?](#what-does-this-package-do)
- [How do I use it?](#how-do-i-use-it)
    - [Basic usage](#basic-usage)
    - [Nested subcommands](#nested-subcommands)
    - [Default action](#default-action)
    - [Help templates](#help-templates)
- [Function reference](#function-reference)
    - [`climax`](#climax)
    - [`command`](#command)
    - [`group`](#group)
    - [Declaring options](#declaring-options)
    - [Accessing values](#accessing-values)
- [License](#license)

<!--te-->

---

### What does this package do?

This package exposes a single public entry point (`climax`) and two builders, `command` and `group`, that can be accessed from the main block. 

Every CLI you build with it automatically gets:

- subcommand dispatch (arbitrarily nested via `group`)
- typed positional arguments
- typed options with aliases, defaults & descriptions
- persistent (inherited) group-level options
- a `default:` command that runs when no sub-command is typed
- `--help` / `--version` for free (read straight from the script metadata)
- `--` rest-args passthrough

The whole DSL is built on top of three first-class types: `:option`, `:command` and `:cli`.

### How do I use it?

#### Basic usage

Simply `import` it, declare your commands and pass them to `climax`:

```red
;; name: webforge
;; description: « stupid simple static site generator
;; version: 1.0.0

import "climax"!

climax .with: [
    verbose? 'v "verbose output"
] [
    build: command "Build the site" [path :string :null][
        print ["building" path ?? "."]
    ]

    serve: command "Build + local server" [path :string :null] .with: [
        watch?
        port   'p  8080       :integer "port number"
        host      "localhost" :string  "bind host"
    ][
        print ["serving" path ?? "." "on" opts\host ":" opts\port]
        if opts\watch? -> print "watching for changes..."
    ]
]
```

Then:

```sh
$ webforge --help
$ webforge serve --help
$ webforge serve mysite --watch -p:9000
$ webforge -v build mysite
```

> [!TIP]
> Trailing `?` on an option name turns it into a boolean switch. Type defaults to `:logical`, default value to `false`.

#### Nested subcommands

Wrap a block of sub-commands in `group` to build `git`-style nesting:

```red
climax [
    remote: group "manage remotes" .with: [
        loud? 'l "verbose remote ops"
    ] [
        add: command "add a remote" [name :string :url :string][
            print ["adding" name url "loud?" opts\loud?]
        ]
        rm: command "remove a remote" [name :string][
            print ["removing" name]
        ]
    ]
]
```

```sh
$ myapp remote --help
$ myapp remote add origin https://example
$ myapp remote --loud add origin https://example   ;; persistent flag
```

Group-level options declared via `.with:` are inherited by every descendant; they can be passed anywhere on the command line and remain visible inside each leaf's `opts` dict. Groups can nest arbitrarily (groups of groups).

#### Default action

Use the reserved key `default:` to declare a command that runs when no recognised sub-command is typed. It can stand alone (flag-only CLI):

```red
;; name: ping
climax [
    default: command "ping a host" [host :string] .with: [
        count 'c 1 :integer "number of pings"
    ][
        loop 1..opts\count 'i -> print ["ping" i "->" host]
    ]
]
```

```sh
$ ping example.com -c 3
```

Or sit alongside named commands as a fallback:

```red
climax [
    default: command "show status" [] [ print "status: nominal" ]
    restart: command "restart service" [] [ print "restarting..." ]
]
```

```sh
$ myapp           ;; runs default
$ myapp restart   ;; runs restart
```

Inside a `group`, a sibling `default:` works the same way: bare `myapp <group>` runs it, named sub-commands still dispatch. The `default` key is elided from rendered help listings.

#### Help templates

Help screens render through a swappable `:climaxTemplate` instance. Two templates ship in `src/templates/`:

- `default` (used when no `.template:` attribute is supplied) — Arturo-style colourful output: bold green app name, bold cyan section headings, magenta flags & args, bold white sub-commands, gray dim hints.
- `plain` — black-and-white, byte-identical to the pre-template output. Opt in for piping, CI logs, or anywhere ANSI is unwanted.

```red
climax .template: 'plain [
    serve: command "..." [...] [...]
]
```

Roll your own template by defining a `:xxxTemplate` type and selecting it with the matching literal. Import your template file before calling `climax`:

```red
;; my-tmpl.art
define :myTemplate is :climaxTemplate [
    styleApp:     method [s :string] -> color.bold #red s
    styleSection: method [s :string] -> color.bold #yellow s
]
```

```red
;; your CLI
import "climax"!
import "./my-tmpl"!

climax .template: 'my [
    serve: command "..." [...] [...]
]
```

`.template:` takes a literal `'name`; climax instantiates `to :nameTemplate []` internally. The bundled `'default` and `'plain` follow the same convention — no special-casing for user templates.

Override points are layered so you only touch what you care about:

**Style primitives** — wrap a string in colour/bold/etc:

| Method | Wraps |
|---|---|
| `styleApp` | app name & path label in titles + USAGE |
| `styleSection` | section headings |
| `styleFlag` | option flag labels |
| `styleArg` | positional arg placeholders (`<name>` / `[<name>]`) |
| `styleCommand` | sub-command names in COMMANDS list |
| `styleDim` | tip footer & `(default: …)` suffix |

**Layout primitives** — constants that drive spacing:

| Method | Default |
|---|---|
| `indent` | `"    "` (4 spaces) |
| `flagColWidth` | `28` |
| `commandColWidth` | `12` |

**Section titles** — overridable for translation or renaming:

| Method | Default |
|---|---|
| `headerUsage` | `"USAGE"` |
| `headerOptions` | `"OPTIONS"` |
| `headerGlobalOptions` | `"GLOBAL OPTIONS"` |
| `headerCommands` | `"COMMANDS"` |

**Inline strings** — small text bits:

| Method | Default |
|---|---|
| `titleSeparator` | `"—"` (between name and description in titles) |
| `defaultLabel val` | `" (default: <val>)"` |
| `tipText label` | `"Run `<label> <command> --help` for command-specific help."` — return `""` to suppress the footer |

**Structural methods** — restructure rather than recolour: `renderRoot`, `renderGroup`, `renderCommand`, `argSig`, `optionLine`, `optionsSection`, `subcommandList`, `usageLine`.

### Function reference

#### `climax`

##### Description

build and dispatch a CLI from the given declarations

##### Usage

<pre>
<b>climax</b> <ins>decls</ins> <i>:block</i>
</pre>

##### Attributes

| Option | Type(s) | Description |
|----|----|----|
| with:     | `:block`   | global options (row grammar) |
| template: | `:literal` | help template — `'default` (default), `'plain`, or the literal name of a user-defined `:xxxTemplate` already imported into scope |

<hr/>

> [!NOTE]
> `command` and `group` are not module-level exports; they only exist as locals inside a `climax` decls block (and recursively inside any `group`'s sub-block). Calling them from anywhere else will result in an error.

#### `command`

Builds a leaf command spec.

<pre>
<b>command</b> <ins>desc</ins> <i>:string</i> <ins>args</ins> <i>:block</i> <ins>body</ins> <i>:block</i>
</pre>

| Option | Type(s) | Description |
|----|----|----|
| with: | `:block` | options block (row grammar) |

Returns `:command`.

<hr/>

#### `group`

Builds a command group whose body is a block of sub-command declarations.

<pre>
<b>group</b> <ins>desc</ins> <i>:string</i> <ins>decls</ins> <i>:block</i>
</pre>

| Option | Type(s) | Description |
|----|----|----|
| with: | `:block` | group-level options (persistent / inherited by sub-commands) |

Returns `:command` (with sub-commands attached).

<hr/>

#### Declaring options

Inside any `.with:` block, each option follows the same forced order:

```
<name>[?]  ['alias]  [<default>]  [:type ...]  ["description"]
```

| Slot | Required? | Notes |
|----|----|----|
| `name` | yes | trailing `?` marks a predicate (boolean switch) |
| `alias` | no | quoted single-char literal, e.g. `'v` |
| `default` | no | any literal value; predicates always default to `false` |
| `type` | no | one or more `:type` literals (union); predicates infer `:logical` |
| `description` | no | trailing `:string` |

#### Accessing values

Every command body sees three injected locals:

- `opts`: dict of resolved options (globals + per-command, with defaults applied)
- `rest`: block of raw args after `--`
- each positional arg, bound to its declared name

### License

MIT License

Copyright (c) 2026 Yanis Zafirópulos

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
