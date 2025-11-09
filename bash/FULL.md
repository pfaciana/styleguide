# Bash Style Guide

## Introduction

This style guide provides comprehensive guidelines for writing safe, predictable, and maintainable Bash scripts. It synthesizes best practices from multiple authoritative sources to help reduce bugs and improve code quality.

### Philosophy

This guide attempts to be as objective as possible, providing reasoning for design decisions.

**Core Principles:**

- **Self-documenting code is mandatory**: Code MUST be written so clearly that comments are rarely needed. Use descriptive names and clear structure
- **Idempotency** scripts should be executed multiple times without changing the outcome beyond the initial execution
- **Consistency over perfection**: When in doubt, prioritize consistency with existing code. Consistency is a good tie-breaker when there's no clear technical argument. However, don't use consistency to justify outdated styles when there are clear advantages to newer approaches
- **Readability matters**: Code is read more often than written. Optimize for the reader, not the writer
- **Explicit over implicit**: Make style choices explicit rather than relying on intuition
- **Namespacing**: Namespacing is required for files that could be sourced, otherwise they are allowed, but not needed

---

## Background

### Which Shell to Use

**✔️ DO**: Use Bash for all scripts

**✔️ DO**: Target Bash 4.4+ - all features up to and including v4.4 are available

**⚠️️ CONSIDER**: If using other shells (sh, zsh, etc.), document the reason clearly

Bash 4.4+ provides powerful features while being widely available on modern systems. Restricting to Bash 4.4+ ensures consistency and allows use of modern shell features including:

- Associative arrays (`declare -A`)
- `readarray`/`mapfile` builtin for reading files into arrays
- `&>>` redirection operator for appending both stdout and stderr
- Improved parameter expansion features
- Better error handling with `BASH_XTRACEFD`
- `${var@Q}` operator for shell-quoted expansion

---

## Shell Files and Interpreter Invocation

### Shebang

**✔️ DO**: Use `#!/usr/bin/env bash` for portability

```bash
#!/usr/bin/env bash
```

**⚠️️ CONSIDER**: Use `#!/bin/bash` when targeting specific environments (e.g., Linux servers with restricted PATHs)

**❌ AVOID**: Using `#!/bin/sh` unless you specifically need POSIX compliance

### File Extensions

**✔️ DO**: Use `.sh` extension for executable scripts

**✔️ DO**: Omit extensions for internal/library scripts (optional)

```bash
# Executable script
my_script.sh

# Library/internal script (optional style)
_functions
```

### Shell Options

**✔️ DO**: Always enable `set -uo pipefail` at the beginning of scripts

**⚠️️ CONSIDER**: Adding `-e` based on project policy (see [Using set -e](#using-set-e))

```bash
#!/usr/bin/env bash
set -uo pipefail
# Enable -e if your project requires fail-fast behavior
```

**Explanation**:

- `set -e`: Exit immediately if a command exits with non-zero status (optional, project preference)
- `set -u`: Treat unset variables as errors
- `set -o pipefail`: Return value of pipeline is the value of the last command to exit with non-zero status

### Sourcing Common Functions

**✔️ DO**: Use `.` (dot) to source common function files

**✔️ DO**: Prefer `.` over `source` for POSIX compliance

**✔️ DO**: Disable shellcheck SC1091 when sourcing files with dynamic paths

```bash
# Good - using dot with shellcheck disable
# shellcheck disable=SC1091
. "$(dirname "${BASH_SOURCE[0]}")/_functions"

# Good - sourcing specific file (inline directory lookup each time)
# shellcheck disable=SC1091
. "$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/_module_helpers.sh"

# Acceptable - source is more readable but less portable
# shellcheck disable=SC1091
source "$(dirname "${BASH_SOURCE[0]}")/_functions"
```

#### Module Load Guards

**✔️ DO**: Add a uniquely named guard variable to every sourced module to prevent duplicate execution

```bash
# At the very top of a module (pick a unique name for _MODULE_NAME)
if [[ -n "${_MODULE_NAME_LOADED:-}" ]]; then
	return 0
fi
_MODULE_NAME_LOADED=1

# DO NOT use generic script dir variable like SCRIPT_DIR - use the unique module prefix
readonly _MODULE_NAME_DIR="${_MODULE_NAME_DIR:-$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)}"
```

`_MODULE_NAME` must be globally unique (e.g., `_ACME_BACKUP_LOADED`). The paired `_MODULE_NAME_DIR` captures that module’s directory exactly once and remains available even when the guard exits early on subsequent loads.

**⚠️ WARNING**: Never assign `$(cd -- "$(dirname -- "$script_file")" && pwd -P)` (or similar) to shared names like `SCRIPT_DIR`, nor wrap it inside generic getters such as `get_script_dir()`. Those shared names collide when multiple modules are sourced and can overwrite each other. Use a unique namespace prefix and inline the lookup per include, e.g.:

```bash
# shellcheck disable=SC1091
. "$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)/fileA.sh"
. "$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)/fileB.sh"
```

If you need to cache the directory, use the same unique namespace prefix: `readonly _MODULE_NAME_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)"`.

---

## Formatting

### Indentation

**✔️ DO**: Use tabs for indentation (displayed as 4 characters wide)

**✔️ DO**: Be consistent within a file and project

**✔️ DO**: Use blank lines between blocks for readability

**❌ AVOID**: Using spaces for indentation

**❌ AVOID**: Mixing tabs and spaces

```bash
# Good - tabs for indentation
function process_file() {
	local file="$1"

	if [[ -f "${file}" ]]; then
		echo "Processing ${file}"
		cat "${file}"
	fi
}

# Wrong - spaces for indentation
function process_file() {
    local file="$1"
    echo "Processing ${file}"
}
```

**Note**: Tabs are also used for `<<-` here-documents to allow indentation stripping:

```bash
function show_help() {
	cat <<-EOF
		Usage: ${0##*/} [OPTIONS] FILE

		Options:
		  -h, --help     Show this help
		  -v, --verbose  Verbose output
	EOF
}
```

### Line Length

**✔️ DO**: Maximum 120 characters per line

**✔️ DO**: Break long lines for readability when approaching the limit

**✔️ DO**: Use here-documents for long and multi-line strings

**❌ AVOID**: Consecutive echo/print statements - they are hard to read

```bash
# Good - using here-document for multi-line output
cat <<-EOF
	This is a long message that spans multiple lines.
	It's much more readable than multiple echo statements.
	You can include variables: ${USER}
	And maintain formatting easily.
EOF

# Wrong - consecutive echo statements
echo "This is a long message that spans multiple lines."
echo "- It's much more readable than multiple echo statements."
echo "- You can include variables: ${USER}"
echo "- And maintain formatting easily."

# Good - single line under 120 chars
echo "Short message that fits on one line"

# Acceptable - long paths or URLs on their own line
readonly LONG_PATH="/very/long/path/to/some/file/that/cannot/be/reasonably/shortened/config.txt"
```

### Semicolons

**✔️ DO**: Only use semicolons when they are required

**❌ AVOID**: Using semicolons when they are optional

```bash
# Wrong - unnecessary semicolons
name='dave';
echo "hello ${name}";

# Right - no semicolons
name='dave'
echo "hello ${name}"

# Required - semicolons needed in control statements
if true; then
	echo "semicolon required before then"
fi

while read -r line; do
	echo "${line}"
done < file.txt

for file in *.txt; do
	process "${file}"
done
```

### Spacing

**✔️ DO**: No more than one blank line in a row

**✔️ DO**: No trailing whitespace

**✔️ DO**: Add newline at end of file

---

## Comments

### General Comment Philosophy

**❌ AVOID**: Comments in general - they should be rare

**✔️ DO**: Write self-documenting code that is so clear comments are unnecessary

**✔️ DO**: Use descriptive variable and function names that explain intent

**✔️ DO**: Break complex logic into well-named functions

**⚠️️ CONSIDER**: Comments only when:

- The code does something non-obvious that cannot be made clearer
- There's a critical warning about dangerous operations (e.g., `eval`)
- Explaining WHY something is done a certain way (not WHAT it does)
- Documenting workarounds for bugs in external tools

```bash
# Wrong - comment states the obvious
# Set the username variable to the current user
username="${USER}"

# Right - self-documenting code
username="${USER}"

# Wrong - comment explains what code does
# Loop through all files and process them
for file in *.txt; do
	process_file "${file}"
done

# Right - code is clear without comment
for file in *.txt; do
	process_file "${file}"
done

# Acceptable - explains WHY, not WHAT
# Using timeout because remote server occasionally hangs indefinitely
timeout 30 ssh remote-server "command"

# Required - warning about dangerous operation
# WARNING: Using eval here - input is sanitized above but review carefully
eval "${dynamic_command}"
```

### File Header

**✔️ DO**: Include minimal file header with purpose only

**❌ AVOID**: Verbose headers, author info, or change logs (use git for history)

```bash
#!/usr/bin/env bash
# Database backup automation for Oracle instances
set -euo pipefail
```

### Function Comments

**❌ AVOID**: Function comments in most cases - use descriptive function names

**✔️ DO**: Make function names and parameters self-explanatory

**⚠️️ CONSIDER**: Brief comment only for complex functions where the name cannot fully convey the behavior

```bash
# Wrong - unnecessary comment
#######################################
# Gets the user's home directory
# Arguments:
#   $1 - username
# Returns:
#   Path to home directory
#######################################
function get_user_home_directory() {
	local username="$1"
	eval echo "~${username}"
}

# Right - self-documenting function name and code
function get_user_home_directory() {
	local username="$1"
	eval echo "~${username}"
}

# Acceptable - complex behavior that needs explanation
# Retries command up to 3 times with exponential backoff (1s, 2s, 4s)
function retry_with_backoff() {
	local max_attempts=3
	local attempt=1
	local delay=1

	while (( attempt <= max_attempts )); do
		if "$@"; then
			return 0
		fi
		sleep "${delay}"
		(( delay *= 2 ))
		(( attempt++ ))
	done

	return 1
}
```

### TODO Comments

**✔️ DO**: Use TODO comments for temporary or imperfect solutions

**✔️ DO**: Format as `TODO: description` or `TODO(name): description`

```bash
# TODO: Handle edge case where backup directory doesn't exist
# TODO(john): Optimize this loop for large file counts (bug #1234)
```

---

## Naming Conventions

### Function Names

**✔️ DO**: Use the `function` keyword

**✔️ DO**: Use `()` after the function name

**✔️ DO**: Use lowercase with underscores to separate words

**✔️ DO**: Use `::` to separate package/namespace from function name

**✔️ DO**: Make function names descriptive and self-explanatory

```bash
# Good - clear, descriptive function names
function backup_database() {
	local database_name="$1"
	pg_dump "${database_name}" > "backup_${database_name}.sql"
}

function validate_email_address() {
	local email="$1"
	[[ "${email}" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
}

function calculate_disk_usage_percentage() {
	local path="$1"
	df -h "${path}" | awk 'NR==2 {print $5}' | tr -d '%'
}

# Good - package/namespace separation
function database::connect() {
	local host="$1"
	local port="$2"
	psql -h "${host}" -p "${port}"
}

function utils::retry_command() {
	local max_attempts="$1"
	shift
	# retry logic here
}

# Wrong - unclear abbreviations
function bkp_db() {
	# What does this do?
}

# Wrong - not descriptive enough
function process() {
	# Process what?
}
```

### Variable Names

**✔️ DO**: Use lowercase with underscores for local variables

**✔️ DO**: Name loop variables descriptively, similar to what they iterate over

```bash
# Good
for zone in "${zones[@]}"; do
	echo "Processing ${zone}"
done

# Less clear
for item in "${zones[@]}"; do
	echo "Processing ${item}"
done
```

### Constants and Environment Variables

**✔️ DO**: Use UPPERCASE with underscores for constants and environment variables

**✔️ DO**: Declare constants at the top of the file

**✔️ DO**: Use `readonly` or `declare -r` for constants

**✔️ DO**: Set constants first, then make them readonly

```bash
# Good - constant
readonly PATH_TO_FILES='/some/path'

# Good - constant and environment variable
declare -xr ORACLE_SID='PROD'

# Good - separate declaration and export
readonly MAX_RETRIES=3
export MAX_RETRIES

# Good - set at runtime, then make readonly
ZIP_VERSION="$(dpkg --status zip | sed -n 's/^Version: //p')"
if [[ -z "${ZIP_VERSION}" ]]; then
	ZIP_VERSION="$(pacman -Q --info zip | sed -n 's/^Version *: //p')"
fi
if [[ -z "${ZIP_VERSION}" ]]; then
	echo "ERROR: Could not determine zip version" >&2
	exit 1
fi
readonly ZIP_VERSION
```

### Variable Declaration Methods

**❌ AVOID**: Using `declare -i` for integer variables

**❌ AVOID**: Using `let` for variable creation or manipulation

**✔️ DO**: Use `declare` only for associative arrays

**✔️ DO**: Use `(( ))` for arithmetic operations

**✔️ DO**: Use simple assignment for regular variables

```bash
# Wrong - unnecessary declare -i
declare -i count=5
let count++

# Wrong - using let
let foo="2 + 2"

# Right - simple assignment and arithmetic
count=5
(( count++ ))

# Right - arithmetic
foo=$((2 + 2))

# Right - declare only for associative arrays
declare -A config_map
config_map[host]="localhost"
config_map[port]="5432"
```

### Script Argument Variables

**✔️ DO**: Use `_UPPERCASE` for variables that receive script arguments

**✔️ DO**: Initialize optional arguments with default values using `${_VAR:=default}`

**✔️ DO**: Display argument values at script and function start for debugging

**✔️ DO**: Use longopts when it makes sense

**✔️ DO**: Try to offer a shortopt for each argument that has a longopt, when possible

```bash
# Parse arguments
while [[ $# -gt 0 ]]; do
	case $1 in
		-d|--database) _DATABASE="$2"; shift 2 ;;
		-h|--host) _HOST="$2"; shift 2 ;;
		-p|--port) _PORT="$2"; shift 2 ;;
		-v|--verbose) _VERBOSE="$2"; shift 2 ;;
		*) shift ;;
	esac
done

# Initialize with defaults and display
echo "Arguments:"
echo "  -d|--database=${_DATABASE}"  # Required - will fail if unset (set -u)
echo "  -h|--host=${_HOST:=localhost}"  # Optional - default to localhost
echo "  -p|--port=${_PORT:=5432}"  # Optional - default to 5432
echo "  -v|--verbose=${_VERBOSE:=false}"  # Optional - default to false
```

---

## Functions

### Function Declaration

**✔️ DO**: Use `function` keyword with `()` parentheses

**✔️ DO**: Place opening brace on same line as function name

**✔️ DO**: Make all function variables local

**✔️ DO**: Use descriptive names that make the function's purpose obvious

```bash
# Good - clear function declaration
function process_log_file() {
	local log_file="$1"
	local error_count

	error_count=$(grep -c "ERROR" "${log_file}")
	echo "${error_count}"
}

# Good - self-documenting with clear variable names
function convert_seconds_to_human_readable() {
	local total_seconds="$1"
	local hours minutes seconds

	hours=$((total_seconds / 3600))
	minutes=$(((total_seconds % 3600) / 60))
	seconds=$((total_seconds % 60))

	printf "%02d:%02d:%02d\n" "${hours}" "${minutes}" "${seconds}"
}

# Wrong - variables leak to global scope
function bad_function() {
	result=$(process "$1")  # Global variable!
	echo "${result}"
}

# Wrong - unclear what this does
function proc() {
	local x="$1"
	echo "${x}"
}
```

### Local Variables

**✔️ DO**: Always use `local` for function variables

**✔️ DO**: Separate declaration and assignment when using command substitution

```bash
# Good - preserves exit code
function my_func() {
  local name="$1"
  local my_var
  my_var="$(some_command)"
  if (( $? != 0 )); then
    return 1
  fi
}

# Wrong - exit code is lost
function bad_func() {
  local my_var="$(some_command)"
  if (( $? != 0 )); then  # This checks exit code of 'local', not 'some_command'
    return 1
  fi
}
```

### Function Location

**✔️ DO**: Place all functions together near the top of a file section

**✔️ DO**: Order: shebang → set options → constants → functions → main code

**❌ AVOID**: Mixing executable code between function declarations

```bash
#!/usr/bin/env bash
set -euo pipefail

# Constants
readonly CONFIG_FILE='/etc/myapp.conf'

# Functions
function helper_func() {
	...
}

function main_func() {
	...
}

# Main execution
main_func "$@"
```

### main() Function

**✔️ DO**: Use a `main()` function for scripts with multiple functions

**✔️ DO**: Call `main "$@"` as the last non-comment line

**✔️ DO**:  Consider namespacing `main()` if it's a file that could be sourced

Using a `main()` function provides:

- Easy identification of program entry point
- Consistency across scripts
- Ability to define more variables as `local`
- Better organization and testability

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly CONFIG_FILE='/etc/myapp.conf'

function load_config() {
	local config_file="$1"
	# Load configuration
}

function process_data() {
	local input_file="$1"
	# Process data
}

function main() {
	local config_file="${1:-${CONFIG_FILE}}"

	load_config "${config_file}"
	process_data "data.txt"

	echo "Processing complete"
}

# Only run main if script is executed directly (not sourced)
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
	main "$@"
fi
```

### Debug and Dry-Run Modes

**✔️ DO**: Implement `--debug` flag for verbose output for larger/destructive scripts

**✔️ DO**: Implement `--dry-run` flag for safe testing for larger/destructive scripts

**⚠️️ CONSIDER**: Setting `--dry-run` default to `true` for destructive scripts

Non-destructive scripts don't need `--dry-run` since they don't modify state.

Debug mode enables `set -x` to trace execution. Dry-run mode shows what would be executed without actually running commands.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Parse arguments
while [[ $# -gt 0 ]]; do
	case $1 in
		--debug) _DEBUG="$2"; shift 2 ;;
		--dry-run) _DRY_RUN="$2"; shift 2 ;;
		--file) _FILE="$2"; shift 2 ;;
		*) shift ;;
	esac
done

# Initialize with defaults
echo "Arguments:"
echo "  --debug=${_DEBUG:=false}"
echo "  --dry-run=${_DRY_RUN:=true}"  # Default true for safety
echo "  --file=${_FILE}"

# Enable debug mode
if [[ "${_DEBUG}" == "true" ]]; then
	set -x
fi

# Setup dry-run prefix
dryrun=""
if [[ "${_DRY_RUN}" == "true" ]]; then
	dryrun="echo [DRY-RUN]"
fi

function main() {
	local file="${_FILE}"

	# Dry-run: shows command instead of executing
	${dryrun} rm -f "${file}"

	# For commands with built-in dry-run support
	if [[ "${_DRY_RUN}" == "true" ]]; then
		kubectl apply -f manifest.yaml --dry-run=server
	else
		kubectl apply -f manifest.yaml
	fi
}

main "$@"
```

---

## Control Flow

### Block Statements

**✔️ DO**: Put `then` on same line as `if`

**✔️ DO**: Put `do` on same line as `while`/`for`

**✔️ DO**: Put `else` and `elif` on their own lines

```bash
# Good
if [[ -f "${file}" ]]; then
	echo "File exists"
elif [[ -d "${file}" ]]; then
	echo "Directory exists"
else
	echo "Does not exist"
fi

# Good
while read -r line; do
	echo "${line}"
done < file.txt

# Wrong
if [[ -f "${file}" ]]
then
	echo "File exists"
fi
```

### Case Statements

**✔️ DO**: Indent alternatives with tabs

**✔️ DO**: Put simple one-liners on same line with space before `;;`

**✔️ DO**: Split complex cases across multiple lines

```bash
case "${expression}" in
	a)
		variable="value"
		some_command "${variable}"
		;;
	b) simple_command ;;
	absolute)
		actions="relative"
		another_command "${actions}"
		;;
	*)
		echo "Unknown option" >&2
		exit 1
		;;
esac
```

### Loops

**✔️ DO**: Declare loop variables as local in functions

```bash
function process_dirs() {
	local dir
	for dir in "${dirs_to_cleanup[@]}"; do
		if [[ -d "${dir}" ]]; then
			rm -rf "${dir}"
		fi
	done
}
```

---

## Bashisms and Best Practices

### Test Expressions

**✔️ DO**: Use `[[ ... ]]` for conditional testing

**❌ AVOID**: Using `[ ... ]` or `test`

```bash
# Good
if [[ -d /etc ]]; then
	echo "Directory exists"
fi

# Wrong
if [ -d /etc ]; then
	echo "Directory exists"
fi

# Wrong
if test -d /etc; then
	echo "Directory exists"
fi
```

**Benefits of `[[ ... ]]`**:

- No word splitting or pathname expansion
- Supports pattern matching and regex
- Safer and more predictable

### Testing Strings

**✔️ DO**: Use `-z` (zero length) and `-n` (non-zero length) explicitly

**✔️ DO**: Use `==` for string equality (more readable than `=`)

**❌ AVOID**: Using filler characters like `"${var}X"` to test strings

**⚠️️ WARNING**: Be careful with `-gt`, `-lt`, etc. - they do numeric comparison, not lexicographical

```bash
# Good - explicit string length tests
if [[ -z "${my_var}" ]]; then
	echo "Variable is empty"
fi

if [[ -n "${my_var}" ]]; then
	echo "Variable is not empty"
fi

# Good - string equality
if [[ "${name}" == "john" ]]; then
	echo "Hello John"
fi

# Wrong - unnecessary filler character
if [[ "${my_var}X" == "X" ]]; then
	echo "Variable is empty"
fi

# Wrong - using = instead of == (less clear)
if [[ "${name}" = "john" ]]; then
	echo "Hello John"
fi

# Dangerous - numeric comparison on strings
if [[ "10" -gt "9" ]]; then
	echo "This works"
fi
if [[ "2" -gt "10" ]]; then
	echo "But this is wrong! Numeric comparison, not lexicographical"
fi
```

### Command Substitution

**✔️ DO**: Use `$(...)` for command substitution

**❌ AVOID**: Using backticks `` `...` ``

```bash
# Good
foo=$(date)
nested=$(command "$(command1)")

# Wrong
foo=`date`
nested=`command \`command1\``
```

### Arithmetic

**✔️ DO**: Use `((...))` and `$((...))` for arithmetic

**✔️ DO**: Variables don't need `$` or `${}` inside `$(( ))`

**✔️ DO**: Use spaces around operators for readability: `(( i += 3 ))`

**❌ AVOID**: Using `let`, `expr`, or `$[...]`

```bash
# Good - arithmetic comparison
if (( a > b )); then
	echo "a is greater"
fi

# Good - arithmetic assignment (no $ needed inside)
(( i = 10 * j + 400 ))
(( i++ ))
(( count += 5 ))

# Good - arithmetic in string
echo "$(( 2 + 2 )) is 4"

# Good - cleaner without $ inside
result=$(( total / count ))

# Acceptable but less clean - using $ inside
result=$(( $total / $count ))

# Wrong - using -gt for arithmetic
if [[ $a -gt $b ]]; then
	echo "a is greater"
fi

# Wrong - using let, expr, or old syntax
let i="2 + 2"
i=$(expr 4 + 4)
i=$[2 * 10]
```

### Parameter Expansion

**✔️ DO**: Prefer parameter expansion over external commands

**✔️ DO**: Use bash built-in string manipulation

```bash
name='bahamas10'

# Good
prog=${0##*/}              # basename
nonumbers=${name//[0-9]/}  # remove all numbers
prefix=${name%10}          # remove suffix

# Wrong
prog=$(basename "$0")
nonumbers=$(echo "$name" | sed -e 's/[0-9]//g')
```

**Common parameter expansions**:

- `${var#pattern}` - Remove shortest match from beginning
- `${var##pattern}` - Remove longest match from beginning
- `${var%pattern}` - Remove shortest match from end
- `${var%%pattern}` - Remove longest match from end
- `${var/pattern/replacement}` - Replace first match
- `${var//pattern/replacement}` - Replace all matches
- `${var:-default}` - Use default if unset
- `${var:=default}` - Assign default if unset

### Arrays

**✔️ DO**: Use bash arrays for lists of elements

**✔️ DO**: Quote array expansions: `"${array[@]}"`

**❌ AVOID**: Using space-separated strings for lists

```bash
# Good
modules=(json httpserver jshint)
for module in "${modules[@]}"; do
	npm install -g "${module}"
done

# Even better if command supports multiple args
npm install -g "${modules[@]}"

# Wrong
modules='json httpserver jshint'
for module in $modules; do
	npm install -g "$module"
done
```

**Array operations**:

```bash
# Declaration
declare -a my_array
my_array=(item1 item2 item3)

# Appending
my_array+=(item4)

# Length
echo "${#my_array[@]}"

# Iteration
for item in "${my_array[@]}"; do
  echo "${item}"
done
```

### Sequences

**✔️ DO**: Use bash builtins for sequences

**❌ AVOID**: Using `seq` command

```bash
# Good - brace expansion for fixed ranges
for i in {1..5}; do
	echo "${i}"
done

# Good - C-style for variable ranges
n=10
for (( i = 0; i < n; i++ )); do
	echo "${i}"
done

# Wrong
for i in $(seq 1 5); do
	echo "${i}"
done
```

### For Loops vs While Read

**✔️ DO**: Use `while read` for processing line-by-line data

**✔️ DO**: Use `for` loops for iterating over arrays or known lists

**❌ AVOID**: Using `for` with command substitution for newline-separated data

**Key differences:**

- `for` with command substitution loads entire output into memory
- `for` splits on all whitespace (spaces, tabs, newlines)
- `while read` streams data line-by-line
- `while read` preserves spaces within lines

```bash
# Wrong - loads entire file into memory, breaks on spaces
users=$(awk -F: '{print $1}' /etc/passwd)
for user in $users; do
	echo "user is $user"
done

# Good - streaming, handles spaces correctly
while IFS=: read -r user _; do
	echo "user is $user"
done < /etc/passwd

# Good - for loop with array (already in memory)
files=(file1.txt file2.txt file3.txt)
for file in "${files[@]}"; do
	process_file "${file}"
done

# Good - for loop with glob
for file in /path/to/*.txt; do
	[[ -e "${file}" ]] || continue
	process_file "${file}"
done

# Good - while read for command output
while IFS= read -r line; do
	echo "Processing: ${line}"
done < <(find . -name "*.txt")
```

### read Builtin

**✔️ DO**: Use `read` with `IFS` for simple parsing

**✔️ DO**: Use `<<<` (here-string) for parsing single strings

**❌ AVOID**: External commands like `cut` or `awk` for simple field splitting

```bash
# Good - parsing with IFS
fqdn="server.example.com"
IFS=. read -r hostname domain tld <<< "${fqdn}"
echo "Host: ${hostname}, Domain: ${domain}, TLD: ${tld}"

# Good - parsing CSV-like data
while IFS=, read -r name email age; do
	echo "Name: ${name}, Email: ${email}, Age: ${age}"
done < users.csv

# Good - reading with default IFS (whitespace)
echo "one two three" | {
	read -r first second third
	echo "First: ${first}"
}

# Wrong - using external command unnecessarily
hostname=$(echo "${fqdn}" | cut -d. -f1)
domain=$(echo "${fqdn}" | cut -d. -f2)
```

---

## Quoting

### General Quoting Rules

**✔️ DO**: Prefer double quotes for all strings (even literal strings)

**✔️ DO**: Use double quotes for strings with variable expansion

**✔️ DO**: Quote all variable expansions unless you have a specific reason not to

**Rationale**: Using double quotes consistently makes it easier to add or remove variable expansions later without having to change quote types. This reduces friction and potential errors when modifying code.

```bash
# Good - double quotes for all strings
foo="Hello World"
bar="You are ${USER}"
message="Processing complete"

# Acceptable - single quotes for truly literal strings (rare cases)
regex='^\d+$'
literal_dollar='This costs $5'

# Good - consistent double quotes make changes easier
# If you later need to add a variable, no quote change needed
status="Success"
# Later: status="Success: ${count} items processed"

# Wrong - single quotes when double quotes would work
foo='hello world'

# Wrong - missing expansion
bar='You are $USER'
```

### Variable Expansion

**✔️ DO**: Quote variables that undergo word-splitting

**✔️ DO**: Use `"${var}"` over `"$var"` (preferred but not mandatory)

**✔️ DO**: Use braces when necessary to delimit variable names

```bash
# Good - double quotes for strings, quotes needed for word-splitting
foo="hello world"
echo "${foo}"

# Good - no quotes needed in [[ ]]
if [[ -n $foo ]]; then
	echo "foo is set"
fi

# Good - no quotes needed for assignment
bar=$foo

# Braces needed for disambiguation
echo "${USER}s home directory"  # "daves home directory"
echo "$USERs home directory"    # " home directory" - wrong!

# Special variables don't need quotes (but can have them)
echo $?
echo $#
echo $$
```

**✔️ DO**: Always quote `"$@"` for argument passing

**❌ AVOID**: Using `$*` unless you specifically want to join arguments

```bash
# Good - preserves arguments as-is
my_function "$@"

# Wrong - word-splits arguments
my_function $@

# Rare case where $* is appropriate
echo "Arguments: $*"
```

### Using {} vs Quotes - Detailed

**Understanding word-splitting behavior:**

When you use `${var}` without quotes, bash performs word-splitting on the value. When you use `"${var}"` with quotes, the value is preserved exactly as-is.

```bash
# Demonstrating word-splitting with spacing
for f in '1 space' '2  spaces' '3   spaces'; do
	echo ${f}  # Wrong - loses spacing
	# Outputs: "1 space", "2 spaces", "3 spaces"
done

for f in '1 space' '2  spaces' '3   spaces'; do
	echo "${f}"  # Good - preserves spacing
	# Outputs: "1 space", "2  spaces", "3   spaces"
done

# When braces are needed for disambiguation
user="dave"
echo "${user}s home directory"  # Good - "daves home directory"
echo "$users home directory"    # Wrong - looks for $users variable

# Braces without quotes still cause word-splitting
files="file1.txt file2.txt file3.txt"
for f in ${files}; do
	echo "${f}"  # Splits on spaces - 3 iterations
done

for f in "${files}"; do
	echo "${f}"  # No splitting - 1 iteration with full string
done

# When quotes are NOT needed
if [[ -n ${var} ]]; then  # Inside [[ ]], no word-splitting occurs
	echo "var is set"
fi

new_var=${old_var}  # Assignment doesn't word-split

# Special variables don't need quotes (but can have them)
exit_code=$?
arg_count=$#
process_id=$$
```

**Rule of thumb:** When in doubt, use quotes. They're almost never wrong, and they prevent many subtle bugs.

---

## External Commands

### Builtin vs External Commands

**✔️ DO**: Prefer bash builtins over external commands

**✔️ DO**: Use parameter expansion instead of `sed`/`awk` for simple operations

```bash
# Good - using builtins
addition=$(( X + Y ))
substitution="${string/#foo/bar}"

# Wrong - forking external commands
addition=$(expr "${X}" + "${Y}")
substitution=$(echo "${string}" | sed -e 's/^foo/bar/')
```

### GNU Userland Tools

**⚠️️ CONSIDER**: Avoiding GNU-specific options for portability

**✔️ DO**: Test scripts on target platforms

The world doesn't run exclusively on GNU/Linux. When using external commands like `awk`, `sed`, `grep`, avoid GNU-specific options when possible.

### Useless Use of Cat

**❌ AVOID**: Using `cat` unnecessarily

**✔️ DO**: Use redirection or pass filename directly

```bash
# Wrong
cat file | grep foo

# Good
grep foo < file

# Better
grep foo file
```

### Listing Files

**❌ AVOID**: Parsing `ls` output

**✔️ DO**: Use bash globbing

```bash
# Wrong - unsafe
for f in $(ls); do
  echo "${f}"
done

# Good
for f in *; do
  echo "${f}"
done

# Good - with nullglob for empty directories
shopt -s nullglob
for f in *.txt; do
  echo "${f}"
done
```

---

## Error Handling

### Error Handling Philosophy

**✔️ DO**: Handle errors within functions, not at the caller level

**✔️ DO**: Return non-zero exit codes from functions on error

**✔️ DO**: Use `|| return 1` pattern for error propagation

**❌ AVOID**: Letting errors propagate up the call stack without handling

Functions should handle their own errors and return appropriate exit codes. Callers should check these codes but shouldn't need to know the details of what went wrong.

```bash
# Good - function handles its own errors
function backup_database() {
	local database="$1"
	local backup_file="backup_${database}.sql"

	if ! pg_dump "${database}" > "${backup_file}"; then
		echo "ERROR: Failed to backup database ${database}" >&2
		return 1
	fi

	if ! gzip "${backup_file}"; then
		echo "ERROR: Failed to compress backup ${backup_file}" >&2
		return 1
	fi

	echo "Successfully backed up ${database}"
	return 0
}

# Good - caller checks return code
if ! backup_database "production"; then
	echo "ERROR: Backup failed, aborting deployment" >&2
	exit 1
fi

# Wrong - caller has to handle function's internal errors
function bad_backup() {
	local database="$1"
	pg_dump "${database}" > "backup.sql"
	gzip "backup.sql"
}

# Caller has no way to know if it worked
bad_backup "production"
# Did it work? Who knows!
```

### Checking Return Values

**✔️ DO**: Always check return values of important commands

**✔️ DO**: Provide informative error messages

```bash
# Good
if ! mv "${file_list[@]}" "${dest_dir}/"; then
	echo "ERROR: Unable to move ${file_list[*]} to ${dest_dir}" >&2
	return 1
fi

# Good - using ||
cd /some/path || exit 1
rm file

# Also good - checking $?
mv "${file}" "${dest}"
if (( $? != 0 )); then
	echo "ERROR: Move failed" >&2
	return 1
fi
```

**✔️ DO**: Use `PIPESTATUS` for checking pipeline return codes

```bash
tar -cf - ./* | ( cd "${dir}" && tar -xf - )
if (( PIPESTATUS[0] != 0 || PIPESTATUS[1] != 0 )); then
  echo "Unable to tar files to ${dir}" >&2
  exit 1
fi

# For complex checking, save PIPESTATUS immediately
tar -cf - ./* | ( cd "${DIR}" && tar -xf - )
return_codes=( "${PIPESTATUS[@]}" )
if (( return_codes[0] != 0 )); then
  echo "Tar creation failed" >&2
fi
if (( return_codes[1] != 0 )); then
  echo "Tar extraction failed" >&2
fi
```

### Error Messages

**✔️ DO**: Send error messages to STDERR

**✔️ DO**: Include context in error messages

```bash
# Good
function err() {
	echo "[$(date +'%Y-%m-%dT%H:%M:%S%z')]: $*" >&2
}

if ! do_something; then
	err "Unable to do_something"
	exit 1
fi
```

### Using set -e

**⚠️️ CONSIDER**: Using `set -e` is a personal/project preference

**✔️ DO**: Be consistent within a project

**✔️ DO**: Understand the implications if using `set -e`:

- Script exits immediately on any non-zero exit code
- Doesn't work in all contexts (e.g., inside `if` conditions, with `||`, or in pipelines without `pipefail`)
- Can hide errors in some cases
- Makes it harder to handle expected failures

**If using `set -e`**:

- Understand its limitations (see BashFAQ105)
- Test thoroughly
- Use `set -euo pipefail` together for comprehensive error handling
- Use `|| true` for commands that are allowed to fail

**If not using `set -e`**:

- Check return values explicitly with `||` or `if`
- Be disciplined about error checking for all critical commands

```bash
# With set -e
set -euo pipefail

function backup_database() {
	local database="$1"

	# This will exit script if pg_dump fails
	pg_dump "${database}" > "backup.sql"

	# Allow this to fail without exiting
	gzip "backup.sql" || true
}

# Without set -e
function backup_database() {
	local database="$1"

	if ! pg_dump "${database}" > "backup.sql"; then
		echo "ERROR: Database backup failed" >&2
		return 1
	fi

	# Explicitly handle optional compression
	if ! gzip "backup.sql"; then
		echo "WARNING: Compression failed, backup remains uncompressed" >&2
	fi
}
```

---

## Advanced Topics

### Pipeline Formatting

**✔️ DO**: Put each pipeline component on its own line for long pipelines

**✔️ DO**: Put the pipe `|` at the end of the line (not the beginning)

**✔️ DO**: Indent continuation lines with one tab

**⚠️️ CONSIDER**: Keeping short pipelines on one line

```bash
# Good - short pipeline on one line
cat file.txt | grep "pattern" | wc -l

# Good - long pipeline split across lines
command1 \
	| command2 \
	| command3 \
	| command4

# Good - complex pipeline with proper indentation
find . -name "*.txt" -type f \
	| xargs grep -l "pattern" \
	| while read -r file; do
		echo "Processing ${file}"
		process_file "${file}"
	done

# Good - pipeline with line breaks for readability
docker ps -a \
	| grep "exited" \
	| awk '{print $1}' \
	| xargs docker rm

# Acceptable - pipe at beginning of line (less common style)
command1 \
	| command2 \
	| command3

# Wrong - inconsistent formatting
command1 | command2 \
| command3 \
	| command4
```

### Pipes to While

**✔️ DO**: Use process substitution or `readarray` instead of piping to `while`

**❌ AVOID**: Piping to `while` - variables don't propagate to parent shell

```bash
# Wrong - last_line will always be 'NULL'
last_line='NULL'
your_command | while read -r line; do
	if [[ -n "${line}" ]]; then
		last_line="${line}"
	fi
done
echo "${last_line}"  # Still 'NULL'!

# Good - using process substitution
last_line='NULL'
while read -r line; do
	if [[ -n "${line}" ]]; then
		last_line="${line}"
	fi
done < <(your_command)
echo "${last_line}"  # Correct value

# Also good - using readarray (bash 4.4+)
last_line='NULL'
readarray -t lines < <(your_command)
for line in "${lines[@]}"; do
	if [[ -n "${line}" ]]; then
		last_line="${line}"
	fi
done
echo "${last_line}"  # Correct value
```

### Wildcard Expansion

**✔️ DO**: Use explicit paths with wildcards

**✔️ DO**: Use `./*` instead of `*` to avoid issues with filenames starting with `-`

```bash
# Wrong - dangerous if files start with -
rm -v *

# Good - safe
rm -v ./*
```

### Eval

**⚠️️ CONSIDER**: `eval` is allowed but requires extreme caution

**✔️ DO**: Add prominent warning comments when using `eval`

**✔️ DO**: Document why `eval` is necessary and what alternatives were considered

**✔️ DO**: Sanitize and validate all inputs before using with `eval`

**❌ AVOID**: Using `eval` with unsanitized user input

`eval` is dangerous and should be avoided when possible. It:

- Opens code to injection attacks if not carefully controlled
- Makes static analysis difficult
- Can often be replaced with arrays, indirect expansion, or proper quoting

However, there are legitimate use cases where `eval` is the cleanest solution (e.g., dynamic variable names, complex command construction).

```bash
# Acceptable - with clear warning and justification
function get_user_home_directory() {
	local username="$1"

	# Validate input to prevent injection
	if [[ ! "${username}" =~ ^[a-zA-Z0-9_-]+$ ]]; then
		echo "ERROR: Invalid username format" >&2
		return 1
	fi

	# WARNING: Using eval to expand tilde for arbitrary username
	# Alternative approaches (getent, awk on /etc/passwd) don't handle all cases
	# Input is validated above to prevent injection
	eval echo "~${username}"
}

# Acceptable - dynamic variable access with validation
function get_config_value() {
	local key="$1"

	# Validate key is alphanumeric only
	if [[ ! "${key}" =~ ^[A-Z_][A-Z0-9_]*$ ]]; then
		echo "ERROR: Invalid config key" >&2
		return 1
	fi

	# WARNING: Using eval for dynamic variable access
	# Input validated above - only allows safe variable names
	eval echo "\${CONFIG_${key}}"
}

# Wrong - dangerous, no validation
function dangerous_eval() {
	local user_input="$1"
	# NEVER DO THIS - allows arbitrary code execution
	eval "${user_input}"
}

# Better alternative - use arrays instead of eval when possible
function set_multiple_variables() {
	local -A config=(
		[database]="mydb"
		[host]="localhost"
		[port]="5432"
	)

	# No eval needed
	echo "${config[database]}"
}
```

### Aliases

**⚠️️ CONSIDER**: Aliases are generally not recommended in scripts, but allowed if needed

**✔️ DO**: Prefer functions over aliases in most cases

**✔️ DO**: If using aliases, document why they're necessary

Aliases have limitations:

- Require careful quoting and escaping
- Evaluated at definition time, not invocation time
- Can be confusing to debug
- Functions provide more flexibility

```bash
# Generally avoid - alias evaluated at definition time
alias random_name="echo some_prefix_${RANDOM}"
# This will always echo the same number!

# Preferred - function evaluated at invocation time
function random_name() {
	echo "some_prefix_${RANDOM}"
}

# Acceptable use case - shortening common command patterns
alias ll='ls -lah'
alias grep='grep --color=auto'

# But functions are still better for scripts
function list_detailed() {
	ls -lah "$@"
}
```

---

## Script Reliability and Stability

### Writing Rerunnable Scripts (Idempotency)

**✔️ DO**: Write scripts that can be run multiple times safely

**✔️ DO**: Ensure same arguments yield same results

**✔️ DO**: Allow scripts to continue from failure points

**✔️ DO**: Use idempotent operations

Idempotent scripts can be run multiple times without causing errors or unintended side effects. This is crucial for automation, CI/CD, and recovery from failures.

```bash
# Wrong - fails on second run
function deploy_service() {
	kubectl create deployment myapp --image=myapp:latest
	kubectl create service clusterip myapp --tcp=80:8080
}

# Good - idempotent, can run multiple times
function deploy_service() {
	kubectl apply -f deployment.yaml
	kubectl apply -f service.yaml
}

# Good - check before creating
function create_directory() {
	local dir="$1"

	if [[ -d "${dir}" ]]; then
		echo "Directory ${dir} already exists, skipping"
		return 0
	fi

	if ! mkdir -p "${dir}"; then
		echo "ERROR: Failed to create directory ${dir}" >&2
		return 1
	fi

	echo "Created directory ${dir}"
}

# Good - idempotent file operations
function ensure_config_line() {
	local config_file="$1"
	local config_line="$2"

	if grep -qF "${config_line}" "${config_file}"; then
		echo "Configuration already present"
		return 0
	fi

	echo "${config_line}" >> "${config_file}"
	echo "Added configuration"
}
```

### Check State Before Changing

**✔️ DO**: Validate state before making changes

**✔️ DO**: Avoid unnecessary operations

**✔️ DO**: Prevent errors from invalid states

Always check the current state before attempting to change it. This prevents errors and makes scripts more robust.

```bash
# Good - check file exists before processing
function process_config() {
	local config_file="$1"

	if [[ ! -f "${config_file}" ]]; then
		echo "ERROR: Config file ${config_file} not found" >&2
		return 1
	fi

	if [[ ! -r "${config_file}" ]]; then
		echo "ERROR: Config file ${config_file} not readable" >&2
		return 1
	fi

	# Now safe to process
	process_file "${config_file}"
}

# Good - check service state before restarting
function restart_service() {
	local service="$1"

	if ! systemctl is-active --quiet "${service}"; then
		echo "Service ${service} is not running, starting instead"
		systemctl start "${service}"
		return $?
	fi

	echo "Restarting ${service}"
	systemctl restart "${service}"
}

# Good - check disk space before operations
function backup_database() {
	local database="$1"
	local backup_dir="/backups"
	local required_space_mb=1000

	local available_space
	available_space=$(df -m "${backup_dir}" | awk 'NR==2 {print $4}')

	if (( available_space < required_space_mb )); then
		echo "ERROR: Insufficient disk space. Need ${required_space_mb}MB, have ${available_space}MB" >&2
		return 1
	fi

	pg_dump "${database}" > "${backup_dir}/backup_${database}.sql"
}
```

### Safely Creating Temporary Files

**✔️ DO**: Use `mktemp` for creating temporary files

**✔️ DO**: Use `trap` to ensure cleanup on exit

**✔️ DO**: Clean up temporary files even on error

**❌ AVOID**: Using predictable temporary file names

**❌ AVOID**: Creating temp files in world-writable directories without `mktemp`

```bash
# Good - safe temporary file with cleanup
function process_with_temp() {
	local temp_file
	temp_file=$(mktemp) || {
		echo "ERROR: Failed to create temporary file" >&2
		return 1
	}

	# Ensure cleanup on exit (success or failure)
	trap 'rm -f "${temp_file}"' EXIT

	# Use temporary file
	curl -s "https://api.example.com/data" > "${temp_file}"
	process_data "${temp_file}"

	# Cleanup happens automatically via trap
}

# Good - temporary directory with cleanup
function process_with_temp_dir() {
	local temp_dir
	temp_dir=$(mktemp -d) || {
		echo "ERROR: Failed to create temporary directory" >&2
		return 1
	}

	trap 'rm -rf "${temp_dir}"' EXIT

	# Use temporary directory
	extract_archive "${temp_dir}"
	process_files "${temp_dir}"

	# Cleanup happens automatically
}

# Good - multiple temp files with single trap
function complex_processing() {
	local temp_file1 temp_file2 temp_dir

	temp_file1=$(mktemp) || return 1
	temp_file2=$(mktemp) || return 1
	temp_dir=$(mktemp -d) || return 1

	# Single trap for all cleanup
	trap 'rm -f "${temp_file1}" "${temp_file2}"; rm -rf "${temp_dir}"' EXIT

	# Process with temporary files
	download_data > "${temp_file1}"
	transform_data < "${temp_file1}" > "${temp_file2}"
	extract_to_dir "${temp_file2}" "${temp_dir}"
}

# Wrong - predictable filename, no cleanup
function bad_temp_usage() {
	local temp_file="/tmp/myapp_temp.txt"
	curl -s "https://api.example.com/data" > "${temp_file}"
	process_data "${temp_file}"
	# File left behind, security risk
}
```

---

## TUI/UI Requirements

**✔️ DO**: For all non-trivial scripts, depend on the latest `pfaciana/tui-toolkit` BPKG package to provide consistent, minimal TUI/UI.

**❌ AVOID**: Creating custom TUI elements already provided by `pfaciana/tui-toolkit`.

`bpkg.json` dependency reference (pin to master by default):

```json
{
  "dependencies": {
    "pfaciana/tui-toolkit": "master"
  }
}
```

**⚠️ EXCEPTION**: Very small standalone throwaway scripts may omit this dependency.

---

## Testing Requirements

**✔️ DO**: Use the latest `pfaciana/tui-testkit` as a `dependencies-dev` for snapshot testing, and require `bashunit` for unit tests.

Example `bpkg.json` (adapt version pins as newer releases become available):

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

---

## Common Mistakes

### Using {} Without Quotes

**❌ AVOID**: Using `${var}` without quotes when word-splitting matters

```bash
# Wrong - loses spacing
for f in '1 space' '2  spaces' '3   spaces'; do
	echo ${f}  # Outputs: "1 space", "2 spaces", "3 spaces"
done

# Good - preserves spacing
for f in '1 space' '2  spaces' '3   spaces'; do
	echo "${f}"  # Outputs: "1 space", "2  spaces", "3   spaces"
done
```

### Abusing For-Loops

**❌ AVOID**: Using `for` loops for newline-separated data

**✔️ DO**: Use `while read` for processing line-by-line

```bash
# Wrong - reads entire file into memory, breaks on spaces
users=$(awk -F: '{print $1}' /etc/passwd)
for user in $users; do
	echo "user is $user"
done

# Good - streaming, handles spaces correctly
while IFS=: read -r user _; do
	echo "user is $user"
done < /etc/passwd
```

---

## Quick Reference

### Script Template

- Namespacing keeps modules isolated—prefer `::` for function namespaces when POSIX compliance is not required, fall back to `_` separators when it is. Variables should always use `_` (with `_` prefixed “private” names).
- Each template section below is optional
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
# NOTE: variables MUST NOT collide with other scripts, use namespacing w/ `_`
readonly _MODULE_NAME_VERSION="1.0.0"
readonly _MODULE_NAME_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)"

# ==============================================================================
# CONFIGURATION LAYER (Declarative, no logic)
# ==============================================================================
declare -gA CONFIG=()

# ==============================================================================
# UTILITY LAYER (Logging, Errors, Common/Helper Functions)
# ==============================================================================
# Example Namespace: util::

# ==============================================================================
# DATA LAYER (Types, Schemas, Validators)
# ==============================================================================
# Example Namespace: data::

# ==============================================================================
# BUSINESS LOGIC LAYER (Pure Functions)
# ==============================================================================
# Example Namespace: logic::

# ==============================================================================
# SERVICE LAYER (External I/O, State Mutations)
# ==============================================================================
# Example Namespace: service:: or test::
# Use the `test::` namespace for functions that test services

# ==============================================================================
# COMMAND LAYER (User-facing commands)
# ==============================================================================
# Example Namespace: cmd::

# ==============================================================================
# MAIN
# ==============================================================================

# A main function is required. Namespacing (e.g., module_name::main) is
# strongly recommended, but small standalone scripts MAY skip it 
# if there is never to be sourced
function module_name::main() {
	init
	# Main dispatch logic
}

# Guard main so sourcing the file will not execute it.
[[ "${BASH_SOURCE[0]}" == "${0}" ]] && module_name::main "$@"
```

### Common Patterns

```bash
# Check if command exists
if command -v git &>/dev/null; then
	echo "git is installed"
fi

# Read file line by line
while IFS= read -r line; do
	echo "${line}"
done < file.txt

# Process command output
while IFS= read -r line; do
	echo "${line}"
done < <(some_command)

# Iterate over files safely
for file in ./*.txt; do
	[[ -e "${file}" ]] || continue
	echo "${file}"
done

# Parse arguments
while [[ $# -gt 0 ]]; do
	case $1 in
		-h|--help)
			show_usage
			exit 0
			;;
		-v|--verbose)
			VERBOSE=1
			shift
			;;
		-*)
			echo "ERROR: Unknown option: $1" >&2
			exit 1
			;;
		*)
			FILES+=("$1")
			shift
			;;
	esac
done

# Multi-line output with heredoc
cat <<-EOF
	This is a multi-line message.
	It's much cleaner than multiple echo statements.
	Variables work: ${USER}
EOF

# Retry logic with exponential backoff
function retry_with_backoff() {
	local max_attempts=3
	local attempt=1
	local delay=1

	while (( attempt <= max_attempts )); do
		if "$@"; then
			return 0
		fi
		sleep "${delay}"
		(( delay *= 2 ))
		(( attempt++ ))
	done

	return 1
}

# Temporary file with cleanup
function use_temp_file() {
	local temp_file
	temp_file=$(mktemp) || return 1
	trap 'rm -f "${temp_file}"' EXIT

	# Use temp_file
	echo "data" > "${temp_file}"
}

# Idempotent directory creation
function ensure_directory() {
	local dir="$1"
	[[ -d "${dir}" ]] || mkdir -p "${dir}"
}

# Parse with IFS
IFS=. read -r hostname domain tld <<< "server.example.com"

# Error handling in functions
function safe_operation() {
	if ! risky_command; then
		echo "ERROR: Operation failed" >&2
		return 1
	fi
	return 0
}

# Debug and dry-run support
if [[ "${_DEBUG}" == "true" ]]; then
	set -x
fi

dryrun=""
if [[ "${_DRY_RUN}" == "true" ]]; then
	dryrun="echo [DRY-RUN]"
fi
${dryrun} dangerous_command

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

### References

#### Other Style Guides

- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)
  - [GitHub](https://github.com/google/styleguide)
- [guitarrapc/bash-styleguide](https://github.com/guitarrapc/bash-styleguide)
- [Dave Eddy](https://style.ysap.sh/) 
  - [GitHub](https://github.com/bahamas10/bash-style-guide)

#### Recommended BPKG packages

- [TUI Toolkit](https://github.com/pfaciana/tui-toolkit)
- [TUI Testkit](https://github.com/pfaciana/tui-testkit)
