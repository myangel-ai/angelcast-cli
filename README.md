# angelcast

**Angelcast from the terminal.** Generate, schedule, and play fun, personalized, kid-safe podcasts made by [AngelQ](https://angelq.ai) for your family, with JSON output built for agents and scripts.

[![Release](https://img.shields.io/github/v/release/myangel-ai/angelcast-cli)](https://github.com/myangel-ai/angelcast-cli/releases)
[![Build](https://github.com/myangel-ai/angelcast-cli/actions/workflows/release.yml/badge.svg)](https://github.com/myangel-ai/angelcast-cli/actions)
[![License](https://img.shields.io/github/license/myangel-ai/angelcast-cli)](LICENSE)

![angelcast demo: install, sign in, create an episode about volcanoes, play it](docs/demo.gif)

## Install

```sh
curl -fsSL https://angelq.ai/install.sh | sh
```

```sh
brew install myangel-ai/tap/angelcast
```

Debian and Ubuntu users can grab the `.deb` from the [latest release](https://github.com/myangel-ai/angelcast-cli/releases/latest). macOS and Linux (x86_64 and arm64) are supported.

## Your first episode

```sh
# 1. Sign up. You get a magic link by email; no password to remember.
angelcast family onboard --family-name Smith --email smith@example.com

# 2. Tell Angelcast who is listening. Age shapes the vocabulary, pacing, and topics.
angelcast family add-member --first-name Ada --birth-month 4 --birth-year 2018

# 3. Ask for an episode. Any question a kid would ask works.
angelcast podcast create --prompt "Why do volcanoes erupt?"

# 4. Listen. Episodes take a few minutes to generate; --wait holds until it is ready.
angelcast podcast play <podcast-id> --wait
```

Already have an account from the app? Skip step 1 and run `angelcast family login --email you@example.com`.

Out of ideas? `angelcast podcast topic-suggestions` returns prompts picked for your kids' ages and interests.

## What you can do

- **Create an episode from a prompt.** For one kid, several kids, or the whole family, in show formats like `story` and `fact-splat`.
- **Subscribe to a topic.** `angelcast subscription create --topic dinosaurs` delivers fresh episodes every week.
- **Play or download.** `podcast play` streams through your system player; `podcast download -o ./episodes/` keeps the mp3.
- **Personalize.** `family update --customization "Ada loves frogs, space, and silly voices"` steers every future episode.
- **Track listening.** `podcast playbacks` shows what was played, where, and when.
- **Script everything.** Every command takes `--format json` and returns stable exit codes.

The full reference is in [docs/CLI.md](docs/CLI.md), or run `angelcast docs` for the same text offline.

## Use it from an agent

Angelcast is built to be driven by scripts and AI agents as much as by people. Output is JSON on request, errors are structured, and exit codes are stable.

```sh
# Create an episode and capture its id
id=$(angelcast podcast create --prompt "How do bees make honey?" --format json | jq -r .podcast_id)

# Play it once it is ready
angelcast podcast play "$id" --wait
```

Using Claude Code, Codex, or another coding agent? Add this to your project instructions and the agent can run the CLI on your behalf:

```markdown
The `angelcast` CLI is installed. Use `--format json` for every call and `angelcast docs` for the reference.
Never run `podcast delete` or `family remove-member` without confirming with me first.
```

Errors come back as `{"error": "<code>", "message": "..."}` with a non-zero exit code. Hitting the usage cap returns `weekly_limit_reached`; `angelcast family usage` shows what is left and when it resets.

## How it works, and why it is safe

Every episode is made by [AngelQ](https://angelq.ai), the same engine behind the Angelcast app. It writes for the specific child: their age sets the vocabulary and pacing, and the interests you add shape the stories and examples. Every script is checked against AngelQ's kid-safety rules before it is voiced, so nothing reaches your kids that you would not want them to hear.

## Limits

Each family can generate up to 36 episodes per rolling 7 days. Plans and pricing are at [angelq.ai](https://angelq.ai).

## Help

- Full command reference: [docs/CLI.md](docs/CLI.md)
- Bugs and feature requests: [issues](https://github.com/myangel-ai/angelcast-cli/issues)
- Everything else: hello@angelq.ai

If your kids loved an episode, tell us with #angelcast. It is the best way to help other families find this.

## We build safety in the open

AngelQ co-created [KidRails](https://github.com/arcee-ai/KidRails) with [Arcee AI](https://www.arcee.ai): an open-source, child-safe language model and the training data and evals behind it, released so anyone can audit how age-appropriate AI should behave. The same thinking shapes every Angelcast episode.

## License

[Apache 2.0](LICENSE). Angelcast and AngelQ are trademarks of AngelKids AI; the license does not grant permission to use them.
