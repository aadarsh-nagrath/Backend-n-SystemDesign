# Scripting & Automation (Bash / Python) — DevOps Interview Questions

---

### 1. What's the difference between `$@` and `$*` in a bash script?
Both expand to all positional arguments, but when quoted, they behave differently: `"$@"` expands to each argument as a *separate* quoted word (preserving arguments containing spaces correctly), while `"$*"` expands to a *single* word with all arguments joined by the first character of `IFS` (usually a space) — `"$@"` is almost always what you want when forwarding arguments to another command, since `"$*"` will break on any argument containing whitespace.

### 2. Explain `set -e`, `set -u`, and `set -o pipefail` in a bash script, and why scripts should generally start with them.
`set -e` — exit immediately if any command returns a non-zero status (without it, a script silently continues after a failed command, potentially causing cascading damage from operating on a failed step's incomplete results). `set -u` — treat referencing an unset variable as an error and exit, instead of silently substituting an empty string (catches typos in variable names immediately). `set -o pipefail` — makes a pipeline (`cmd1 | cmd2`) return the exit status of the *first* failing command in the pipe, instead of only the last command's status (without it, `false | true` "succeeds" because `true`'s exit code is what's checked, masking `false`'s failure). Together (`set -euo pipefail`), these turn bash's normally very permissive, silent-failure-prone default behavior into something closer to a language that fails loudly and immediately, which is much safer for automation scripts.

### 3. What's the difference between `[ ]`, `[[ ]]`, and `(( ))` in bash conditionals?
`[ ]` — the POSIX test command; word-splitting and glob expansion happen on its arguments, so unquoted variables can break/behave unexpectedly (`[ $var = "x" ]` breaks if `$var` is empty or contains spaces — always quote: `[ "$var" = "x" ]`). `[[ ]]` — a bash (and other modern shells') keyword with safer parsing (no word-splitting/glob expansion on unquoted variables inside it), supports pattern matching (`[[ $file == *.txt ]]`) and regex (`[[ $str =~ ^[0-9]+$ ]]`), and logical `&&`/`||` directly inside — generally preferred over `[ ]` in bash-specific scripts. `(( ))` — arithmetic evaluation/comparison context (`(( x > 5 ))`), not string-based like the others.

### 4. How do you safely pass a variable that might contain spaces or special characters to a command?
Always double-quote variable expansions: `cp "$source_file" "$dest_dir"` — unquoted `$source_file` undergoes word-splitting (a filename with a space becomes two arguments) and glob expansion (a filename containing `*` could expand into a list of matching filenames). This is one of the most common sources of bash scripting bugs, especially when scripts are later used with unexpected/user-controlled filenames.

### 5. What's the difference between a subshell `(...)` and a command group `{ ...; }` in bash?
A subshell `(...)` runs its commands in a *forked child* shell process — variable assignments, `cd`, and `exit` inside it don't affect the parent shell's environment once it completes. A command group `{ ...; }` runs in the *current* shell — assignments and `cd` inside it persist afterward, and it's more efficient (no fork). A common practical use of subshells: `(cd /some/dir && do_something)` temporarily changes directory only for that command group without affecting the rest of the script's working directory.

### 6. How would you write a bash script that retries a command up to N times with exponential backoff?
```bash
retry() {
  local max=$1 delay=$2; shift 2
  local attempt=1
  until "$@"; do
    if (( attempt >= max )); then
      echo "Failed after $attempt attempts" >&2
      return 1
    fi
    echo "Attempt $attempt failed, retrying in ${delay}s..." >&2
    sleep "$delay"
    delay=$(( delay * 2 ))
    (( attempt++ ))
  done
}
retry 5 2 curl -sf https://example.com/health
```
Key points an interviewer looks for: exiting non-zero on final failure (so calling scripts/CI can detect it), doubling delay between attempts (exponential backoff), and using `"$@"` so the wrapped command's own arguments/quoting are preserved correctly.

### 7. What's the difference between `>&2` and `2>&1`, and why log errors to stderr instead of stdout?
`command >&2` (inside a script, on an `echo`, e.g. `echo "error" >&2`) redirects that specific output to stderr. `2>&1` redirects stderr *to wherever stdout currently points* — commonly used to merge both streams into one file/pipe. Errors/diagnostics should go to stderr (not stdout) so that a script's actual *output* (stdout) can be safely piped/captured by another program (`result=$(my_script.sh)`) without diagnostic noise being mixed into the captured data — a script that prints errors to stdout can silently corrupt anything consuming its output programmatically.

### 8. What's the difference between a bash array and an associative array, and how do you iterate one?
A regular (indexed) array: `arr=(a b c)`; access `${arr[0]}`; iterate `for x in "${arr[@]}"; do ...; done`. An associative array (bash 4+): `declare -A map=([key1]=val1 [key2]=val2)`; access `${map[key1]}`; iterate keys with `for k in "${!map[@]}"; do echo "$k=${map[$k]}"; done`. Always quote `"${arr[@]}"` when iterating to correctly preserve elements containing spaces.

### 9. In Python, what's the difference between a list, a tuple, and a set, and when would a DevOps script use each?
List — ordered, mutable, allows duplicates; use for an ordered sequence you'll modify (e.g., accumulating log lines to process). Tuple — ordered, immutable; use for fixed structured data you don't want accidentally mutated (e.g., a function returning `(status, message)`). Set — unordered, unique elements, very fast membership testing (`O(1)` average); use when checking "have I already processed this ID" across a large collection, which would be `O(n)` per check with a list.

### 10. Why is it dangerous to use `subprocess.run(cmd, shell=True)` with any user-controllable input in Python?
`shell=True` passes the command string to the system shell for interpretation, meaning shell metacharacters in the input (`;`, `|`, `` ` ``, `$()`, `&&`) are interpreted as shell syntax rather than literal data — if any part of `cmd` comes from user input, this is a direct command injection vulnerability (e.g., a filename argument of `"; rm -rf / #"` could execute arbitrary commands). The safe pattern is `subprocess.run([cmd, arg1, arg2], shell=False)` — passing a list of literal arguments bypasses shell interpretation entirely, so metacharacters in arguments are treated as plain data, not syntax.

### 11. How do you handle configuration/secrets in a Python automation script without hardcoding them?
Read from environment variables (`os.environ.get("API_KEY")`) populated by the CI system's secret store or a local `.env` file (loaded via `python-dotenv`, with `.env` itself gitignored — never committed), or fetch dynamically at runtime from a secrets manager SDK (`boto3` for AWS Secrets Manager, `hvac` for Vault) using the script's own IAM role/service identity rather than an embedded static credential.

### 12. What's the difference between a Python virtual environment and a system-wide package install, and why does it matter for automation scripts?
A virtual environment (`venv`, `virtualenv`) creates an isolated Python environment with its own installed packages, independent of the system Python and other projects' environments. It matters because different scripts/projects often need different, potentially conflicting versions of the same dependency — without isolation, installing one script's dependencies can silently break another script (or the OS's own Python-based tools, on Linux distros that rely on system Python) — automation should always run inside a dedicated, reproducible (`requirements.txt`/lockfile-pinned) virtual environment or container rather than depending on ambient system-wide package state.

### 13. How would you write a Python script to parse a large log file for error patterns without loading the whole file into memory?
Iterate the file object directly line by line rather than calling `.readlines()` or `.read()` (which load the entire file into memory at once) — a file object in Python is itself a lazy iterator over lines:
```python
import re
pattern = re.compile(r"ERROR|CRITICAL")
with open("app.log") as f:
    for line in f:
        if pattern.search(line):
            process(line)
```
For genuinely huge files needing more complex multi-pass analysis, consider streaming/chunked processing or tools like `awk`/`grep` piped into Python, or a proper log-aggregation system rather than ad hoc parsing at all for recurring needs.

### 14. What's the difference between `==` and `is` in Python, and why does it matter when checking for `None`?
`==` checks value equality (can be overridden by a class's `__eq__`). `is` checks object identity (are these literally the same object in memory). `None` is a singleton (only one `None` object ever exists), so `is None` is both the idiomatic and safer check — it can't be fooled by a custom `__eq__` implementation on some other object that happens to compare equal to `None`, and it's marginally faster since it's a simple identity comparison rather than a method call.

### 15. Explain Python's GIL (Global Interpreter Lock) and why it matters for choosing between threading and multiprocessing for a DevOps automation task.
The GIL allows only one thread to execute Python bytecode at a time within a single process, even on a multi-core machine — meaning `threading` does *not* achieve true CPU parallelism for CPU-bound Python code (multiple threads still take turns on one core for pure-Python computation). However, threading still helps significantly for I/O-bound work (waiting on network requests, file I/O, subprocess calls) because the GIL is released during blocking I/O operations, letting other threads run — so a script making many concurrent API calls benefits from threading (or `asyncio`), while a script doing heavy CPU-bound computation (data processing, parsing) needs `multiprocessing` (separate processes, each with its own interpreter/GIL) to actually use multiple cores.

### 16. How would you make a Python script idempotent when it manages infrastructure resources directly (e.g., via a cloud SDK) rather than via Terraform?
Always check current state before acting — "does this resource already exist with this configuration" — and only create/modify if actually needed, rather than blindly issuing create calls every run (which would either fail on a duplicate or, worse, silently create duplicates if the API allows it). Use the cloud provider's natural idempotency keys/tags where available (many APIs accept a client-supplied idempotency token so a retried request with the same token doesn't double-create a resource), and design the script so that running it twice in a row produces the same end state and the second run reports "no changes needed" rather than erroring or duplicating work.

### 17. What's the difference between `try/except/finally` and a context manager (`with` statement) for resource cleanup in Python, and why prefer the latter?
`try/finally` requires manually writing cleanup code in every place a resource is acquired, and it's easy to forget or get subtly wrong (e.g., forgetting to close a file/connection in one of several exit paths). A context manager (`with open(...) as f:`, or a custom class implementing `__enter__`/`__exit__`) guarantees cleanup runs automatically and correctly on any exit path (normal completion, an exception, an early return) without the caller having to remember to write `finally` logic themselves — for automation scripts managing external resources (files, DB connections, SSH sessions, cloud API clients with connection pools), this reliability matters a lot, since a leaked connection/file handle in a long-running or frequently-invoked script compounds into real production issues (see file descriptor exhaustion in the Linux fundamentals file).

### 18. How would you structure a Python CLI automation tool so it's testable, rather than one large `if __name__ == "__main__":` script?
Separate the argument-parsing/CLI entry point from the actual business logic — put the real logic in plain, importable functions that take explicit parameters and return values (or raise exceptions) rather than reading from `sys.argv` or printing directly, so those functions can be unit-tested in isolation with mocked inputs. Use a proper CLI framework (`argparse`, `click`, or `typer`) for argument parsing rather than manually indexing `sys.argv`, and keep the `if __name__ == "__main__":` block as a thin wrapper that just parses args and calls into the testable logic — this separation is the single biggest factor in whether an "automation script" can actually be reliably tested and maintained as it grows, versus becoming an untested, fragile pile of top-level statements.
