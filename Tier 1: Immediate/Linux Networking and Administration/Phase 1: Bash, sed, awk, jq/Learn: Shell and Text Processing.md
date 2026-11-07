# Claude-generated learning phase

## Phase 1: Shell and text processing

Learn

- Bash: quoting and word splitting, parameter expansion (`${var:-default}, ${var%suffix}`), indexed and associative arrays, `[[ ]]` compared with `[ ]`, exit status, functions, `trap`, `getopts`, here-documents, process substitution, `mktemp`.
- `set -euo pipefail`: learn each option and its exceptions. For example, `set -e` is ignored inside conditions.
- `sed`: the s command and its flags, line addresses, `-n` with `p`, `-E`, and `-i` with a backup suffix.
- `awk`: pattern-action pairs, `$1` to `$NF`, `-F`, `BEGIN` and `END`, associative arrays, `printf`. On Debian and Ubuntu the default `awk` is `mawk`. Use `gawk` explicitly for gawk-only features.
- `jq`: path expressions, `select`, `map`, `group_by`, `--arg`, `--argjson`, `-r`, `@csv`.
