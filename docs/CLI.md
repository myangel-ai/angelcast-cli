# angelcast CLI — Command Reference

The command line over the same endpoints as [`docs/API.md`](API.md). The crate
is `angelcast-cli`; the **binary is `angelcast`**.

```bash
angelcast --help                                         # installed
cargo run -p angelcast-cli -- --help                     # customer build, from the repo
cargo run -p angelcast-cli --features admin -- --help    # admin build
```

`make cli-customer` / `make cli-admin` (or `make cli` for both) `cargo install` the
two flavours and link them into `~/.cargo/bin` as `acc` and `aca`.

Examples use `angelcast`; prefix with `cargo run -p angelcast-cli --` if you
haven't installed it.

Human output and `--help` are coloured when stdout is a terminal. Piped output,
`--format json`, `angelcast docs`, and any run with `NO_COLOR` set are plain
text; `CLICOLOR_FORCE=1` keeps colour when piping.

The command reference below is **generated**: it is what `angelcast docs`
prints in the **customer build**, built from the same clap definitions as
`--help`, so a customer with only the binary has the same reference. Every command's `--help` also ends
with example invocations. The prose around it — session and error details the
binary does not explain — is hand-written.

> **Regenerating.** `cargo test -p angelcast-cli` (without `--features admin`)
> fails while the generated section is stale. After changing a command, run
> `scripts/update_cli_docs.sh` (or `UPDATE_CLI_DOCS=1 cargo test -p
> angelcast-cli cli_md_matches`) and commit the result. Never edit between the
> markers by hand.

---

## Three builds: customer, `preview`, and `admin`

The crate has two Cargo features, both off by default. The default
(**customer**) build is what gets released; ops builds with
`cargo build -p angelcast-cli --features admin`, which also turns on
`preview`. Everything below is compiled out of the customer build — absent from `--help`, from `angelcast docs`, and
from the generated reference in this file, and rejected by the parser as an
unknown subcommand or argument, not merely hidden:

- the `admin` command group (`admin login`, `admin create`,
  `admin change-password`) and the `host` command group (`host list`,
  `host add`, `host use`);
- `family list`;
- every `--family-id` flag (a customer build always acts on the session's own
  family; an admin session has no family and must pass it, naming the family
  that owns the row on `podcast`/`series`/member commands);
- `podcast record-playback` (`podcast download` still records its own
  `downloaded` playback in every build — that is the only way a customer build
  records one).

The customer build targets `https://api.angelcast.rocks` by default and seeds
only the `prod` alias — no `dev` or `local`. `ANGELCAST_HOST` (and `--host`)
still override it, so support can point a customer at another host without the
`host` command. The admin build defaults to `dev`
(`https://dev-api.angelcast.rocks`) and seeds `dev`/`local` too. Both builds
read the same `config.yml`, so a customer build on a machine that already has
one honours its active host.

To see the admin build's reference, run `angelcast docs` from an admin build;
it is not checked in. Passwords on the `admin` commands are plain argv, so
they land in shell history and the process list.

CI (`.github/workflows/cli.yml`) builds and tests the crate both ways, so a
missed `cfg` fails the PR rather than an ops build.

### Preview commands

`preview` holds customer features that are merged but not yet in the public
release, so the team can use them for real before deciding they are ready.
Unlike `admin` surface, a preview command is written for customers and ships
once its `#[cfg(feature = "preview")]` is deleted. `cli/build.rs` lists the
held-back surface and prints a `warning:` line for each when `preview` is
off, so a release build's log shows exactly what it left out. Currently
behind it: the `subscription` command group, `series create
--subscription`, and `family update --customization`.

## Releasing

Releases are built by [cargo-dist](https://opensource.axo.dev/cargo-dist/)
from `dist-workspace.toml` and published to the public
`myangel-ai/angelcast-cli` repo; this repo's source never ships. To cut one:

1. Bump `version` in `cli/Cargo.toml` and merge it.
2. Tag that commit `angelcast-cli-v<version>` and push the tag. The version
   must match the crate or the workflow refuses to run.

`.github/workflows/angelcast-cli--release.yml` then builds the customer
binary for macOS (arm64, x86_64) and Linux (musl, arm64 and x86_64),
`build-deb.yml` adds the two `.deb` packages, and the release job attaches
everything plus `angelcast-cli-installer.sh` to a GitHub Release on the
public repo and commits `Formula/angelcast.rb` to `myangel-ai/homebrew-tap`.
A `-rc` suffix (`angelcast-cli-v1.1.0-rc1`) produces a release marked
pre-release, which `releases/latest` and the tap both ignore.

Auth: build jobs assume the AWS deploy role for the private angel registry
(`dist-build-setup.yml`); the release and tap jobs mint a token from the
`angelcast-release` GitHub App (`RELEASE_APP_ID`, `RELEASE_APP_PRIVATE_KEY`).
The workflow is hand-edited after `dist generate` for those steps and to
add the AWS step to every job that runs `dist`, so
`allow-dirty = ["ci"]` is set; reapply the edits if you rerun `dist init`.

---

## Sessions

`family login`, `family onboard`, and `admin login` (admin build only) save the
session cookie they receive, so later commands against the same host send it
automatically:

```bash
angelcast admin login --email ops@example.com --password hunter2
angelcast family list
```

- Sessions live in `sessions.yml` next to `config.yml` (see
  [`angelcast host`](#angelcast-host)), keyed by the **resolved host URL** — an
  alias and its literal URL share one entry, and `dev`/`prod` sessions coexist.
  The file is written with mode `0600`.
- Precedence for the cookie sent: `--session`, then `ANGELCAST_SESSION`, then
  the stored entry for the resolved host. Login commands still print the cookie
  as their first stdout line, so `export ANGELCAST_SESSION="$(angelcast admin
  login …)"` pipelines keep working.
- The stored entry carries the cookie's `Max-Age`/`Expires`; an expired entry is
  treated as absent.
- A **401** prints `error: unauthorized for <host>: not logged in or session
  expired; run \`angelcast family login\`` and exits 1 (the admin build's
  message also names `angelcast admin login` and the admin-session case). A cookie that was actually
  sent is never deleted — the server also answers 401 when a family session
  hits an admin-only endpoint — so use `logout` to drop a rejected one. The
  only automatic cleanup is pruning a stored entry that had already expired
  locally (nothing was sent). `admin login`, `family login`,
  `family set-password`, and `admin change-password` are exempt: their 401
  means wrong credentials and is reported as the plain
  `<context> failed (401 Unauthorized)` — except `family login`, which says
  `wrong email or password`, or `no CLI password set for this email; contact
  support@angelq.ai to get one` when the family has none yet.
- **Passwords never go on the command line.** `family login`,
  `family onboard --password`, and `family set-password` prompt with echo off,
  or read the password from stdin (one per line, in prompt order) with
  `--password-stdin` for scripts — so nothing lands in `ps` or shell history.
  The admin-build `admin` commands still take `--password <value>`.
- `angelcast logout` forgets the resolved host's entry; `logout --all` forgets
  every host.

## Errors

Errors print to stderr as `error: <context> failed (<status>): <tag>` and exit
**1** — e.g. `error: family update failed (409 Conflict): email_in_use`. The
tag is the server's domain-error string; an untagged body is shown raw.
The one exception is the weekly usage cap (429 `weekly_limit_reached`), which
prints `error: weekly podcast limit reached (36 of 36 in the last 7 days); a
slot frees at <resets_at>` — see [Usage cap](#usage-cap).

---

<!-- BEGIN GENERATED COMMAND REFERENCE -->
<!-- Generated by `angelcast docs` from the CLI's clap definitions. Do not edit by hand:
     run `scripts/update_cli_docs.sh` after changing a command. -->

Personalized kids' podcasts from the terminal: onboard a family, generate episodes and series, and download the audio.

Every command talks to one host, https://api.angelcast.rocks unless --host or ANGELCAST_HOST says otherwise.

`family onboard` and `family login` remember the cookie they receive per host, so later commands against the same host send it automatically; `--session` and ANGELCAST_SESSION override it. A 401 means no session or an expired one — run `angelcast family login` again, or `angelcast logout` to drop a rejected one.

## Global flags

Accepted before or after any subcommand.

| Argument | Description |
|---|---|
| `--format <FORMAT>` | Output format; `json` prints the raw response body. One of `human`, `json`. |
| `--host <HOST>` | Host alias (e.g. prod) or a raw base URL. Falls back to ANGELCAST_HOST, then the active host in the config file. |
| `--session <SESSION>` | Session cookie for authenticated requests. Falls back to ANGELCAST_SESSION, then the session stored by the last login for this host. |

## Commands

| Command | Description |
|---|---|
| [`angelcast family`](#angelcast-family) | Onboard, log in, and manage your family and its members |
| [`angelcast podcast`](#angelcast-podcast) | Generate podcast episodes and fetch their audio |
| [`angelcast series`](#angelcast-series) | Plan multi-episode series |
| [`angelcast docs`](#angelcast-docs) | Print the command reference as Markdown |
| [`angelcast logout`](#angelcast-logout) | Forget the stored session for the resolved host (or every host with --all) |
| [`angelcast whoami`](#angelcast-whoami) | Show who the current session is (GET /sessions/me) |

Examples:

```bash
angelcast family onboard --family-name Smith --email smith@example.com
angelcast podcast create --prompt "Why do frogs change color?"
angelcast podcast download <podcast-id> --wait
angelcast docs
```

## `angelcast family`

Alias: `fam`

Onboard, log in, and manage your family and its members.

```
Usage: angelcast family [OPTIONS] <COMMAND>
```

| Subcommand | Description |
|---|---|
| [`onboard`](#angelcast-family-onboard) | Onboard a new family (POST /onboarding); saves the session for this host and prints the cookie + family |
| [`login`](#angelcast-family-login) | Log in with email and CLI password (POST /sessions/cli); saves the session for this host and prints the cookie |
| [`set-password`](#angelcast-family-set-password) | Change the family's CLI password (PUT /family/password). Prompts for the current password, then the new one; with `--password-stdin` they are read as two lines from stdin, current first |
| [`get`](#angelcast-family-get) | Get the session's family (GET /family) |
| [`update`](#angelcast-family-update) | Update the session's family (PATCH /family). Omitted fields are unchanged |
| [`usage`](#angelcast-family-usage) | Show the family's podcast generation usage: the rolling 7-day cap and the short-window burst counters (GET /family/usage) |
| [`members`](#angelcast-family-members) | List the family's members (GET /family/members); use it to discover member ids |
| [`add-member`](#angelcast-family-add-member) | Add a member to the family (POST /family/members) |
| [`update-member`](#angelcast-family-update-member) | Update a member (PATCH /family/members/{member_id}) |
| [`remove-member`](#angelcast-family-remove-member) | Remove a member (DELETE /family/members/{member_id}). Destructive |

### `angelcast family onboard`

Onboard a new family (POST /onboarding); saves the session for this host and prints the cookie + family.

The child is optional — omit all three child flags to create a family with no members and add them later with `add-member`. If you give any one of them, you must give all three.

The session cookie is the first stdout line, ahead of the family, so a `--format json` pipeline needs `tail -n +2`.

The password is the CLI login password for `family login`. `--password` takes no value: the password is prompted for, or read from stdin with `--password-stdin`, so it never appears in `ps` or shell history.

```
Usage: angelcast family onboard [OPTIONS] --family-name <FAMILY_NAME> --email <EMAIL> <--password|--password-stdin>
```

| Argument | Description |
|---|---|
| `--family-name <FAMILY_NAME>` **(required)** | The family's (last) name. |
| `--email <EMAIL>` **(required)** | Contact/login email. |
| `--password` | Set a CLI login password; prompts for it (takes no value). Conflicts with `--password-stdin`. |
| `--password-stdin` | Set a CLI login password read as one line from stdin. |
| `--child-name <CHILD_NAME>` | First name of the first child. |
| `--child-birth-month <CHILD_BIRTH_MONTH>` | Birth month (1–12) of the first child. |
| `--child-birth-year <CHILD_BIRTH_YEAR>` | Birth year of the first child, e.g. 2018; no earlier than 90 years ago and not in the future. |

Examples:

```bash
angelcast family onboard --family-name Smith --email smith@example.com --password
printf '%s\n' "$PW" | angelcast family onboard --family-name Smith --email smith@example.com --password-stdin
angelcast family onboard --family-name Smith --email smith@example.com --password --child-name Ada --child-birth-month 4 --child-birth-year 2018
```

### `angelcast family login`

Log in with email and CLI password (POST /sessions/cli); saves the session for this host and prints the cookie.

The password is prompted for (never passed on the command line), or read as one line from stdin with `--password-stdin`. A family that has no CLI password yet is told to contact support@angelq.ai.

The cookie is the first stdout line, ahead of the payload.

```
Usage: angelcast family login [OPTIONS] --email <EMAIL>
```

| Argument | Description |
|---|---|
| `--email <EMAIL>` **(required)** | The family's login email. |
| `--password-stdin` | Read the password as one line from stdin instead of prompting. |

Examples:

```bash
angelcast family login --email smith@example.com
printf '%s\n' "$PW" | angelcast family login --email smith@example.com --password-stdin
export ANGELCAST_SESSION="$(angelcast family login --email smith@example.com | head -n 1)"
```

### `angelcast family set-password`

Change the family's CLI password (PUT /family/password). Prompts for the current password, then the new one; with `--password-stdin` they are read as two lines from stdin, current first.

```
Usage: angelcast family set-password [OPTIONS]
```

| Argument | Description |
|---|---|
| `--password-stdin` | Read the password(s) from stdin, one per line, instead of prompting. |

Examples:

```bash
angelcast family set-password
printf '%s\n%s\n' "$OLD" "$NEW" | angelcast family set-password --password-stdin
```

### `angelcast family get`

Get the session's family (GET /family).

```
Usage: angelcast family get [OPTIONS]
```

Examples:

```bash
angelcast family get
angelcast family get --format json
```

### `angelcast family update`

Update the session's family (PATCH /family). Omitted fields are unchanged.

```
Usage: angelcast family update [OPTIONS]
```

| Argument | Description |
|---|---|
| `--last-name <LAST_NAME>` | New family last name. |
| `--email <EMAIL>` | New contact/login email. |
| `--clear-customization` | Clear the customization prompt (sends an explicit null). |

Examples:

```bash
angelcast family update --last-name Smith-Jones
angelcast family update --clear-customization
```

### `angelcast family usage`

Show the family's podcast generation usage: the rolling 7-day cap and the short-window burst counters (GET /family/usage).

```
Usage: angelcast family usage [OPTIONS]
```

Examples:

```bash
angelcast family usage
angelcast family usage --format json
```

### `angelcast family members`

List the family's members (GET /family/members); use it to discover member ids.

```
Usage: angelcast family members [OPTIONS]
```

Examples:

```bash
angelcast family members
```

### `angelcast family add-member`

Add a member to the family (POST /family/members).

```
Usage: angelcast family add-member [OPTIONS] --first-name <FIRST_NAME>
```

| Argument | Description |
|---|---|
| `--first-name <FIRST_NAME>` **(required)** | The member's first name. |
| `--birth-month <BIRTH_MONTH>` | Birth month (1–12). Must be given together with --birth-year. |
| `--birth-year <BIRTH_YEAR>` | Birth year, e.g. 2018; no earlier than 90 years ago and not in the future. Must be given together with --birth-month. |

Examples:

```bash
angelcast family add-member --first-name Ada --birth-month 4 --birth-year 2018
angelcast family add-member --first-name Sam
```

### `angelcast family update-member`

Update a member (PATCH /family/members/{member_id}).

A full replace, not a partial update: `--first-name` is required, and omitting `--birth-month`/`--birth-year` clears the stored birth date.

```
Usage: angelcast family update-member [OPTIONS] --member-id <MEMBER_ID> --first-name <FIRST_NAME>
```

| Argument | Description |
|---|---|
| `--member-id <MEMBER_ID>` **(required)** | Member id, as printed by `family members`. |
| `--first-name <FIRST_NAME>` **(required)** | The member's first name. |
| `--birth-month <BIRTH_MONTH>` | Birth month (1–12). Must be given together with --birth-year. |
| `--birth-year <BIRTH_YEAR>` | Birth year, e.g. 2018; no earlier than 90 years ago and not in the future. Must be given together with --birth-month. |

Examples:

```bash
angelcast family update-member --member-id <member-id> --first-name Ada --birth-month 4 --birth-year 2018
```

### `angelcast family remove-member`

Remove a member (DELETE /family/members/{member_id}). Destructive.

```
Usage: angelcast family remove-member [OPTIONS] --member-id <MEMBER_ID>
```

| Argument | Description |
|---|---|
| `--member-id <MEMBER_ID>` **(required)** | Member id, as printed by `family members`. |

Examples:

```bash
angelcast family remove-member --member-id <member-id>
```

## `angelcast podcast`

Alias: `pod`

Generate podcast episodes and fetch their audio.

The audience flags are shared with `series create`. `--audience family` needs no members and forbids `--member-id` and `--i`; `one-kid` takes exactly one `--member-id`; `multi-kid` takes several. `--i` (or `--interactive`) picks the members from a checklist of the family's members instead. Without `--audience` the episode is not personalized to particular members.

```
Usage: angelcast podcast [OPTIONS] <COMMAND>
```

| Subcommand | Description |
|---|---|
| [`create`](#angelcast-podcast-create) | Start generating a podcast (POST /podcasts/async) |
| [`create-ftue`](#angelcast-podcast-create-ftue) | Generate the first-time-user episode (POST /podcasts/ftue) |
| [`topic-suggestions`](#angelcast-podcast-topic-suggestions) | Suggest episode topics for the family (POST /podcasts/topic-suggestions) |
| [`get`](#angelcast-podcast-get) | Show one podcast (GET /podcasts/{id}) |
| [`delete`](#angelcast-podcast-delete) | Delete a podcast (DELETE /podcasts/{id}). Destructive |
| [`outputs`](#angelcast-podcast-outputs) | Print a finished podcast's flow outputs (GET /podcasts/{id}/outputs) |
| [`download`](#angelcast-podcast-download) | Save a podcast's mp3 to disk and record a `downloaded` playback |
| [`play`](#angelcast-podcast-play) | Play a podcast through a system audio player and record the listen |
| [`list`](#angelcast-podcast-list) | List the family's podcasts (GET /podcasts) |
| [`repoll`](#angelcast-podcast-repoll) | Poll angelq once for this podcast and print it (POST /podcasts/{id}/repoll) |
| [`refresh`](#angelcast-podcast-refresh) | Poll angelq once for every in-flight podcast and print them (POST /podcasts/refresh) |
| [`playbacks`](#angelcast-podcast-playbacks) | Print a podcast's playback history, newest first (GET /podcasts/{id}/playbacks) |

### `angelcast podcast create`

Start generating a podcast (POST /podcasts/async).

Never waits: prints the `pending` podcast, id included, while the server runs the flow and updates the row. Follow up with `download --wait` to block until the audio is ready, or `get` / `repoll` / `refresh` to check on it.

```
Usage: angelcast podcast create [OPTIONS] --prompt <PROMPT>
```

| Argument | Description |
|---|---|
| `--prompt <PROMPT>` **(required)** | What the episode should be about. |
| `--audience <AUDIENCE>` | Who the episode is for. `family` reads the family from GET /family and forbids --member-id/--i; `one-kid` takes exactly one member. One of `one-kid`, `multi-kid`, `family`. |
| `--member-id <MEMBER_ID>` | A member the episode is for; repeat for several (with --audience one-kid or multi-kid). Repeatable. |
| `--i`, `--interactive` | Pick the members from a checklist instead of passing --member-id. Spelled --i (two dashes), or --interactive. |
| `--show-format <SHOW_FORMAT>` | Pin the show for this episode. Omitted means app-decides: the classifier picks. One of `app-decides`, `fact-splat`, `case-closed`, `sports-report`, `good-news`, `around-the-fire`, `story`, `free-form`. |

Examples:

```bash
angelcast podcast create --prompt "Why do frogs change color?"
angelcast podcast create --prompt "Sharks!" --audience one-kid --member-id <member-id> --show-format fact-splat
angelcast podcast create --prompt "A bedtime story about a brave snail" --audience multi-kid --i
angelcast podcast create --prompt "Our weekend plans" --audience family
```

### `angelcast podcast create-ftue`

Generate the first-time-user episode (POST /podcasts/ftue).

Always async, like `create`. With --member-id (or --i to pick one) the episode is personalized for that kid; the first time user experience flow only takes one kid; without it the flow personalizes on the prompt alone.

```
Usage: angelcast podcast create-ftue [OPTIONS] --prompt <PROMPT>
```

| Argument | Description |
|---|---|
| `--member-id <MEMBER_ID>` | The kid the episode is for. |
| `--i`, `--interactive` | Pick the kid from a list instead of passing --member-id. Spelled --i (two dashes), or --interactive. Conflicts with `--member-id`. |
| `--prompt <PROMPT>` **(required)** | The curated first-run topic. |

Examples:

```bash
angelcast podcast create-ftue --prompt "outer space" --member-id <member-id>
angelcast podcast create-ftue --prompt "outer space" --i
angelcast podcast create-ftue --prompt "outer space"
```

### `angelcast podcast topic-suggestions`

Suggest episode topics for the family (POST /podcasts/topic-suggestions).

```
Usage: angelcast podcast topic-suggestions [OPTIONS]
```

| Argument | Description |
|---|---|
| `--n-ideas <N_IDEAS>` | How many ideas to ask for. |

Examples:

```bash
angelcast podcast topic-suggestions
angelcast podcast topic-suggestions --n-ideas 5 --format json
```

### `angelcast podcast get`

Show one podcast (GET /podcasts/{id}).

```
Usage: angelcast podcast get [OPTIONS] <PODCAST_ID>
```

| Argument | Description |
|---|---|
| `<PODCAST_ID>` | Podcast id, as printed by `create` or `list`. |

Examples:

```bash
angelcast podcast get <podcast-id>
angelcast podcast get <podcast-id> --format json
```

### `angelcast podcast delete`

Delete a podcast (DELETE /podcasts/{id}). Destructive.

```
Usage: angelcast podcast delete [OPTIONS] <PODCAST_ID>
```

| Argument | Description |
|---|---|
| `<PODCAST_ID>` | Podcast id, as printed by `create` or `list`. |

Examples:

```bash
angelcast podcast delete <podcast-id>
```

### `angelcast podcast outputs`

Print a finished podcast's flow outputs (GET /podcasts/{id}/outputs).

The `mp3_output` value is the base64-encoded audio; `download` decodes it for you.

```
Usage: angelcast podcast outputs [OPTIONS] <PODCAST_ID>
```

| Argument | Description |
|---|---|
| `<PODCAST_ID>` | Podcast id, as printed by `create` or `list`. |

Examples:

```bash
angelcast podcast outputs <podcast-id>
angelcast podcast outputs <podcast-id> --format json
```

### `angelcast podcast download`

Save a podcast's mp3 to disk and record a `downloaded` playback.

Fetches GET /podcasts/{id}/outputs, writes the `mp3_output`, then posts a `downloaded` playback (method `cli`) — the command that fetches an episode is what marks it downloaded. A `failed` or `timed_out` podcast errors before anything is written. If the mp3 lands but the playback POST fails the file is kept: the failure is a stderr warning, `recorded` is `false` in the JSON, and the exit code stays 0. The JSON line carries `title` and `path`, so a pipeline can chain `create` into `download --wait` and read both off the last line.

```
Usage: angelcast podcast download [OPTIONS] <PODCAST_ID>
```

| Argument | Description |
|---|---|
| `<PODCAST_ID>` | Podcast id, as printed by `create` or `list`. |
| `-o <OUTPUT>`, `--output <OUTPUT>` | A file path, or an existing directory to drop `<title-slug>.mp3` into (`<podcast-id>.mp3` when the episode has no title). Defaults to the working directory; an existing file is overwritten. |
| `--wait` | Poll GET /podcasts/{id} every 5s until the podcast leaves `waiting`/`pending`. Without it a still-generating podcast is an error. |
| `--wait-timeout <SECONDS>` | Give up on --wait after this many seconds. Default: `600`. |

Examples:

```bash
angelcast podcast download <podcast-id> --wait
angelcast podcast download <podcast-id> -o ./episodes/ --format json
```

### `angelcast podcast play`

Play a podcast through a system audio player and record the listen.

Fetches GET /podcasts/{id}/outputs into a temp file, hands it to the player (afplay on macOS, else the first of mpg123, ffplay, mpv on PATH), then deletes the file unless -o is given. Posts a `started-listening` playback when the player launches and a `completed-listening` one when it exits cleanly; ctrl-c stops the player and records no completion. A failed playback POST is a stderr warning, not an error. A `failed` or `timed_out` podcast errors before anything is written. If you want to download and not listen, use download command.

```
Usage: angelcast podcast play [OPTIONS] <PODCAST_ID>
```

| Argument | Description |
|---|---|
| `<PODCAST_ID>` | Podcast id, as printed by `create` or `list`. |
| `-o <OUTPUT>`, `--output <OUTPUT>` | Keep the mp3: a file path, or an existing directory to drop `<title-slug>.mp3` into. Without it the file is deleted after playing. |
| `--player <COMMAND>` | Player command (name or path) to run with the mp3 path as its only argument, instead of the built-in candidates. |
| `--wait` | Poll GET /podcasts/{id} every 5s until the podcast leaves `waiting`/`pending`. Without it a still-generating podcast is an error. |
| `--wait-timeout <SECONDS>` | Give up on --wait after this many seconds. Default: `600`. |
| `--family-id <FAMILY_ID>` | Target family (admin only; a family session always uses its own). |

Examples:

```bash
angelcast podcast play <podcast-id>
angelcast podcast play <podcast-id> --wait
angelcast podcast play <podcast-id> -o ./episodes/
```

### `angelcast podcast list`

List the family's podcasts (GET /podcasts).

```
Usage: angelcast podcast list [OPTIONS]
```

Examples:

```bash
angelcast podcast list
angelcast podcast list --format json
```

### `angelcast podcast repoll`

Poll angelq once for this podcast and print it (POST /podcasts/{id}/repoll).

```
Usage: angelcast podcast repoll [OPTIONS] <PODCAST_ID>
```

| Argument | Description |
|---|---|
| `<PODCAST_ID>` | Podcast id, as printed by `create` or `list`. |

Examples:

```bash
angelcast podcast repoll <podcast-id>
```

### `angelcast podcast refresh`

Poll angelq once for every in-flight podcast and print them (POST /podcasts/refresh).

```
Usage: angelcast podcast refresh [OPTIONS]
```

Examples:

```bash
angelcast podcast refresh
```

### `angelcast podcast playbacks`

Print a podcast's playback history, newest first (GET /podcasts/{id}/playbacks).

```
Usage: angelcast podcast playbacks [OPTIONS] <PODCAST_ID>
```

| Argument | Description |
|---|---|
| `<PODCAST_ID>` | Podcast id, as printed by `create` or `list`. |

Examples:

```bash
angelcast podcast playbacks <podcast-id>
```

## `angelcast series`

Alias: `ser`

Plan multi-episode series.

A series is planned up front and its episodes are generated in the background; `list` and `get` embed the episodes. The audience flags work as for `podcast create`.

```
Usage: angelcast series [OPTIONS] <COMMAND>
```

| Subcommand | Description |
|---|---|
| [`create`](#angelcast-series-create) | Plan a series and start generating its episodes (POST /series) |
| [`get`](#angelcast-series-get) | Show one series with its episodes (GET /series/{id}) |
| [`delete`](#angelcast-series-delete) | Delete a series and every episode in it (DELETE /series/{id}). Destructive |
| [`list`](#angelcast-series-list) | List the family's series with their episodes (GET /series) |
| [`podcasts`](#angelcast-series-podcasts) | List a series' episodes (GET /series/{id}/podcasts) |

### `angelcast series create`

Plan a series and start generating its episodes (POST /series).

Blocks while the planner runs, so it can take tens of seconds.

```
Usage: angelcast series create [OPTIONS] --prompt <PROMPT>
```

| Argument | Description |
|---|---|
| `--prompt <PROMPT>` **(required)** | What the series should be about. |
| `--audience <AUDIENCE>` | Who the series is for. `family` reads the family from GET /family and forbids --member-id/--i; `one-kid` takes exactly one member. One of `one-kid`, `multi-kid`, `family`. |
| `--member-id <MEMBER_ID>` | A member the series is for; repeat for several (with --audience one-kid or multi-kid). Repeatable. |
| `--i`, `--interactive` | Pick the members from a checklist instead of passing --member-id. Spelled --i (two dashes), or --interactive. |
| `--show-format <SHOW_FORMAT>` | Pin the show for every episode. Omitted means app-decides. One of `app-decides`, `fact-splat`, `case-closed`, `sports-report`, `good-news`, `around-the-fire`, `story`, `free-form`. |

Examples:

```bash
angelcast series create --prompt "Space adventures" --audience family
angelcast series create --prompt "Dinosaur detectives" --audience one-kid --member-id <member-id> --show-format story
```

### `angelcast series get`

Show one series with its episodes (GET /series/{id}).

```
Usage: angelcast series get [OPTIONS] <SERIES_ID>
```

| Argument | Description |
|---|---|
| `<SERIES_ID>` | Series id, as printed by `create` or `list`. |

Examples:

```bash
angelcast series get <series-id>
angelcast series get <series-id> --format json
```

### `angelcast series delete`

Delete a series and every episode in it (DELETE /series/{id}). Destructive.

```
Usage: angelcast series delete [OPTIONS] <SERIES_ID>
```

| Argument | Description |
|---|---|
| `<SERIES_ID>` | Series id, as printed by `create` or `list`. |

Examples:

```bash
angelcast series delete <series-id>
```

### `angelcast series list`

List the family's series with their episodes (GET /series).

```
Usage: angelcast series list [OPTIONS]
```

Examples:

```bash
angelcast series list
```

### `angelcast series podcasts`

Alias: `episodes`

List a series' episodes (GET /series/{id}/podcasts).

```
Usage: angelcast series podcasts [OPTIONS] <SERIES_ID>
```

| Argument | Description |
|---|---|
| `<SERIES_ID>` | Series id, as printed by `create` or `list`. |

Examples:

```bash
angelcast series podcasts <series-id>
angelcast series episodes <series-id> --format json
```

## `angelcast docs`

Print the command reference as Markdown.

Generated from the same definitions as `--help`, so it cannot drift from the binary. `docs/CLI.md` in the repo embeds this output.

```
Usage: angelcast docs [OPTIONS]
```

Examples:

```bash
angelcast docs | less
angelcast docs > angelcast-cli.md
```

## `angelcast logout`

Forget the stored session for the resolved host (or every host with --all).

Local only: the session is not invalidated server-side, and `--session` / ANGELCAST_SESSION values are unaffected. Honours `--host` like every other command, so `logout --host prod` leaves the `dev` session alone.

```
Usage: angelcast logout [OPTIONS]
```

| Argument | Description |
|---|---|
| `--all` | Forget the stored sessions for every host, not just the resolved one. |

Examples:

```bash
angelcast logout
angelcast logout --host prod
angelcast logout --all
```

## `angelcast whoami`

Alias: `me`

Show who the current session is (GET /sessions/me).

Prints e.g. `family <id> (prod, from stored login)`: the host — its alias when the URL matches one, else the URL — and where the cookie came from (`--session`, ANGELCAST_SESSION, or the stored login). `--format json` prints the server body alone. The server answers from the cookie itself, so a session whose row was deleted still reports an identity; not logged in is the usual 401.

```
Usage: angelcast whoami [OPTIONS]
```

Examples:

```bash
angelcast whoami
angelcast me --host prod
```

<!-- END GENERATED COMMAND REFERENCE -->

---

## Usage cap

`podcast create`, `podcast create-ftue`, and every episode planned by
`series create` count against the family's 36 podcasts per rolling 7 days
(prod). At the cap they fail with `error: weekly podcast limit reached (36 of
36 in the last 7 days); a slot frees at <resets_at>`. A family already at the
cap is rejected before the series planner runs; otherwise, if the planned
episodes don't all fit, `series create` fails after it with `error: weekly
podcast limit reached (34 of 36 in the last 7 days, this needs 5); the oldest
counted podcast leaves the window at <resets_at>` — no slot is promised, since
one row aging out may still leave too few. Either way nothing is inserted.

`family usage` prints `used=<n> limit=<n> remaining=<n> resets_at=<rfc3339>`,
or `used=<n> limit=uncapped` where the server has no cap configured.
`resets_at` is when the oldest counted podcast leaves the window (omitted when
nothing is counted). See "Usage cap" in `docs/API.md` for what counts.

---

## Repo extras

`angelcast podcast play <podcast-id>` is the way to listen from the CLI. The
older `scripts/play_podcast_mp3.py` still plays a saved outputs JSON/JSONL file
(the shape `podcast outputs --format json` prints), which is useful for flow
fixtures that never became a podcast row:

```bash
angelcast podcast outputs <podcast-id> --format json | scripts/play_podcast_mp3.py
```
