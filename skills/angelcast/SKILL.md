---
name: angelcast
description: Make screen-free, kid-safe audio episodes (MP3) for a specific child from a topic, a lesson file, or a request like "make Mia an episode about volcanoes." Use when a parent asks for an audio episode, a podcast for their kid, a bedtime story, a lesson turned into audio, or a series/daily subscription. Requires the `angelcast` CLI on PATH and a logged-in family session.
metadata:
  openclaw:
    requires:
      bins: ["angelcast"]
    install:
      - kind: brew
        formula: myangel-ai/tap/angelcast
        os: ["darwin", "linux"]
      - kind: download
        url: https://github.com/myangel-ai/angelcast-cli/releases/latest/download/angelcast-x86_64-unknown-linux-musl.tar.gz
        sha256: "FILL_AT_RELEASE"
        archive: tar.gz
        os: ["linux"]
---

# angelcast — audio episodes for kids, from the CLI

`angelcast` turns a topic into a short two-host audio episode written for one
child's age, checks the script with KidRails (AngelQ's child-safety layer),
voices it, and gives you an MP3. Generation runs on AngelCast's API; the CLI
is the client. Every command accepts `--format json`. Nothing prompts
interactively unless `--i` is passed, so never pass `--i`.

## Before you start

1. Confirm the binary and session:
   ```
   angelcast --version
   angelcast whoami --format json
   ```
   A 401 means no session. Tell the parent to run
   `angelcast family login --email <email>` (or `angelcast family onboard …`
   for a new family) and stop. Do not attempt logins yourself; you do not
   hold the parent's credentials.
2. List the children so you use the right `member_id`:
   ```
   angelcast family members --format json
   ```
   Match on first name. If two children match or none match, ask the parent.

## Make one episode

1. Write the prompt. Rules:
   - One topic per episode. Put the learning goal in plain words if there is
     one ("teach the difference between 6 and 9", "why the sky is blue").
   - If the parent gave you a lesson file, summarize it into a prompt of
     under 80 words: the concept, 2–3 key facts or vocabulary words, and the
     goal. Do not paste the whole lesson.
   - Never include the child's last name, address, school, or anything the
     parent has marked as off-limits for that child.
   - Do not add jokes, characters, or framing of your own; the show formats
     handle that.
2. Create it (returns immediately with a pending episode):
   ```
   angelcast podcast create --prompt "<prompt>" --audience one-kid --member-id <member_id> --format json
   ```
   Options:
   - `--audience family` (no `--member-id`) for an episode aimed at all kids
     in the family; `--audience multi-kid --member-id A --member-id B` for
     some of them.
   - `--show-format <fact-splat|case-closed|sports-report|good-news|around-the-fire|story|free-form>`
     to pin a format. Omit it to let the classifier choose (`app-decides`).
     Use the table below only when the parent asks for a specific feel.
3. Read the `id` from the JSON, then download when it's ready:
   ```
   angelcast podcast download <id> --wait --wait-timeout 600 -o <directory>/ --format json
   ```
   Generation usually takes a few minutes. The JSON line contains `title`,
   `path`, and `bytes`. Put the file where the parent keeps their episodes
   (their vault, a shared folder, a Slack channel), not in a temp directory.
4. Report back with the title, the path, and the prompt you used. Do not
   play, publish, upload, or hand the episode to a child's device unless the
   parent has asked you to do that for this family. Default is: the parent
   listens first.

## Show formats (only when the parent asks for a feel)

| Format | Use for |
|---|---|
| `fact-splat` | surprising facts, science, how things work |
| `case-closed` | mysteries, puzzles, a whodunit (not for kids who scare easily) |
| `sports-report` | sports news, players, a game recap |
| `good-news` | uplifting news, animals, kindness stories |
| `around-the-fire` | history, battles, explorers, big events told as a story |
| `story` | a bedtime or original story |
| `free-form` | anything that doesn't fit above |

## Series and daily episodes

- A linked run of episodes on one theme (memory carries across episodes):
  ```
  angelcast series create --prompt "<theme and goal>" --audience one-kid --member-id <id> --format json
  angelcast series podcasts <series_id> --format json
  ```
  `series create` blocks for tens of seconds while the planner runs.
- New episodes on a topic every day, no create step:
  ```
  angelcast subscription create --topic "<topic>" --audience one-kid --member-id <id> --format json
  angelcast subscription list --format json
  angelcast subscription unsubscribe <series_id>
  ```
  Only create a subscription when the parent explicitly asks for recurring
  episodes; it keeps generating until unsubscribed.

## Errors and limits

- `401`: session missing or expired → tell the parent to log in (see above).
- `weekly_limit_reached`: the family has used its 36 episodes for the rolling
  week. Tell the parent (`angelcast family usage` shows when it resets); do
  not retry.
- Status `failed` or `timed_out` on download: retry the download once with
  `--wait`. If it fails again, report the id to the parent.
- `podcast delete`, `series delete`, and `family remove-member` are
  destructive. Never run them unless the parent names the exact item.
- Do not use any `admin` subcommand.

## Privacy

The CLI sends the prompt, the child's first name and birth month, and receives
the script and audio. Do not send anything else about the child. If the parent
asks what AngelCast keeps, point them to PRIVACY.md in the repo.
