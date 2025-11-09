# Bash Style Guide - Quick Reference Card

## Core Principles

1. **Self-documenting code is MANDATORY** - Write code so clear that comments are unnecessary
2. **Descriptive names** - Use clear, full words that explain purpose
3. **Consistency over perfection** - When in doubt, be consistent
4. **Explicit over implicit** - Make style choices clear
5. **Safety first** - Use constructs that prevent errors
6. **Idempotency** - Scripts should run multiple times safely

## File Structure

- Namespacing keeps modules isolated—prefer `::` for function namespaces when POSIX compliance is not required, fall back to `_` separators when it is. Variables should always use `_` (with `_` prefixed “private” names).
- Every template section below is optional
  - However, if you include the section, then you MUST keep the matching header comment for readability.
  - The final `MAIN` section is required because every script needs a guarded `main`.
- It is common to create BASH script to test other services
  - The tests should be split into functions and placed within the `SERVICE LAYER` section
  - They should also be namespaced with `test::` to clearly indicate their purpose and put below any other `service::` functions

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

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
# UTILITY LAYER (Logging, Errors, Helper Functions)
# ==============================================================================
# Namespace example: util::

# ==============================================================================
# DATA LAYER (Types, Schemas, Validators)
# ==============================================================================
# Namespace example: data::

# ==============================================================================
# BUSINESS LOGIC LAYER (Pure Functions)
# ==============================================================================
# Namespace example: logic::

# ==============================================================================
# SERVICE LAYER (External I/O, State Mutations)
# ==============================================================================
# Namespace example: service::

# ==============================================================================
# COMMAND LAYER (User-facing commands)
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

### bpkg.json Example (Tooling & Tests)

```json
{
	"dependenciesv": {
		"pfaciana/tui-toolkit": "master"
	},
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

Small, throwaway scripts may omit these dependencies.

## Quick Rules

| Category            | Rule                       | Example                                    |
|---------------------|----------------------------|--------------------------------------------|
| **Bash Version**    | 4.4+                       | All features up to v4.4 available          |
| **Shebang**         | `#!/usr/bin/env bash`      | Works across PATH                          |
| **Indentation**     | Tabs (4-char wide)         | Use tabs, never spaces                     |
| **Line Length**     | 120 chars max              | Break long lines                           |
| **Semicolons**      | Only when required         | `if true; then` ✓ / `name="x";` ✗          |
| **Comments**        | Avoid - self-document      | Use descriptive names instead              |
| **Functions**       | `function pkg::name() { }` | Keyword + () + optional namespace          |
| **Variables**       | lowercase_with_underscores | Avoid `declare -i`/`let`                   |
| **Script Args**     | `_UPPERCASE`               | `_DATABASE`, `_HOST`                       |
| **Constants**       | UPPERCASE_WITH_UNDERSCORES | `readonly MAX_RETRIES=3`                   |
| **Test**            | `[[ ... ]]`                | Never use `[ ... ]` or `test`              |
| **String Tests**    | `-z`, `-n`, `==`           | `[[ -z "${var}" ]]`, prefer `==` over `=`  |
| **Arithmetic**      | `(( ... ))`                | Never use `let` or `expr`                  |
| **Arithmetic Vars** | No `$` inside `$(())`      | `(( i = j + 1 ))` not `(( i = $j + 1 ))`   |
| **Substitution**    | `$(command)`               | Never use backticks                        |
| **Arrays**          | `"${array[@]}"`            | Always quote                               |
| **Quoting**         | Prefer double quotes       | `"${var}"` and `"literal"`                 |
| **For vs While**    | For=arrays, While=lines    | `for` loads memory, `while read` streams   |
| **Sourcing**        | Prefer `.`                 | `. "${SCRIPT_DIR}/_functions"`             |
| **Temp Files**      | `mktemp` + `trap`          | Always cleanup with trap                   |
| **Idempotency**     | Check before change        | `[[ -d "${dir}" ]] \|\| mkdir -p "${dir}"` |
| **TUI Toolkit**     | Use latest `pfaciana/tui-toolkit` (BPKG) | Do not reimplement toolkit UI elements     |
| **Testing**         | Dev: latest `pfaciana/tui-testkit`; Unit: bashunit | Snapshot + unit testing required           |

## Do's and Don'ts

### ✅ DO

```bash
# Self-documenting function names
function calculate_disk_usage_percentage() { ... }
function validate_email_address() { ... }

# Heredoc for multi-line output
cat <<-EOF
	Multi-line message
	With proper formatting
EOF

# Prefer double quotes for all strings
message="Processing complete"
readonly database_backup_directory="/var/backups/db"
local user_home_directory="/home/${username}"

# Use [[ ]] for tests
if [[ -f "${file}" ]]; then
	process_file "${file}"
fi

# Use (( )) for arithmetic
if (( count > max_retries )); then
	exit 1
fi

# Quote array expansions
for item in "${items[@]}"; do
	echo "${item}"
done

# Variables don't need $ inside (( ))
(( i = j + 1 ))
(( count += 5 ))

# Use while read for line-by-line processing
while IFS= read -r line; do
	echo "${line}"
done < file.txt

# Parse with IFS
IFS=. read -r hostname domain tld <<< "server.example.com"

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

### ❌ DON'T

```bash
# Unclear abbreviations
function proc_db() { ... }  # What does this do?

# Consecutive echo statements
echo "Line 1"
echo "Line 2"
echo "Line 3"

# Unnecessary semicolons
name="value";
echo "${name}";

# Single quotes when double quotes would work
name='value'  # Use "value" instead

# Old-style tests
if [ -f "${file}" ]; then  # Use [[ ]] instead

# Old-style arithmetic
let count++  # Use (( count++ )) instead

# Backticks
result=`command`  # Use $(command) instead

# Unquoted variables (when word-splitting matters)
echo $variable  # Use "${variable}"

# Using $ inside (( ))
(( i = $j + 1 ))  # Don't use $, write: (( i = j + 1 ))

# For loop with command substitution
for user in $(cat users.txt); do  # Loads all into memory, breaks on spaces
	echo "${user}"
done

# Pipe to while (variables don't propagate)
cat file.txt | while read -r line; do  # Use: while read -r line; do ... done < file.txt
	last_line="${line}"
done
```

## Common Patterns

```bash
# Module load guard (place at top of every sourced file; _MODULE_NAME must be unique)
if [[ -n "${_MODULE_NAME_LOADED:-}" ]]; then
	return 0
fi
_MODULE_NAME_LOADED=1
readonly _MODULE_NAME_DIR="${_MODULE_NAME_DIR:-$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)}"

# If not using module guard namespacing, then inline script dir lookup when sourcing; never store it in variables/helpers
# shellcheck disable=SC1091
. "$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/fileA.sh"
. "$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/fileB.sh"

# Error handling in functions
function safe_operation() {
	if ! risky_command; then
		echo "ERROR: Operation failed" >&2
		return 1
	fi
	return 0
}

# Process substitution (not pipe to while)
while read -r line; do
	echo "${line}"
done < <(command)

# Safe file iteration
for file in ./*.txt; do
	[[ -e "${file}" ]] || continue
	process_file "${file}"
done

# Argument parsing with defaults
while [[ $# -gt 0 ]]; do
	case $1 in
		-d|--database) _DATABASE="$2"; shift 2 ;;
		-h|--host) _HOST="$2"; shift 2 ;;
		*) shift ;;
	esac
done
echo "  --database=${_DATABASE}"
echo "  --host=${_HOST:=localhost}"

# Check command exists
if command -v git &>/dev/null; then
	git status
fi

# Temporary file with cleanup
temp_file=$(mktemp) || exit 1
trap 'rm -f "${temp_file}"' EXIT

# Idempotent operations
[[ -d "${dir}" ]] || mkdir -p "${dir}"

# Parse with IFS
IFS=. read -r hostname domain tld <<< "server.example.com"

# Debug and dry-run
if [[ "${_DEBUG}" == "true" ]]; then
	set -x
fi

dryrun=""
[[ "${_DRY_RUN}" == "true" ]] && dryrun="echo [DRY-RUN]"
${dryrun} dangerous_command

# Check state before changing
if [[ ! -f "${config_file}" ]]; then
	echo "ERROR: Config not found" >&2
	return 1
fi

# Retry with exponential backoff
function retry_with_backoff() {
	local max_attempts=3 attempt=1 delay=1
	while (( attempt <= max_attempts )); do
		"$@" && return 0
		sleep "${delay}"
		(( delay *= 2, attempt++ ))
	done
	return 1
}

# main() function pattern
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
	main "$@"
fi
```

## When Comments ARE Needed (incl. eval)

```bash
# Explain WHY (not WHAT)
timeout 30 ssh remote-server "command"  # remote can hang

# WARNING for dangerous ops (eval)
# WARNING: Using eval after validating/sanitizing input
eval echo "~${username}"

# TODOs for temporary solutions
# TODO: Optimize for large file counts (bug #1234)
```

## Parameter Expansion Quick Reference

```bash
${var#pattern}    # Remove shortest match from beginning
${var##pattern}   # Remove longest match from beginning
${var%pattern}    # Remove shortest match from end
${var%%pattern}   # Remove longest match from end
${var/pat/repl}   # Replace first match
${var//pat/repl}  # Replace all matches
${var:-default}   # Use default if unset
${var:=default}   # Assign default if unset
```

## Loop Types

```bash
# For loops - use for arrays, globs, fixed ranges
for i in {1..5}; do echo "${i}"; done
for (( i = 0; i < n; i++ )); do echo "${i}"; done
for file in ./*.txt; do [[ -e "${file}" ]] || continue; done

# While read - use for line-by-line, streaming
while IFS= read -r line; do echo "${line}"; done < file.txt
while read -r line; do echo "${line}"; done < <(command)
```

## Case Statement

```bash
case "${var}" in
	pattern1) command ;;
	pattern2)
		multi_line
		commands
		;;
	*) echo "Unknown" >&2; exit 1 ;;
esac
```

## Pipeline Formatting

```bash
# Short - keep on one line
cat file.txt | grep "pattern" | wc -l

# Long - break with | at end of line
command1 \
	| command2 \
	| command3
```

## Key Decisions Summary

1. Bash 4.4+, tabs (4-char wide), 120 char lines
2. Semicolons only when required
3. Self-documenting code (HARD requirement)
4. `function name() { }` format (both keyword AND parentheses)
5. Prefer double quotes for all strings
6. Script arguments use `_UPPERCASE` convention
7. Variables don't need `$` inside `$(( ))`
8. Handle errors IN functions, not at caller level
9. `set -e` is project preference
10. eval allowed with prominent WARNING comments
11. Idempotency required for all scripts

## Remember

- **Idempotency > One-time execution**
- **Consistency > Personal preference**
- **Avoid GNU-only flags unless documented**
- **Safety > Convenience**
