---
name: agenix
description: "Use agenix — an engine for building agents that improve themselves: write a profile, keep the agent's knowledge as files, run it on any executor (claude-code, codex, cline, aider, goose, …), evaluate it, teach it, and grow it. Use when the repo has a profile.yaml with a `route:`, when the user mentions agenix or its CLI, or when work involves running one agent across several executors or models."
---

# Using agenix

agenix separates three things that are usually tangled together: **what an agent
knows** (files you can read and diff), **what runs it** (an executor CLI —
claude-code, codex, cline, aider, goose, opencode, openhands, qwen-code,
gemini-cli, grok-cli, mini-swe-agent, swe-agent), and **how it reaches a model**
(a provider and a credential). Swap any one without touching the other two.

This is the shared reference; each agent tool wraps this content in its own
format.

## Install

```bash
npm install -g @fedtuna-ai/agenix     # the CLI is `agenix`
agenix init my-agent && cd my-agent   # profile.yaml + knowledge/ + suites/
agenix doctor -p profile.yaml         # says what is missing before you run
```

The executor is a separate program and agenix does not install it. `agenix
agents` lists the ones it knows; install the one you want the normal way
(`npm i -g @anthropic-ai/claude-code`, `npm i -g @openai/codex`, `npm i -g
cline`, …).

## Rules agents get wrong — read first

1. **A model id names its provider: `<provider>/<model>`.** The provider table
   already knows the address, and writing a second one beside it is an error:
   ```yaml
   route: { executor: cline, model: deepseek/deepseek-v4-pro }        # RIGHT
   # route: { executor: cline, model: deepseek/deepseek-v4-pro,
   #          baseUrl: https://api.deepseek.com/v1 }                   # WRONG — refused
   ```
   For an endpoint the engine cannot know — a proxy, a vLLM, LM Studio, GitHub
   Models — there is exactly one form, and it needs the address **including the
   path the server actually serves on**:
   ```yaml
   route: { executor: cline, model: openai-compatible/qwen2.5-coder:14b,
            baseUrl: http://localhost:11434/v1 }     # Ollama's OpenAI surface is /v1
   ```
2. **`builtin` is not a model name.** It means "whatever this executor's own
   login runs" — a ChatGPT plan for codex, a Claude plan for claude-code. A bare
   id with no provider (`sonnet`, `opus`, `gemini-2.5-pro`) means the executor's
   own model too, and is passed through to it.
3. **Under `auth.mode: subscription`, a stored token is used only if the profile
   names it.** Otherwise the executor's own login file is in charge, and if
   there isn't one you get "Not logged in" from a profile that looks configured:
   ```yaml
   route:
     executor: claude-code
     model: opus
     auth: { mode: subscription, credential: claude-code }   # names the stored token
   ```
4. **`-x/--executor` swaps the executor and nothing else.** The profile's auth
   mode stays. Swapping a subscription profile onto an api-key executor fails
   with "has no subscription auth" — write a small profile instead of fighting
   the flag.
5. **Keys never travel in argv.** `agenix auth set <name>` reads the key from
   stdin. Profiles name credentials; they never contain them.
6. **`agenix run` needs `-m`**, and `agenix models` needs `-a` even when you
   pass `-p`.
7. **With layered knowledge, writes land in the LAST layer.** Everything below
   it is read-only by construction, not by file permissions.
8. **A timeout is about silence, not duration.** `--timeout` is how long an
   executor may say *nothing* before it is treated as wedged; work that keeps
   reporting keeps going. `--ceiling` (a day by default) is the backstop for
   something that talks without getting anywhere.

## The profile

One file. Everything a run needs, and nothing about a particular machine.

```yaml
name: my-agent
route:
  executor: claude-code          # agenix agents
  model: opus                    # <provider>/<model>, `builtin`, or the executor's own
  auth:
    mode: subscription           # inherit | subscription | api-key | env
    credential: claude-code      # the stored entry, when a token is used
  reasoning: high                # low|medium|high|xhigh|max|ultra, where supported
  extraArgs: ["--allowedTools", "Bash"]   # passed to the executor as-is
  env: { HTTP_PROXY: "…" }       # extra environment for the child
mode: dev                        # dev | user — what may be taught (see policy.yaml)
knowledge: ./knowledge           # one directory, or a list of layers
recall: false                    # give the executor a `recall` MCP tool over its own Mind
mcpServers:
  - name: filesystem
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "."]
```

`auth.mode`:

| mode | what it means |
|---|---|
| `inherit` | inject nothing; the executor's ambient environment is in charge |
| `subscription` | the executor's own plan login (plus a stored token when named) |
| `api-key` | a key from the credential store, named by `credential:` |
| `env` | a key already in the environment, named by `apiKeyEnv:` |

## The knowledge directory

Plain files. This is the whole point: what the agent knows is reviewable,
diffable and revertible.

| file | what it holds |
|---|---|
| `constitution.md` | how it works, in prose. Given, never learned |
| `persona.yaml` | who it is |
| `rules.yaml` | imperatives, scoped `domain` / `persona` / `style` |
| `facts.yaml` | what is true, with tags |
| `skills/<name>.skill.md` | procedures: frontmatter + prose |
| `episodes.jsonl` | what happened, appended |
| `goals.yaml` | what it is for. Given, never learned |
| `competence.yaml` | what it must be able to do |
| `world.yaml` | the machines, datasets and repositories it works with. Given |
| `policy.yaml` | what may be taught or evolved in each mode |

A skill is a file, not a paragraph:

```markdown
---
name: profile-a-dataset
description: one line, what this is for
triggers: [when it applies]
steps: [one step per entry]
invariants: [what must always hold]
---

Then the prose, in as much detail as the reader needs.
```

**Layers.** `knowledge:` may be a list. Later layers override earlier ones by id,
and writes go to the last:

```yaml
knowledge:
  - /opt/base-mind          # shipped, replaced whole on upgrade
  - /var/lib/local-mind     # what this instance learned — survives
```

## Commands

**Running.**
```bash
agenix run -p profile.yaml -m "the task"          # one turn in a temp workspace
agenix run -p profile.yaml -m "…" -w ./work       # in a directory you keep
agenix chat -p profile.yaml                       # interactive, same TUI for every executor
agenix chat -p profile.yaml --native              # drop into the executor's own TUI
agenix compile -p profile.yaml -o ./out           # materialize the executor's native config
```
`-x/--executor`, `--model`, `--base-url`, `-r/--reasoning` override the route for
one invocation.

**Credentials and logins.** Two different things: a *credential* is a key you
store; a *login* is the executor signing into its own account.
```bash
agenix auth set openrouter     # reads the key from stdin
agenix auth list               # names only — values are never printed
agenix auth login codex        # device code, a token, or the tool's own UI
agenix auth status             # per executor: signed in, how, and whether it still works
```

**Knowing what it can run.**
```bash
agenix agents                              # executors, and how each authenticates
agenix models -a cline -p profile.yaml     # asked of the executor, live
agenix doctor -p profile.yaml              # binary, auth, knowledge, gateway
```

**Teaching and evaluating.**
```bash
agenix teach -p profile.yaml "always report the exact command you ran"
agenix eval -p profile.yaml -s suites/smoke.yaml
agenix audit -s suites/smoke.yaml          # can a task pass WITHOUT doing the work?
agenix compare -p a.yaml -p b.yaml -s suites/smoke.yaml   # paired A/B with a sign test
```

A suite task is a prompt and a checker — a shell command that must succeed in the
workspace afterwards:

```yaml
name: smoke
tasks:
  - id: hello-file
    prompt: Create hello.txt containing exactly the line "hello agenix".
    checker: grep -qx "hello agenix" hello.txt
    timeoutS: 180
```

**Growing.** A round is: the agent decides what to take up, does it, writes down
what it learned, another executor judges that writing and sets tasks from it, a
weaker executor tries those tasks on the changed Mind, and the judge accepts or
sends it back.

```bash
agenix grow -p profile.yaml                            # the agent chooses
agenix grow -p profile.yaml --task "master EDA on tabular data"
agenix grow -p profile.yaml --journal ./round.jsonl    # for something else to watch
agenix serve -p profile.yaml                           # a face for it, on the loopback
```

The parts are named in the profile, and are deliberately different from each
other and from the chat route:

```yaml
evolution:
  worker: { executor: claude-code, model: opus, auth: { mode: subscription, credential: claude-code } }
  second: { executor: codex, model: gpt-5.6-sol, reasoning: high, auth: { mode: subscription } }
  weak:
    - { executor: cline, model: openrouter/mistralai/ministral-8b-2512, auth: { mode: api-key }, credential: openrouter }
    - { executor: cline, model: openrouter/mistralai/ministral-3b-2512, auth: { mode: api-key }, credential: openrouter }
```

`agenix serve` has no authentication by design: it binds to the loopback and is
reached from elsewhere through an ssh tunnel.
```bash
ssh -N -L 7777:127.0.0.1:7777 <host>    # then open http://127.0.0.1:7777
```

**Recording what happened.**
```bash
agenix usage -p profile.yaml --software flower --data cifar10 --server lattice-a --outcome ok
agenix feedback -p profile.yaml --score 0.8 --note "found the leak, missed the imbalance"
```

## Recipes

**The same agent on a different executor, to see what changes:**
```bash
agenix run -p profile.yaml -x codex --model builtin -m "the task"
```

**A local model through Ollama.** Its OpenAI surface is under `/v1`, and the
provider table's address is not — so name it as a custom endpoint:
```yaml
route: { executor: cline, model: openai-compatible/qwen2.5-coder:14b, baseUrl: http://localhost:11434/v1 }
```

**A gateway (LiteLLM, on-prem proxy)** for every executor at once: point the
route at `openai-compatible/<model>` with the proxy's address, and store the
gateway's key under a name.

**Teach from inside a session.** Every session is told: when the user says
"remember that …", persist it by running
```bash
agenix teach -p "$AGENIX_PROFILE" "<the statement>"
```
which becomes a reviewable commit in the knowledge directory. Four executors
(claude, codex, gemini, qwen) also get a native `/teach` command.

## When something does not work

| what you see | what it means |
|---|---|
| `route: Required` | the profile is in the pre-`route:` shape — `agent:`/`model: {id}` moved under `route:` |
| `openai-compatible needs a baseUrl` | a custom endpoint with no address |
| `"deepseek" has an address of its own` | a baseUrl beside a table provider — drop the baseUrl |
| `a bare model has no endpoint` | `model: qwen2.5-coder` + a baseUrl — write `openai-compatible/qwen2.5-coder` |
| `Not logged in` from a subscription profile | no stored token named, and no executor login file — `agenix auth login <executor>`, or name the credential |
| `has no subscription auth` | `-x` swapped to an api-key executor while the profile says subscription |
| `credential "x" not found` | `agenix auth set x` |
| `said nothing for Ns` | the executor went silent and was treated as wedged; raise `--timeout` only if it was genuinely thinking |
| the executor's shell calls are all refused | headless executors deny their own tools until told otherwise — claude needs `extraArgs: ["--allowedTools", "Bash"]`, codex `["--sandbox", "danger-full-access"]`, gemini/qwen `--yolo` |
