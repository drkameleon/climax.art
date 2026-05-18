<h1 align="center">
    Climax
</h1>

<p align="center">
     <i>Minimalist CLI builder for Arturo</i>
     <br><br>
     <img src="https://img.shields.io/github/license/arturo-lang/grafito?style=for-the-badge">
    <a href="https://github.com/arturo-lang/arturo" style="text-decoration: none; display: inline-block;"><img src="https://img.shields.io/badge/language-Arturo-6A156B.svg?style=for-the-badge" alt="Language"/></a>
</p>

---

<!--ts-->

- [What does this package do?](#what-does-this-package-do)
- [How do I use it?](#how-do-i-use-it)
- [Function Reference](#function-reference)
- [License](#license)

<!--te-->

---

### What does this package do?

This package provides a tiny set of building blocks (`climax`, `command`, `group`) for declaring command-line interfaces. Every CLI you build with it automatically gets:

- subcommand dispatch (arbitrarily nested via `group`)
- typed positional arguments
- typed options with aliases, defaults & descriptions
- persistent (inherited) group-level options
- `--help` / `--version` for free (read straight from the script metadata)
- `--` rest-args passthrough

The whole DSL is built on top of three first-class types: `:option`, `:command` and `:cli`.

### How do I use it?

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
        port 'p  8080       :integer "port number"
        host    "localhost" :string  "bind host"
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

Group-level options declared via `.with:` are inherited by every descendant — they can be passed anywhere on the command line and remain visible inside each leaf's `opts` dict. Groups can nest arbitrarily (groups of groups).

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
| with: | `:block` | global options (row grammar) |

<hr/>

#### `command`

##### Description

build a leaf command spec for use inside `climax` or `group`

##### Usage

<pre>
<b>command</b> <ins>desc</ins> <i>:string</i> <ins>args</ins> <i>:block</i> <ins>body</ins> <i>:block</i>
</pre>

##### Attributes

| Option | Type(s) | Description |
|----|----|----|
| with: | `:block` | options block (row grammar) |

##### Returns

- *:command*

<hr/>

#### `group`

##### Description

build a command group whose body is a block of sub-command declarations

##### Usage

<pre>
<b>group</b> <ins>desc</ins> <i>:string</i> <ins>decls</ins> <i>:block</i>
</pre>

##### Attributes

| Option | Type(s) | Description |
|----|----|----|
| with: | `:block` | group-level options (persistent — inherited by sub-commands) |

##### Returns

- *:command*

<hr/>

#### Option row grammar

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

##### Inside the command body

Every command body sees three injected locals:

- `opts` — dict of resolved options (globals + per-command, with defaults applied)
- `rest` — block of raw args after `--`
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
