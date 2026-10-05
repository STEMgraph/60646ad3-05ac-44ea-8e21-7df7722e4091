<!---
{
  "id": "60646ad3-05ac-44ea-8e21-7df7722e4091",
  "teaches": "Debugging: Reproducing a Bug as a Test Protocol and make test",
  "depends_on": [
    "960cf9b4-6321-433d-a08d-9aacffccd4e2",
    "ec9e6408-a2c2-4ef3-9546-0212928a9abb"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-05",
  "keywords": [
    "reproduction",
    "test protocol",
    "lab notebook",
    "make test",
    "regression test",
    "minimal example",
    "scientific method"
  ]
}
--->

# Debugging: Reproducing a Bug as a Test Protocol and `make test`

> In this exercise you receive a program and a vague bug report. Before touching a debugger you will turn the report into a written, reproducible test protocol, as you would in a measurement lab, then into a minimal failing example, and finally into an automated `make test` that fails today and passes once the bug is fixed. No `sudo` is required.

## Introduction

"It sometimes prints the wrong number" is not a bug, it is an observation. Engineering starts when the observation becomes a **protocol**: exact input, exact command, exact environment, observed output, expected output. In a measurement lab you learned to write this down in a lab notebook before interpreting anything. Software is no different, except that the protocol can be executed by a machine.

That is the goal here: a bug report becomes a shell script or a test program that *fails*. Then the debugger finds the cause, the fix makes the test pass, and the test stays in the project so the bug can never silently return. This is what professional teams mean by "reproduce first" and "regression test".

### 1.1) Further Readings and Other Sources

* [Stack Overflow: How to create a Minimal, Reproducible Example](https://stackoverflow.com/help/minimal-reproducible-example)
* [Andreas Zeller: Why Programs Fail, chapter "Reproducing Problems"](https://www.whyprogramsfail.com/)
* [GNU Coding Standards: Standard Targets (`check`)](https://www.gnu.org/prep/standards/html_node/Standard-Targets.html)

## Tasks

### Task 1: The Report and the Program

A colleague writes: "The statistics tool gives a wrong median for some files. Worked yesterday."

```bash
mkdir -p ~/dbg-repro && cd ~/dbg-repro
```

```c
// file: median.c
#include <stdio.h>
#include <stdlib.h>

static int cmp(const void *a, const void *b) { return *(const int *)a - *(const int *)b; }

double median(int *v, int n) {
    qsort(v, n, sizeof v[0], cmp);
    if (n % 2 == 1) return v[n / 2];
    return (v[n / 2] + v[n / 2 - 1]) / 2;        /* integer division */
}

int main(int argc, char **argv) {
    (void)argc; (void)argv;
    int v[1024], n = 0;
    while (n < 1024 && scanf("%d", &v[n]) == 1) n++;
    if (n == 0) { fprintf(stderr, "no input\n"); return 1; }
    printf("%.1f\n", median(v, n));
    return 0;
}
```

```bash
gcc -Wall -Wextra -O0 -g median.c -o median
```

### Task 2: Write the Protocol Before You Debug

Create `PROTOCOL.md` and fill it in while you experiment. Every line is a fact, not an opinion:

```markdown
# Bug: wrong median

- Date, machine, compiler version (`gcc --version`), commit
- Command: `printf '1 2 3\n' | ./median`
- Observed: ...
- Expected: 2.0
- Command: `printf '1 2 3 4\n' | ./median`
- Observed: ...
- Expected: 2.5
```

Run at least six inputs: odd count, even count, negative numbers, duplicates, a single value, and `printf '3 1 2\n'` to check that order does not matter. Record every result. Which inputs fail? What do they have in common?

### Task 3: Minimise

Find the *smallest* input that shows the bug. Write it into the protocol as "Minimal failing example". Then state a hypothesis in one sentence, also in the protocol.

### Task 4: The Protocol as a Script

A protocol that a machine can run cannot be misread:

```bash
# file: tests/run.sh
#!/bin/bash
set -u
fail=0
check() {                          # check "input" expected
    got=$(printf '%s\n' "$1" | ./median)
    if [ "$got" = "$2" ]; then echo "PASS  [$1] -> $got"
    else echo "FAIL  [$1] -> $got (expected $2)"; fail=1; fi
}
check "1 2 3"      2.0
check "1 2 3 4"    2.5
check "3 1 2"      2.0
check "5"          5.0
check "-3 -1 -2 -4" -2.5
check "2 2 2 2"    2.0
exit $fail
```

```bash
chmod +x tests/run.sh && ./tests/run.sh; echo "exit $?"
```

The script fails. Good: that is the protocol, executable.

### Task 5: `make test`

```make
CC = gcc
CFLAGS = -Wall -Wextra -O0 -g
median: median.c
	$(CC) $(CFLAGS) $< -o $@
test: median
	./tests/run.sh
.PHONY: test
```

```bash
make test; echo "exit $?"
```

`make` propagates the script's exit status. A CI system would mark this build red.

### Task 6: Now the Debugger

With the minimal example from Task 3, run `gdb ./median`, `break median`, `run < <(printf '1 2 3 4\n')`, step to the `return` line and `print (v[n/2] + v[n/2-1]) / 2` versus `print (v[n/2] + v[n/2-1]) / 2.0`. Confirm the hypothesis, fix the line, rebuild:

```bash
make test
```

All green. Write "Cause" and "Fix" into the protocol with the commit hash.

### Task 7: Keep the Test, Break It on Purpose

Reintroduce the bug, run `make test`, watch it fail, fix it again. Then add a test for the edge case the report did not mention: an empty input must exit with status 1 and print `no input`. Extend `run.sh` with a check on the exit status.

## Questions

- What is the difference between an observation and a reproducible bug? Give the criteria your protocol had to meet.
- Why write the protocol *before* opening the debugger, when the debugger would have found the integer division in two minutes?
- What made `printf '3 1 2'` a useful test case although it passed?
- Why does `make test` return a non-zero exit status, and who consumes that?
- The test from Task 4 would have caught this bug "yesterday". What is the cost of keeping it, and what is the cost of not keeping it?
- In the measurement lab you record instrument, settings and ambient conditions. What are the equivalents for a software bug protocol?

## Advice

The most expensive debugging sessions are the ones spent on bugs that are not reproducible yet. Writing the protocol first feels slow and is the fastest path, because it forces you to find the minimal failing input, and the minimal input usually points straight at the cause.

Turn every protocol into a test and keep it. A project with `make test` and a growing `tests/` directory is a project where bugs die once. A project without it is a project where the same bug is debugged by every new team member.
