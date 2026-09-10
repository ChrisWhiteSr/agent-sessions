# agent-sessions

One command that lists every Claude Code and Codex session on your machine, newest first, with the directory each one belongs to. Pick one and resume it from wherever you are.

```
$ sessions
  tool    age  id             directory                            title
  claude   0m  3559607e-d083  ~                                5p  So I'm having a problem using this Linux server and trigger…
  claude  33m  a5729fbe-4182  …e/agent_comms/Writing/scaffolding  15p  is there a difference between these two essays?
  codex    1d  01a07ef0-5cfc  ~/Code                           2p  Evaluate and improve docloop repos
  codex    2d  01a08148-1193  ~/Code/docloop                   2p  can you show me session topics from
  claude   2d  24f7534b-92dd  ~/Code/docloop                   5p  Are there any agents running in this repo right now trying…
  … 35 more (-a for all, -n N for more)

40 of 79 sessions   (headless runs hidden, add -H)
flags
  -s TERM          search prompts and replies for TERM (case-insensitive)
  -d PATH          only sessions whose directory contains PATH
  ...
```

The table sizes itself to the terminal. Directory and title columns grow on a wide window and shrink on a narrow one, with the directory trimmed from the left so the project name survives. The flag guide at the bottom is always printed, so you never have to remember the options. Output is coloured when stdout is a terminal. Set `NO_COLOR=1` or pass `--no-color` to turn that off.

## The problem

Claude Code keeps one transcript folder per working directory. The `/resume` picker only shows sessions from the directory you launched it in. Codex filters its picker by directory too. If you run agents from a dozen project folders, finding last Tuesday's conversation means remembering where you were sitting when you had it.

It gets worse if you script these tools. Every `claude -p` call and every `codex exec` run leaves a transcript behind. A grading pipeline that fires off fifty one-shot prompts buries the three sessions you actually typed into.

## What it does

- Reads the transcript files directly from `~/.claude/projects` and `~/.codex/sessions`. No API calls, no daemon.
- Shows tool, age, session id, directory, prompt count, and the first prompt as a title.
- Hides headless runs by default. A Claude session counts as interactive if its id shows up in `~/.claude/history.jsonl`, which only records prompts typed into the terminal UI. A Codex session counts as interactive if it was started from the TUI or VS Code rather than `codex exec`.
- Searches the actual conversations. `sessions -s presquared` returns every session where you or the agent said "presquared", with a hit count and the first matching line. Tool output, file contents and injected system context are excluded from the search, so a word sitting in a memory file does not match every session.
- Resumes a session from any directory by changing into its original folder and calling `claude -r` or `codex resume` with the full id.

Listing runs in well under a second on a few hundred transcripts. A full-text search over 230 MB of transcripts takes about a second. Python 3 standard library only.

## Install

```
git clone https://github.com/ChrisWhiteSr/agent-sessions.git
ln -s "$PWD/agent-sessions/sessions" ~/bin/sessions
```

Or copy the single `sessions` file anywhere on your PATH. Requires Python 3.8 or newer.

## Usage

```
sessions                     40 most recent sessions, newest first
sessions -n 100              show up to 100
sessions -a                  show every session
sessions -s presquared       sessions whose prompts or replies mention "presquared"
sessions -d docloop          only directories whose path contains "docloop"
sessions -t claude           Claude Code only (or -t codex)
sessions -H                  include headless runs, marked with *
sessions --by-dir            group by directory, 3 per directory (-n changes the count)
sessions --min-prompts 3     hide sessions with fewer than 3 prompts
sessions --no-color          plain output

sessions resume 01a081       cd into that session's directory and resume it
sessions cmd 01a081          print the shell command instead of running it
```

Ids can be abbreviated to any unique prefix. The table shows 13 characters because Codex uses time-ordered ids and two sessions started in the same second share the first 8.

### Searching

```
$ sessions -s presquared
  tool    age  id             directory       title
  claude   1d  2b75e1a4-8119  ~            3p  Remember how you've been able to use the AWS CLI tools? I n…
                                    40 hits  …can delete production DNS for presquared.com and three other domains) and…
  codex   17d  01a033e1-8140  ~/Code       5p  NexSys agent services strategy and website plan
                                     5 hits  …egist agent | Generalize from PreSquared to NexSys services | | Content Ex…
```

The match is case-insensitive and applies to user prompts and assistant replies. Combine with `-d`, `-t`, or `-H` to narrow it further. The search term is highlighted in the title and snippet.

### Resuming

`sessions resume` replaces itself with the agent process, so when you quit the agent you land back in the shell you started from. If you would rather see the command first:

```
$ sessions cmd a5729fbe
cd '/home/varmint/Code/agent_comms/Writing/scaffolding' && claude -r a5729fbe-4182-4437-944d-433ae71907b4
```

## How it reads the files

Claude Code writes one `<uuid>.jsonl` per session under `~/.claude/projects/<encoded-cwd>/`. Each line is a JSON record. The script scans the first few hundred lines for the `cwd` field and the first user message that is not a system attachment or a slash command.

Codex writes `rollout-<timestamp>-<uuid>.jsonl` under `~/.codex/sessions/YYYY/MM/DD/`. The first line is a `session_meta` record with the id, working directory, and the originator that started it. Thread names come from `~/.codex/session_index.jsonl` when Codex has assigned one.

Last activity is the file's modification time. Titles and directories are cut to whatever fits the current terminal width, so rows never wrap.

Search does a cheap substring check on each raw line before parsing it as JSON, then only counts matches inside user and assistant text. That keeps a full scan fast and keeps tool output and system context out of the results.

## Limits

- Only looks at the local machine and the default config locations. `CLAUDE_CONFIG_DIR` and `CODEX_HOME` overrides are not read yet.
- Sessions spawned into a real terminal by another tool look interactive, because their first prompt went through the TUI. Use `--min-prompts 2` to thin those out.
- Subagent transcripts (Claude's `agent-*.jsonl` files in subfolders) are skipped on purpose.

## License

MIT
