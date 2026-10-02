---
name: angelcast
description: Walk a parent from zero to kid-safe audio episodes with the angelcast CLI - install it, onboard or log in the family, turn any input (a PDF, a teacher's email, a travel plan, a book, local places, a plain topic) into episodes, and deliver the MP3s to a folder or straight onto a Yoto player. Use when someone mentions angelcast, AngelCast, a podcast or bedtime story for their kid, turning a lesson or trip into audio, or putting episodes on a Yoto.
---

# angelcast: episodes for kids, from the terminal

`angelcast` turns a prompt into a short two-host audio episode written for a
child's age, checks the script against AngelQ's kid-safety rules, voices it,
and hands back an MP3. You drive the CLI for the parent, end to end: install,
account, episodes, delivery. Generation takes a few minutes per episode.

Always pass `--format json`. Never pass `--i` (it prompts interactively).
Errors print `error: <command> failed (<status>): <tag>` on stderr and exit 1;
usage mistakes exit 2.

## 1. Install

Check first: `angelcast --version`. If it is missing:

- Homebrew present (`command -v brew`): `brew install myangel-ai/tap/angelcast`
- Otherwise: `curl -fsSL https://github.com/myangel-ai/angelcast-cli/releases/latest/download/angelcast-cli-installer.sh | sh`
- Debian/Ubuntu users who prefer a package: the `.deb` on
  https://github.com/myangel-ai/angelcast-cli/releases/latest

macOS and Linux (x86_64 and arm64) only. No Windows build; suggest WSL.
Re-run `angelcast --version` to confirm.

## 2. Account

`angelcast whoami --format json`. A 401 means no session.

**Terms of Use.** Before running `family onboard` or `family login`, show the
parent the link https://www.angelq.ai/terms-of-use and ask them directly
whether they accept the angelq Terms of Use. You never accept for them. Pass
`--accept-terms` only after the parent answers with an explicit yes. A no, or
no answer, means stop: do not onboard or log in.

**New family.** Ask for: family (last) name, email, and for the first child a
first name, birth month (1-12) and birth year. Nothing else is needed.

The CLI needs a login password. At a terminal `family onboard` prompts for
it, which fails in an agent shell, so generate one and pipe it with
`--password-stdin`; the password never touches the command line:

```
PW=$(openssl rand -base64 18 | tr -d '/+=')
printf '%s\n' "$PW" | angelcast family onboard --family-name <Name> --email <email> \
  --password-stdin --accept-terms --child-name <First> --child-birth-month <M> --child-birth-year <YYYY> --format json | tail -n +2
echo "$PW"
```

Show the password once and tell the parent to store it; it is their
`family login` password from now on. If the parent would rather choose it,
print `angelcast family onboard` for them to run in their own terminal; it
walks them through every field and prompts for the password.
`tail -n +2` skips the session cookie printed on the first line; the family
JSON with each member's `id` follows, and the session is saved.

**Existing app account.** `printf '%s\n' "$PW" | angelcast family login --email <email> --password-stdin --accept-terms`.
If it says `This account has no password yet; contact support@angelq.ai to set one.`,
pass that on to the parent; stop there.

**More kids.** `angelcast family add-member --first-name <First> --birth-month <M> --birth-year <YYYY> --format json`.

Then `angelcast family members --format json` and keep the `id` for each child.

## 3. Turn the input into episodes

The input can be anything: a PDF, a teacher's email, an itinerary, a book, a
list of local places, a topic. Read it yourself (Read handles PDFs) and write
the prompts. Do not ask the parent to approve a plan; ask only when something
is missing (which child, how many episodes, a folder). Then create.

Prompt rules:

- One topic per episode, under 80 words. State the concept, two or three key
  facts or vocabulary words, and the learning goal if there is one. Never
  paste the source; distil it.
- Never include the child's last name, address, school, or anything the
  parent marked off-limits.
- Do not add jokes, characters, or framing; the show formats do that.
- Interests the parent mentioned can shape examples ("uses a soccer example").

Audience: `--audience one-kid --member-id <id>` for one child,
`--audience multi-kid --member-id A --member-id B` for some,
`--audience family` (no member ids) for everyone.

Show format: omit `--show-format` unless the parent asks for a feel.

| Format | Use for |
|---|---|
| `fact-splat` | surprising facts, science, how things work |
| `case-closed` | mysteries, puzzles, a whodunit (not for kids who scare easily) |
| `sports-report` | sports news, players, a game recap |
| `good-news` | uplifting news, animals, kindness |
| `around-the-fire` | history, explorers, big events told as a story |
| `story` | a bedtime or original story |
| `free-form` | anything else |

**Series or separate podcasts.** Decide, and say why in one line:

- Series when episodes should build on each other and are heard in order: a
  book chapter by chapter, a road trip stop by stop, a unit of lessons.
  `angelcast series create --prompt "<theme, order, and goal>" --audience ... --format json`
  blocks for tens of seconds while the planner runs, then
  `angelcast series podcasts <series_id> --format json` lists the episodes.
- Separate podcasts when the topics stand alone: five local landmarks, a
  handful of questions from a teacher's email.
  `angelcast podcast create --prompt "<prompt>" --audience ... --format json`
  returns at once with a pending podcast and its `id`.

Daily subscriptions (`angelcast subscription create`) exist only in preview
builds and keep generating until unsubscribed. Mention them only if the parent
asks for recurring episodes and the command is present in `angelcast --help`.

**Limits, up front.** Run `angelcast family usage --format json` before a
batch. A family gets 36 episodes per rolling 7 days, and at most 2 podcast
creates or 1 series create per 5 minutes. Tell the parent how many fit and
roughly how long the batch takes (about 5 minutes per pair of podcasts), then
pace it yourself: after every second `podcast create`, sleep until
`podcast_burst.retry_after` seconds have passed (or 300 seconds). Downloads do
not count. A create past the burst limit prints
`error: rate limited: 2 of 2 in the last 5 minutes; retry in <n>s`; sleep that
long and retry once. On `weekly_limit_reached`, stop and report `resets_at`;
do not retry.

## 4. Deliver

Ask once per session where episodes should go, offering both:

**A folder.** Suggest `~/AngelCast/<topic-slug>/`, create it, and download each
episode as it finishes:

```
angelcast podcast download <id> --wait --wait-timeout 900 -o <folder>/ --format json
```

The JSON line carries `title`, `path`, and `bytes`. A `failed` or `timed_out`
podcast errors before anything is written: retry the download once with
`--wait`; if it fails again, report the id.

**A Yoto player.** Download to a folder first (the parent keeps the MP3 either
way), then follow `references/yoto.md`: it uploads each MP3 to a "Make Your
Own" playlist on my.yotoplay.com through the Playwright browser tools, using
the parent's own Yoto login. It needs the Playwright MCP server; if the
`mcp__playwright__*` tools are absent, give the parent the install line in
that file and offer the folder path meanwhile.

Report back with each episode's title and path (or Yoto track number), plus
the prompt you used. Never auto-play; mention `angelcast podcast play <id>` as
an option.

## Destructive commands

`podcast delete`, `series delete`, `family remove-member`, and `logout` are
destructive. Run them only when the parent names the exact item in this
conversation. Do not use any `admin` subcommand.

## Privacy

The CLI sends the prompt, the child's first name and birth month and year, and
receives the script and audio. Send nothing else about the child. For what
AngelCast keeps, point the parent to PRIVACY.md in the angelcast-cli repo.
