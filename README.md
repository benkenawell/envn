# envn

A small CLI for imperatively reading and editing a dotenv file, modeled on
`mise config get/set/ls`. Edits preserve comments, blank lines, ordering,
`export` prefixes, and inline comments.

This utility is NOT for loading envvars into a session or running a command with them in its environment. It is for management of the files only. If you need something to load them, checkout direnv or mise.

Mostly vibed, use as your own risk. I couldn't find an easy cli to install that let me manage dotenvs well from the command line, so I had the AI write this up for me.

## Install

With [mise](https://mise.jdx.dev):

```sh
mise use -g github:benkenawell/envn@latest    # prebuilt release binary
mise use -g go:github.com/benkenawell/envn    # build from source
```

With Go:

```sh
go install github.com/benkenawell/envn@latest
go build -o envn .          # local build from a checkout
```

Prebuilt binaries for Linux, macOS, and Windows are attached to each
[GitHub Release](https://github.com/benkenawell/envn/releases).

## Usage

The target file defaults to `.env` in the current directory. Override it with
`--file`/`-f` on any command, or the `ENVN_FILE` environment variable.

```sh
envn get DATABASE_URL            # print one value (unquoted, escapes resolved)
envn get DB_HOST --all           # -a, every active value of a repeated key, in file order
envn get DB_HOST --all --json    # every occurrence incl. disabled, with line and description
envn get                         # print the whole file

envn set PORT 8080               # rewrite in place, or append if new
envn set DATABASE_URL=postgres://localhost/app
envn set SECRET 'has spaces'     # quoted automatically when needed
envn set -e 'Postgres DSN' DATABASE_URL postgres://localhost/app
                                   # --explain writes a comment block above the key
envn set --append DB_HOST staging.internal
                                   # add another occurrence after the last one; it becomes effective
envn set --append --disabled DB_HOST prod.internal -e 'Production'
                                   # add it commented out; effective value unchanged
envn set --disabled FEATURE_X 1  # write the key but leave it commented out

envn ls                          # effective view: one row per key, last active value wins
envn ls --no-header
envn ls --json                   # -J, flat {key: value} object
envn ls --keys                   # -k, key names only
envn ls --all                    # -a, file view: every occurrence with line and description, disabled included
envn ls --all --json             # array of {line, key, value, description, enabled}

envn explain                     # line, key, and comment for every key
envn explain --json
envn explain DATABASE_URL        # print the comment block above the key
envn explain get DATABASE_URL    # same thing, long form
envn explain get DB_HOST --all   # -a, one row per occurrence, disabled included
envn explain set DATABASE_URL 'Postgres connection string'
envn explain remove DATABASE_URL # delete the comment block, keep the key

envn disable DEBUG               # comment the line out: "# DEBUG=..."
envn enable DEBUG                # uncomment it again
envn toggle DEBUG                # flip between the two

envn use DB_HOST staging.internal   # make that occurrence the only active one
envn use DB_HOST --line 6           # select by line number from `ls --all`
envn use DB_HOST                    # with several occurrences: list them, change nothing

envn remove DEBUG                # aliases: rm, unset, delete
envn remove DB_HOST staging.internal   # one occurrence by value
envn remove DB_HOST --line 6           # one occurrence by line
envn remove DB_HOST --all              # every occurrence

envn format                      # print the file with repeated keys grouped (alias: fmt)
envn format -i                   # --in-place: rewrite the file instead
envn format --check              # exit 1 if the file would change

envn -f .env.production get API_KEY
```

A comment block is one or more consecutive `#` lines directly above an entry;
a blank line ends it. `explain set` and `set --explain` replace any existing
block, and newlines in the text produce multiple comment lines. `explain set`
needs the key to exist; `set --explain` creates key and description together.
`explain remove` deletes only the block, while `remove` deletes the key and its
block. All of these work on disabled keys too.

Given this `.env`:

```sh
# Postgres connection string
DATABASE_URL=postgres://localhost/app

# Port the API listens on
PORT=8080

# Verbose request logging
# DEBUG=true
```

`envn explain` prints each key's line, name, and comment. The disabled
`DEBUG` key is listed with a leading `# ` and keeps its description:

```
.env
   2  DATABASE_URL  Postgres connection string
   5  PORT          Port the API listens on
   8  # DEBUG       Verbose request logging
```

`envn ls` shows the effective environment, so the disabled key is absent:

```
Key           Value
DATABASE_URL  postgres://localhost/app
PORT          8080
```

`envn ls --all` shows the file as written, with every occurrence, its line,
its value, and its description:

```
Line  Key           Value                     Description
   2  DATABASE_URL  postgres://localhost/app  Postgres connection string
   5  PORT          8080                      Port the API listens on
   8  # DEBUG       true                      Verbose request logging
```

To add an alternative database, append a second occurrence. It lands directly
under the first and, being last, becomes the effective value:

```sh
envn set --append DATABASE_URL postgres://staging.internal/app -e 'Staging database'
envn get DATABASE_URL
# postgres://staging.internal/app
```

```sh
# Postgres connection string
DATABASE_URL=postgres://localhost/app
# Staging database
DATABASE_URL=postgres://staging.internal/app
```

`envn use` switches between them. With no selector it refuses and lists the
choices; with a value it enables that occurrence and disables the rest:

```sh
envn use DATABASE_URL
# envn: .env: "DATABASE_URL" has 2 occurrences; pass a VALUE or --line
# Line  Key           Value                            Description
#    2  DATABASE_URL  postgres://localhost/app         Postgres connection string
#    4  DATABASE_URL  postgres://staging.internal/app  Staging database

envn use DATABASE_URL postgres://localhost/app
```

```sh
# Postgres connection string
DATABASE_URL=postgres://localhost/app
# Staging database
# DATABASE_URL=postgres://staging.internal/app
```

A commented-out line that still reads as `# KEY=value` is treated as a
*disabled* key rather than as comment text. It is hidden from `get` and `ls`,
but `explain` and the `--all` listings show it with a leading `# `, the comment
block above it stays attached to it, `set` re-enables it in place, and `remove`
deletes it.

`--all` means the same thing everywhere: every occurrence in file order,
disabled included. Without it, `get`, `ls`, and `explain` describe the
effective environment, where the last active occurrence of a key wins.

### Alternative values

A key may appear more than once, typically one active value and some disabled
alternatives. `set --append` adds an occurrence after the key's last one; it is
active and shadows the previous value unless `--disabled` is given. `use`
switches: the selected occurrence is enabled and every other one disabled.

Commands that act on one occurrence (`use`, `remove`, `enable`, `disable`,
`toggle`) select it by VALUE or by `--line N` using the numbers from
`ls --all`. When a key has several occurrences and no selector is given,
`use`, `remove`, and `enable` refuse, print the occurrences, and change
nothing, because guessing would either destroy data or create silent
shadowing. `disable` without a selector disables every active occurrence,
since turning a key off is unambiguous. `remove --all` deletes every
occurrence.

`format` moves every later occurrence of a key, active or disabled, up to sit
after its first occurrence. Occurrences keep their relative order and each
keeps its own comment block, so a file with `DB=1` near the top and a disabled
`# DB=3` with its own description further down ends up with both together.
Nothing else is reordered or rewritten.

`set` creates the file if it does not exist; `get`, `ls`, and `remove` error
if it is missing. Missing keys exit non-zero.
