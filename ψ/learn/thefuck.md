# /learn thefuck

> Source: https://github.com/nvbn/thefuck
> Learned: 2026-03-26
> 170 correction rules, 7 shell integrations, ~90k stars

## What thefuck Is

A Python CLI tool that corrects previous console commands. You type a wrong command, get an error, type `fuck`, and thefuck proposes the corrected command. Simple concept, elegant architecture.

Inspired by a tweet: "I just mass-renamed a bunch of files to remove the spaces in file names. The command I used was: `fuck`."

## Architecture Overview

```
thefuck/
├── entrypoints/          # CLI entry points
│   ├── alias.py          # `thefuck --alias` → prints shell function
│   ├── fix_command.py    # Core: `fuck` → fix previous command
│   ├── shell_logger.py   # Instant mode logger
│   └── not_configured.py # First-run guidance
├── types.py              # Command, Rule, CorrectedCommand classes
├── corrector.py          # Rule loading + matching engine
├── conf.py               # Settings system (file + env + args)
├── shells/               # Shell-specific integration
│   ├── bash.py
│   ├── zsh.py
│   ├── fish.py
│   ├── powershell.py
│   ├── tcsh.py
│   └── generic.py        # Base class
├── rules/                # 170 correction rules
├── output_readers/       # Capture command output
├── specific/             # Tool-specific helpers (git, etc.)
├── ui.py                 # Interactive selection UI
└── utils.py              # Shared utilities
```

## The Correction Flow

```
1. User runs wrong command → gets error
2. User types `fuck` (or configured alias)
3. Shell function captures:
   - Previous command script (from history)
   - Command output (stderr + stdout)
4. Python engine receives Command(script, output)
5. Corrector loads all enabled rules, sorted by priority
6. Each rule's match(command) is tested
7. Matching rules yield CorrectedCommand(s) via get_new_command(command)
8. Commands are deduplicated and sorted by priority
9. UI presents options: [enter/↑/↓/ctrl+c]
10. Selected command runs, optionally added to history
```

## Core Types (types.py)

### Command
```python
class Command:
    script: str     # The command that was run (e.g., "git brnch")
    output: str     # Combined stderr+stdout output
    script_parts: list  # Shell-split command parts (lazy property)
```

Created via `Command.from_raw_script(raw_script)`:
- Formats the raw script
- Expands shell aliases via `shell.from_shell(script)`
- Captures output via `get_output(script, expanded)`

### Rule
```python
class Rule:
    name: str                    # Rule name (filename without .py)
    match: (Command) -> bool     # Does this rule apply?
    get_new_command: (Command) -> str|list  # What's the fix?
    enabled_by_default: bool     # Opt-in vs opt-out
    side_effect: callable|None   # Extra action on fix
    priority: int                # Lower = tried first
    requires_output: bool        # Skip if no output captured
```

Rules are loaded from Python files via `Rule.from_path(path)` using `importlib`.

### CorrectedCommand
```python
class CorrectedCommand:
    script: str           # The corrected command
    side_effect: callable # Optional side effect
    priority: int         # For ordering multiple fixes
```

Has a `run(old_cmd)` method that:
1. Executes side_effect if present
2. Adds to shell history if `alter_history` enabled
3. Writes corrected script to stdout (shell evals it)

## The Rule System

### Anatomy of a Rule

Every rule is a Python file with 2 required functions:

```python
def match(command):
    """Return True if this rule can fix the command."""
    return 'some error pattern' in command.output

def get_new_command(command):
    """Return the corrected command string (or list of strings)."""
    return 'fixed {}'.format(command.script)
```

Optional attributes:
- `enabled_by_default = True` — whether rule is active without explicit config
- `priority = 1000` — lower = higher priority (default: 1000)
- `side_effect = None` — function called when fix is applied
- `requires_output = True` — skip rule if no output captured

### Rule Examples

**sudo.py** — The most iconic rule:
```python
patterns = ['permission denied', 'eacces', 'must be root', ...]

def match(command):
    # Don't match if already using sudo
    if command.script_parts[0] == 'sudo':
        return False
    return any(p in command.output.lower() for p in patterns)

def get_new_command(command):
    return 'sudo {}'.format(command.script)
```

**git_push.py** — Context-aware git correction:
```python
@git_support  # Decorator handles git command normalization
def match(command):
    return ('push' in command.script_parts
            and 'git push --set-upstream' in command.output)

@git_support
def get_new_command(command):
    # Parse git's suggestion from the error output
    arguments = re.findall(r'git push (.*)', command.output)[-1]
    return replace_argument(command.script, 'push', 'push {}'.format(arguments))
```

**no_such_file.py** — Auto-create directories:
```python
patterns = (
    r"mv: cannot move .* to '([^']*)': No such file or directory",
    r"cp: cannot create regular file '([^']*)': No such file or directory",
)

def get_new_command(command):
    # Extract missing directory, create it, then retry
    dir = file[0:file.rfind('/')]
    return shell.and_('mkdir -p {}', '{}').format(dir, command.script)
```

### Rule Loading Pipeline (corrector.py)

```python
def get_rules():
    # 1. Collect paths from 3 sources:
    #    - Bundled: thefuck/rules/*.py
    #    - User: ~/.config/thefuck/rules/*.py
    #    - Third-party: thefuck_contrib_*/rules/*.py

    # 2. Load each .py file via importlib
    # 3. Filter: only enabled rules (checked against settings.rules)
    # 4. Sort by priority
    return sorted(loaded_rules, key=lambda r: r.priority)

def get_corrected_commands(command):
    # For each rule, if it matches, get its corrections
    # Deduplicate, sort by priority, yield
```

### Rule Count by Category

170 rules total, covering:
- **Git**: ~30 rules (push, pull, branch, checkout, stash, merge, rebase, etc.)
- **Package managers**: apt, brew, cargo, composer, conda, npm, pip, pacman, dnf, yum
- **Docker**: image, container, login, not_command
- **Python**: command, execute, module_error
- **Shell builtins**: cd, cp, mv, rm, chmod, mkdir, ln, history
- **Language tools**: cargo, go, java, gradle, lein, ruby
- **Cloud/infra**: aws, az, gcloud, heroku, vagrant
- **And more**: ssh, systemctl, tmux, open, man, etc.

## Shell Integration

### How It Works

`eval $(thefuck --alias)` prints a shell function that:

1. Captures shell aliases (`TF_SHELL_ALIASES`)
2. Captures recent history (`TF_HISTORY`)
3. Sets encoding (`PYTHONIOENCODING=utf-8`)
4. Runs `thefuck ARGUMENT_PLACEHOLDER $@`
5. Evals the output (`TF_CMD`)
6. Optionally adds fix to history

### Zsh Integration (zsh.py)

```bash
fuck () {
    TF_SHELL=zsh
    TF_SHELL_ALIASES=$(alias)
    TF_HISTORY="$(fc -ln -10)"
    TF_CMD=$(thefuck THEFUCK_ARGUMENT_PLACEHOLDER $@) && eval $TF_CMD
    test -n "$TF_CMD" && print -s $TF_CMD  # Add to history
}
```

### Instant Mode

An experimental mode that captures output in real-time (not after the fact):
1. First call spawns a new shell with `script` logging all output
2. Inserts invisible markers in PS1
3. On `fuck`, reads the log file instead of re-running the command
4. Much faster for slow commands

### Supported Shells

| Shell | File | Alias Eval | History Source |
|-------|------|-----------|---------------|
| Bash | bash.py | `eval $(thefuck --alias)` | `HISTFILE` |
| Zsh | zsh.py | `eval $(thefuck --alias)` | `fc -ln -10` |
| Fish | fish.py | `thefuck --alias \| source` | `builtin history` |
| PowerShell | powershell.py | `iex "$(thefuck --alias)"` | `Get-History` |
| Tcsh | tcsh.py | `eval \`thefuck --alias\`` | `history` |

## Configuration (conf.py)

Settings loaded in priority order: defaults → file → env → args.

### Settings File

`~/.config/thefuck/settings.py` (Python file):

```python
rules = ['ALL']              # Which rules to enable
exclude_rules = []           # Rules to disable
require_confirmation = True  # Ask before running fix
wait_command = 3             # Seconds to wait for output
no_colors = False
debug = False
alter_history = True         # Add fix to shell history
priority = {}                # Rule priority overrides
slow_commands = ['lein', 'react-native', 'gradle']
wait_slow_command = 15       # Extra wait for slow commands
num_close_matches = 3
instant_mode = False
```

### Environment Variables

All settings can be overridden via env: `THEFUCK_RULES`, `THEFUCK_REQUIRE_CONFIRMATION`, etc.

### User Rules Directory

`~/.config/thefuck/rules/` — drop a `.py` file here and it's automatically loaded as a rule. No registration needed.

## How to Write a New Rule

1. Create `thefuck/rules/my_rule.py`:

```python
def match(command):
    return 'specific error text' in command.output

def get_new_command(command):
    return command.script.replace('wrong', 'right')
```

2. Create test `tests/rules/test_my_rule.py`:

```python
import pytest
from thefuck.rules.my_rule import match, get_new_command
from thefuck.types import Command

@pytest.fixture
def output():
    return 'specific error text'

def test_match(output):
    assert match(Command('wrong', output))

def test_not_match():
    assert not match(Command('wrong', 'different output'))

def test_get_new_command(output):
    assert get_new_command(Command('wrong', output)) == 'right'
```

3. If needed, add to `enabled_by_default = False` for risky rules.

### Rule Writing Patterns

- **Pattern matching**: Use `re.search()` on `command.output` for complex patterns
- **Script parts**: Use `command.script_parts` for command argument analysis
- **Shell helpers**: Use `shell.and_()`, `shell.or_()` for composing commands
- **Git decorator**: Use `@git_support` for git subcommand rules
- **Multiple fixes**: Return a `list` from `get_new_command` for alternatives

## Key Insights

1. **Plugin architecture without registration** — Rules are just Python files in a directory. Drop a file, it works. No manifest, no registry, no imports. `importlib` + glob = plugin system.

2. **Shell-agnostic core, shell-specific edges** — The correction engine is pure Python. Shell integration is an adapter pattern (Generic base class, shell-specific subclasses).

3. **Output-driven matching** — Rules match on error output, not command syntax. This is why thefuck works across languages and tools — it reads what the tool says, not what the user typed.

4. **Graceful degradation** — If output capture fails, rules with `requires_output=True` are skipped. If a rule throws an exception, it's logged and skipped. The system never crashes on a bad rule.

5. **Community-driven growth** — 170 rules from hundreds of contributors. The simplicity of the rule API (just `match` + `get_new_command`) makes contribution trivial.

6. **The `sudo` pattern** — Match on permission error patterns, prepend `sudo`. Simple, but handles edge cases (already has sudo, uses `&&`, uses `>`). This is the most-used rule and a masterclass in handling real-world shell complexity.

---

*"Every typo is a lesson."*
*— Thefuck Community Oracle*
