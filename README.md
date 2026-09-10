# agent-sessions

One command that lists every Claude Code and Codex session on your machine, newest first, with the directory each one belongs to. Pick one and resume it from wherever you are.

```
$ sessions
  #   tool    age  id             directory                            title
  1   claude   0m  3559607e-d083  ~                               10p  So I'm having a problem using this Linux server and trigger…
  2   claude   3h  a5729fbe-4182  …e/agent_comms/Writing/scaffolding  16p  is there a difference between these two essays?
  3   codex    2d  01a07ef0-5cfc  ~/Code                           5p  Evaluate and improve docloop repos
  4 s claude   3d  2f74da63-2f7b  ~/Code/docloop-surface           2p  You are building docloop M2/M3/M4 (rubric + loop + surface)…
  5   codex    3d  01a08148-1193  ~/Code/docloop                   2p  can you show me session topics from
     … 35 more (-a for all, -n N for more)

40 of 77 sessions   (headless hidden, add -H)
flags
  -s TERM          search prompts and replies for TERM (case-insensitive)
  -d PATH          only sessions whose directory contains PATH
  ...

$ sessions 2          # resumes row 2, from any directory
```

The table sizes itself to the terminal. Directory and title columns grow on a wide window and shrink on a narrow one, with the directory trimmed from the left so the project name survives. The flag guide at the bottom is always printed, so you never have to remember the options. Output is coloured when stdout is a terminal. Set `NO_COLOR=1` or pass `--no-color` to turn that off.

## The problem

Claude Code keeps one transcript folder per working directory. The `/resume` picker only shows sessions from the directory you launched it in. Codex filters its picker by directory too. If you run agents from a dozen project folders, finding last Tuesday's conversation means remembering where you were sitting when you had it.

It gets worse if you script these tools. Every `claude -p` call and every `codex exec` run leaves a transcript behind. A grading pipeline that fires off fifty one-shot prompts buries the three sessions you actually typed into.

## What it does

- Reads the transcript files directly from `~/.claude/projects` and `~/.codex/sessions`. No API calls, no daemon.
- Shows tool, age, session id, directory, prompt count, and the first prompt as a title.
- Numbers every row. `sessions 3` resumes row 3 of the last listing, and `sessions show 3` prints its opening prompts and replies. `sessions pick` opens fzf with a transcript preview when fzf is installed.
- Hides headless runs by default. A Claude session counts as interactive if its id shows up in `~/.claude/history.jsonl`, which only records prompts typed into the terminal UI. A Codex session counts as interactive if it was started from the TUI or VS Code rather than `codex exec`.
- Tells your sessions apart from agent briefs. A first prompt that opens with "You are the…" or "Act as…" was almost certainly handed to an agent by a script or another agent, even if it went through the terminal UI. Those rows carry an `s` marker, and `-m` hides them.
- Searches the actual conversations. `sessions -s presquared` returns every session where you or the agent said "presquared", with a hit count and the first matching line. Tool output, file contents and injected system context are excluded from the search, so a word sitting in a memory file does not match every session.
- Resumes a session from any directory by changing into its original folder and calling `claude -r` or `codex resume` with the full id.

The first run reads every transcript once and caches what it learned under `~/.cache/agent-sessions/`. After that a listing takes well under a tenth of a second, and only files that changed get re-read. A full-text search over 230 MB of transcripts takes about a second. Python 3 standard library only.

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
sessions -m                  mine only: hide agent-spawned briefs (marked s)
sessions -s presquared       sessions whose prompts or replies mention "presquared"
sessions -d docloop          only directories whose path contains "docloop"
sessions -t claude           Claude Code only (or -t codex)
sessions -H                  include headless runs, marked with *
sessions --by-dir            group by directory, 3 per directory (-n changes the count)
sessions --min-prompts 3     hide sessions with fewer than 3 prompts
sessions --no-color          plain output

sessions 3                   resume row 3 of the last listing
sessions resume 01a081       resume by id prefix (or row number)
sessions cmd 01a081          print the shell command instead of running it
sessions show 01a081         print the first prompts and replies
sessions pick                fuzzy-pick with fzf, preview shows the transcript
sessions refresh             rebuild the metadata cache
```

Row numbers refer to whatever listing you printed last, filters included, so `sessions -s presquared` followed by `sessions 2` resumes the second search hit. Ids can be abbreviated to any unique prefix. The table shows 13 characters because Codex uses time-ordered ids and two sessions started in the same second share the first 8.

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

`sessions resume` replaces itself with the agent process, so when you quit the agent you land back in the shell you started from. If the session's directory has since been deleted, it warns you and resumes from your home directory instead, which works because both tools find a session by id. If you would rather see the command first:

```
$ sessions cmd a5729fbe
cd '/home/varmint/Code/agent_comms/Writing/scaffolding' && claude -r a5729fbe-4182-4437-944d-433ae71907b4
```

## How it reads the files

Claude Code writes one `<uuid>.jsonl` per session under `~/.claude/projects/<encoded-cwd>/`. Each line is a JSON record. The script scans the first few hundred lines for the `cwd` field and the first user message that is not a system attachment or a slash command.

Codex writes `rollout-<timestamp>-<uuid>.jsonl` under `~/.codex/sessions/YYYY/MM/DD/`. The first line is a `session_meta` record with the id, working directory, and the originator that started it. Thread names come from `~/.codex/session_index.jsonl` when Codex has assigned one.

Last activity is the timestamp of the last record in the file, not the file's modification time. Claude Code rewrites the header records of old transcripts from time to time, which would otherwise float month-old sessions to the top. Titles and directories are cut to whatever fits the current terminal width, so rows never wrap.

The cache lives at `~/.cache/agent-sessions/index.json` (or under `XDG_CACHE_HOME`), keyed by file path, size and mtime. Delete it or run `sessions refresh` if it ever looks wrong.

Search does a cheap substring check on each raw line before parsing it as JSON, then only counts matches inside user and assistant text. That keeps a full scan fast and keeps tool output and system context out of the results.

## Limits

- Only looks at the local machine. `CLAUDE_CONFIG_DIR` and `CODEX_HOME` are honoured if set.
- The agent-brief detection is a pattern match on the first prompt. A brief that opens some other way will show as yours, and a prompt you typed that happens to start with "You are" will be marked `s`.
- Subagent transcripts (Claude's `agent-*.jsonl` files in subfolders) are skipped on purpose.

## License

MIT
