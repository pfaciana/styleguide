# Bash Style Guide (Compact)

## Core Principles
- **Self-documenting code is MANDATORY** - write code so clear comments are unnecessary
- **Consistency over perfection** - when in doubt, be consistent
- **Explicit over implicit** - make choices clear
- **Safety first** - choose constructs that prevent errors
- **Idempotency** - scripts should run multiple times safely

## Shell & Version
- Default to Bash 4.4+; document other shells
- Shebang: `#!/usr/bin/env bash`
- File extension: `.sh` for executables
- Shell options: Always set `-uo pipefail`; decide on `-e` per project

## Formatting
- **Indentation**: Tabs (4-char wide), never spaces
- **Line length**: 120 chars max
- **Semicolons**: Only when required (`if true; then`, `while read; do`)
- **Spacing**: No trailing whitespace, newline at EOF, max 1 blank line
- **Heredocs**: Use for multi-line strings, avoid consecutive echo statements

## Comments
- **Avoid comments** - code must be self-documenting
- Use only when: explaining WHY (not WHAT), warning about dangerous operations (eval), documenting workarounds
- File header: Brief one-line description only
- Function comments: Rarely needed - use descriptive names instead
- TODO format: `TODO: description` or `TODO(name): description`

## Naming
- **Functions**: `function name() { }` - lowercase_with_underscores, highly descriptive, use both `function` keyword AND `()`
- **Variables**: lowercase_with_underscores, descriptive
- **Script arguments**: `_UPPERCASE` with defaults `${_VAR:=default}`
- **Constants**: UPPERCASE_WITH_UNDERSCORES, use `readonly`, can set conditionally then make readonly
- **Namespaces**: Use `::` separator (e.g., `mypackage::function_name`)

## Quoting
- **Prefer double quotes for ALL strings** (easier to add variables later)
- Single quotes only when needed: literal `$`, regex patterns
- Always quote variable expansions: `"${var}"` not `$var`
- Always quote `"$@"` for argument passing
- No quotes needed: inside `[[ ]]`, assignments, special vars (`$?`, `$#`, `$$`)
- Use braces for disambiguation: `"${USER}s"` not `"$USERs"`

## Tests & Conditionals
- Use `[[ ... ]]` never `[ ... ]` or `test`
- String tests: `-z` (empty), `-n` (non-empty), `==` (equality, preferred over `=`)
- Numeric comparison: Use `(( a > b ))` not `[[ $a -gt $b ]]`
- Block format: `then` on same line as `if`, `do` on same line as `while`/`for`

## Arithmetic
- Use `(( ... ))` and `$(( ... ))` exclusively
- Variables don't need `$` inside: `(( i = j + 1 ))` not `(( i = $j + 1 ))`
- Never use: `let`, `expr`, `$[...]`
- Spacing: `(( i += 3 ))` for readability

## Command Substitution & Expansion
- Use `$(command)` never backticks
- Prefer parameter expansion over external commands:
  - `${var#pattern}` remove shortest prefix
  - `${var##pattern}` remove longest prefix
  - `${var%pattern}` remove shortest suffix
  - `${var%%pattern}` remove longest suffix
  - `${var/pattern/replacement}` replace first
  - `${var//pattern/replacement}` replace all
  - `${var:-default}` use default if unset
  - `${var:=default}` assign default if unset

## Arrays
- Declare: `declare -a array` or `array=(item1 item2)`
- Always quote: `"${array[@]}"` for all elements
- Length: `${#array[@]}`
- Append: `array+=(item)`
- Never use space-separated strings for lists

## Loops & Iteration
- **For loops**: Use for arrays, globs, fixed ranges
  - Brace expansion: `for i in {1..5}; do`
  - C-style: `for (( i = 0; i < n; i++ )); do`
  - Glob: `for file in ./*.txt; do [[ -e "${file}" ]] || continue`
- **While read**: Use for line-by-line processing, streaming data
  - `while IFS= read -r line; do ... done < file`
  - Process substitution: `while read -r line; do ... done < <(command)`
- **Never**: `for item in $(command)` - loads all into memory, breaks on spaces

## read Builtin
- Use with IFS for parsing: `IFS=. read -r host domain tld <<< "${fqdn}"`
- CSV parsing: `while IFS=, read -r name email age; do ... done < file.csv`
- Avoid external commands (cut, awk) for simple field splitting

## Functions
- Structure: `function name() { local vars; logic; return code; }`
- Always use `local` for function variables
- Separate declaration from assignment when capturing exit codes:
  ```bash
  local result
  result=$(command)
  if (( $? != 0 )); then return 1; fi
  ```
- Location: After constants, before main code
- Order: shebang → set options → constants → functions → main execution

## main() Function
- Use for scripts with multiple functions
- Pattern: `if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then main "$@"; fi`
- Benefits: clear entry point, better scoping, testability

## Argument Parsing
```bash
while [[ $# -gt 0 ]]; do
	case $1 in
		--debug) _DEBUG="$2"; shift 2 ;;
		--dry-run) _DRY_RUN="$2"; shift 2 ;;
		--file) _FILE="$2"; shift 2 ;;
		*) shift ;;
	esac
done
echo "  --debug=${_DEBUG:=false}"
echo "  --dry-run=${_DRY_RUN:=true}"
```

## Debug & Dry-Run
- `--debug`: Enable with `if [[ "${_DEBUG}" == "true" ]]; then set -x; fi`
- `--dry-run`: Use prefix pattern:
  ```bash
  dryrun=""
  [[ "${_DRY_RUN}" == "true" ]] && dryrun="echo [DRY-RUN]"
  ${dryrun} dangerous_command
  ```

## Sourcing
- Prefer `.`; use `source` only when portability is irrelevant (# shellcheck disable=SC1091)
- Pattern: `. "$(dirname "${BASH_SOURCE[0]}")/_functions"`
- Add unique load guard per module to avoid double sourcing:
  ```bash
  if [[ -n "${_MODULE_NAME_LOADED:-}" ]]; then
  	return 0
  fi
  _MODULE_NAME_LOADED=1
  readonly _MODULE_NAME_DIR="${_MODULE_NAME_DIR:-$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)}"
  ```
- **Never** store `$(cd -- "$(dirname -- "$script_file")" && pwd -P)` (or variations) in generically named variables/functions like `SCRIPT_DIR` or `get_script_dir()`
  Inline it every time you source a file (prevents name collisions):
  ```bash
  . "$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/fileA.sh"
  . "$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/fileB.sh"
  ```
- `_MODULE_NAME` must be globally unique (e.g., `_ACME_BACKUP_LOADED`); use the same prefix for `_MODULE_NAME_DIR` if you need a cached directory after the guard returns.

## Variable Declaration
- Never use `declare -i` for integers
- Never use `let` for variables
- Use `declare` only for associative arrays: `declare -A map`
- Use `(( ))` for arithmetic operations

## Case Statements
```bash
case "${var}" in
	pattern1) command ;;
	pattern2)
		multi_line_command
		another_command
		;;
	*) echo "Unknown" >&2; exit 1 ;;
esac
```

## Pipelines
- Short: Keep on one line
- Long: Break with `|` at end of line, indent continuations with one tab
```bash
command1 \
	| command2 \
	| command3
```
- Prefer process substitution to pipe-while: `while read -r line; do ... done < <(cmd)`
- Check PIPESTATUS: `return_codes=( "${PIPESTATUS[@]}" )`

## Error Handling
- **Philosophy**: Handle errors IN functions, not at caller level
- Always check return values of important commands
- Use `|| return 1` pattern for error propagation
- Send errors to stderr: `echo "ERROR: message" >&2`
- Provide context in error messages
```bash
function safe_op() {
	if ! risky_command; then
		echo "ERROR: Operation failed" >&2
		return 1
	fi
	return 0
}
```

## External Commands
- Prefer bash builtins over external commands
- Avoid `cat` unnecessarily: `grep foo file` not `cat file | grep foo`
- Never parse `ls` output - use globs
- Avoid GNU-specific options for portability
- Never use `seq` - use brace expansion or C-style for

## Wildcard Expansion
- Use `./*` not `*` to avoid issues with `-` prefixed filenames
- Use `shopt -s nullglob` for empty directory handling

## eval
- Allowed but requires prominent WARNING comment
- Must validate/sanitize all inputs
- Document why eval is necessary and alternatives considered
```bash
# WARNING: Using eval for tilde expansion
# Input validated above - only alphanumeric usernames
eval echo "~${username}"
```

## Aliases
- Generally prefer functions over aliases
- Aliases evaluated at definition time, functions at invocation time

## Script Reliability

### Idempotency
- Scripts must run multiple times safely with same results
- Use idempotent operations: `kubectl apply` not `kubectl create`
- Check before creating: `[[ -d "${dir}" ]] || mkdir -p "${dir}"`
- Check if config already present before adding

### State Checking
- Validate state before changes
- Check file existence, readability, disk space before operations
- Check service state before restart
```bash
if [[ ! -f "${file}" ]]; then
	echo "ERROR: File not found" >&2
	return 1
fi
```

### Temporary Files
- Always use `mktemp` never predictable names
- Always use `trap` for cleanup
```bash
temp_file=$(mktemp) || return 1
trap 'rm -f "${temp_file}"' EXIT
# use temp_file
```

## Common Patterns
```bash
# Check command exists
command -v git &>/dev/null || { echo "git required" >&2; exit 1; }

# Read file line by line
while IFS= read -r line; do echo "${line}"; done < file.txt

# Safe file iteration
for file in ./*.txt; do [[ -e "${file}" ]] || continue; process "${file}"; done

# Retry with backoff
function retry() {
	local max=3 attempt=1 delay=1
	while (( attempt <= max )); do
		"$@" && return 0
		sleep "${delay}"
		(( delay *= 2, attempt++ ))
	done
	return 1
}

# Example Service Layer test
function test::service() {
	# Section header is required
	print_header "Testing XYZ of the Service Layer"
	
	# Check cache containers
	for cache in "redis" "memcached"; do
		print_test "Checking $cache container"
		COUNT=$(echo "$CONTAINERS" | grep -c "$cache" || echo "0")
		if [ "$COUNT" = "1" ]; then
			print_pass "$cache container found"
		else
			print_fail "$cache container not found"
		fi
	done
}
```

## Quick Decisions Summary
1. Bash 4.4+, tabs (4-wide), 120 char lines
2. Semicolons only when required
3. Self-documenting code (HARD requirement)
4. `function name() { }` format
5. Prefer double quotes for all strings
6. `set -e` is project preference
7. eval allowed with warnings
8. Aliases allowed but prefer functions

## Script Template
- Namespacing keeps modules isolated—prefer `::` for function namespaces when POSIX compliance is not required, fall back to `_` separators when it is. Variables should always use `_` (with `_` prefixed “private” names).
- Every template section below is optional
  - However, if you include the section, then you MUST keep the matching header comment for readability.
  - The final `MAIN` section is required because every script needs a guarded `main`.
- It is common to create BASH script to test other services
  - The tests should be split into functions and placed within the `SERVICE LAYER` section
  - They should also be namespaced with `test::` to clearly indicate their purpose and put below any other `service::` functions

```bash
#!/usr/bin/env bash
set -euo pipefail

# ==============================================================================
# METADATA & CONSTANTS
# ==============================================================================
readonly _MODULE_NAME_VERSION="1.0.0"
readonly _MODULE_NAME_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)"

# ==============================================================================
# CONFIGURATION LAYER (Declarative, no logic)
# ==============================================================================
declare -gA CONFIG=()

# ==============================================================================
# UTILITY LAYER
# ==============================================================================
# Namespace example: util::

# ==============================================================================
# DATA LAYER
# ==============================================================================
# Namespace example: data::

# ==============================================================================
# BUSINESS LOGIC LAYER
# ==============================================================================
# Namespace example: logic::

# ==============================================================================
# SERVICE LAYER
# ==============================================================================
# Namespace example: service::

# ==============================================================================
# COMMAND LAYER
# ==============================================================================
# Namespace example: cmd::

# ==============================================================================
# INITIALIZATION & MAIN
# ==============================================================================
module_name::main() {
	init
	# Main dispatch logic
}

[[ "${BASH_SOURCE[0]}" == "${0}" ]] && module_name::main "$@"
```

## TUI/UI
- Use latest `pfaciana/tui-toolkit` (BPKG) for all non-trivial scripts; do not implement custom TUI elements already covered by the toolkit
- Example `bpkg.json` dependency:
```json
{
  "dependencies": {
    "pfaciana/tui-toolkit": "master"
  }
}
```

## Testing
- Dev testing: latest `pfaciana/tui-testkit` for snapshot tests; require `bashunit` for unit tests
- Exception: tiny throwaway scripts may omit these
```json
{
  "dependencies-dev": {
    "pfaciana/tui-testkit": "v0.1.2"
  },
  "commands": {
    "init": "bpkg run install-bashunit & bpkg run build-entry-file & bpkg run build-test-runner & wait",
    "install-bashunit": "curl -s https://bashunit.typeddevs.com/install.sh | bash -s ./deps/bin",
    "build-entry-file": "[ -e ./index.sh ] || { printf '%b' '#!/usr/bin/env bash\n\n. \u0022$(cd \u0022$(dirname \u0022${BASH_SOURCE[0]}\u0022)\u0022 && pwd)/src/index.sh\u0022\n' > ./index.sh && chmod +x ./index.sh; }",
    "build-test-runner": "[ -e ./test.sh ] || { printf '%b' '#!/usr/bin/env bash\n\n(\nsleep() { :; };\nexport -f sleep;\n\u0022${BPKG_DEPS:-$(cd \u0022$(dirname \u0022${BASH_SOURCE[0]}\u0022)/deps\u0022 && pwd)}/tui-testkit/src/snapshot-testing.sh\u0022 \u0022$(cd \u0022$(dirname \u0022${BASH_SOURCE[0]}\u0022)\u0022 && pwd)\u0022 \u0022$@\u0022\nunset -f sleep\n)\n\nexit $?\n' > ./test.sh && chmod +x ./test.sh; }",
    "tests": "(sleep() { :; } && export -f sleep && ./deps/tui-testkit/src/snapshot-testing.sh . $* && unset -f sleep); exit $?",
    "bashunit": "./deps/bin/bashunit --detailed ./tests/*_test.sh"
  }
}
```

## Key Differences from Common Styles
- **Tabs not spaces** (4-char wide)
- **Double quotes preferred** for all strings (not just expansions)
- **Both `function` keyword AND `()`** (not one or the other)
- **Comments avoided** (self-documenting code required)
- **Script arguments use `_UPPERCASE`** convention
- **Error handling in functions** not callers
- **Idempotency required** for all scripts
