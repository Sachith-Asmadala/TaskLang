# TaskLang++: code learning and interview guide

Reviewed snapshot: `b52087ddf6f0a4de6d0971314a7f8eeb18bc9ec9`. Examples below are source traces; this documentation change does not claim a new build/test execution.

## Problem and implemented scope

TaskLang++ is a small assignment-focused language for declaring tasks, script names, schedules, conditions, and dependencies. Flex tokenizes input; Bison parses grammar rules; C code performs semantic checks and prints tasks in dependency order. It simulates execution by printing descriptions. `RUN "backup.sh"` does not execute a shell script, and a schedule string does not install a real timer.

## Startup and main components

The Makefile generates a lexer/parser and compiles a `tasklang` executable. When run with a `.tl` file redirected to standard input, [parser.y:main](https://github.com/Sachith-Asmadala/TaskLang/blob/b52087ddf6f0a4de6d0971314a7f8eeb18bc9ec9/parser.y) calls `yyparse()`, validates semantics, checks lexical/parser error counts, and returns failure if errors exist. Only a valid program reaches `run_simulation()`.

| File or function | Responsibility |
| --- | --- |
| [lexer.l](https://github.com/Sachith-Asmadala/TaskLang/blob/b52087ddf6f0a4de6d0971314a7f8eeb18bc9ec9/lexer.l) | Recognize keywords, names, strings, times, braces, comments, and invalid characters |
| `parser.y` grammar rules | Define task/statement syntax and call semantic helper functions |
| `begin_task()`, `end_task()` | Reset the current-task accumulator and store a completed task |
| `apply_before_dependencies()` | Convert BEFORE declarations into dependencies on the target task |
| `validate_semantics()`, `dfs_cycle()` | Check references, conditions, and circular dependencies |
| `topo_sort()`, `run_simulation()` | Calculate and print a dependency-respecting order |
| [run_tests.py](https://github.com/Sachith-Asmadala/TaskLang/blob/b52087ddf6f0a4de6d0971314a7f8eeb18bc9ec9/run_tests.py), `tests/*.tl` | Exercise valid and invalid language examples |

## Walkthrough: a backup followed by a report

The existing [tests/valid_02.tl](https://github.com/Sachith-Asmadala/TaskLang/blob/b52087ddf6f0a4de6d0971314a7f8eeb18bc9ec9/tests/valid_02.tl) contains these tasks, plus a cleanup task:

```text
TASK backupDB {
    RUN "backup.sh"
    EVERY DAY AT 02:00
}
TASK sendReport {
    RUN "report.py"
    AFTER backupDB
    IF success
}
```

1. Flex returns tokens such as `TASK`, `IDENTIFIER`, `LBRACE`, `RUN`, and `STRING_LIT`. A string/name token supplies its text through `yylval.str`.
2. Bison recognizes a task declaration and invokes `begin_task()`. Statement actions populate the current script, schedule, dependencies, and condition.
3. `end_task()` checks for duplicate names and a missing RUN statement, then copies the fields into `tasks[]`, subject to fixed capacity limits.
4. After parsing all declarations, `validate_semantics()` resolves dependency names and uses DFS to detect cycles. `sendReport` has a dependency on `backupDB`; an undefined `backupDB` would be a semantic error even if the syntax parsed correctly.
5. `topo_sort()` calculates each task's dependency count, puts zero-dependency tasks into an array queue, and releases dependent tasks as prerequisites are emitted. The backup appears before the report. Independent cleanup has no required order relative to them.
6. `print_task()` prints the script, schedule, dependencies, and condition. `IF success` is recorded and validated for dependency presence; no real script result is evaluated.

## Data structures and why they fit

`Task` is a C struct containing fixed-size character arrays, dependency-name arrays, counters, and DFS state. `tasks[MAX_TASKS]` permits up to 64 tasks, with up to 16 entries in each dependency/before list. This keeps assignment storage understandable but imposes explicit limits.

The workflow is a directed graph represented using task records and dependency names. `find_task()` performs a linear name search. DFS uses `visited`, `in_stack`, and `cycle_stack[]` to distinguish completed traversal from an active path. An edge back to an active task indicates a cycle. Topological sorting uses `in_degree[]`, `queue[]`, and `order[]` with front/back indexes.

Do not describe this implementation as an optimized O(V+E) graph library. It repeatedly scans task/dependency arrays and resolves names linearly. The graph concepts transfer to larger systems, but this representation favors small, readable inputs.

## Errors and decisions

The lexer counts invalid characters, invalid times, and overlong tokens. Keyword rules appear before identifiers so reserved words are recognized correctly. Valid clock tokens are restricted to 00:00–23:59. The parser reports syntax errors with line information and has recovery rules; semantic helpers report duplicates, missing fields, undefined dependencies, and cycles. Counters determine the final nonzero exit status.

Token strings allocated by the lexer are released in grammar actions, with a Bison `%destructor` for discarded string-valued symbols. This distinction matters because parser recovery can discard values that never reach normal actions. Allocation-failure handling could still be improved.

## Limitations and sensible next work

There is no actual script execution, timer service, persisted scheduler state, or runtime success/failure handling. Fixed task/string capacities limit input. `run_simulation()` has a fallback for a partial topological order, but the normal main path rejects cycle errors before simulation. Keep these boundaries clear in a demo.

Useful future checks: assert the actual ordering in output, verify multiple diagnostics/recovery, and test boundary capacities. Add name-to-index lookup or integer adjacency lists only if larger inputs justify the added complexity. Preserve the assignment syntax and simple build process.

## Contribution evidence

GitHub attributes [commit b52087d](https://github.com/Sachith-Asmadala/TaskLang/commit/b52087ddf6f0a4de6d0971314a7f8eeb18bc9ec9) to `Sachith-Asmadala`; its diff touches lexer/parser error handling, tests, build tooling, CI, and documentation. Confirm which changes you made and how assistance was used. This evidence does not prove sole authorship of the complete language.

## Five interview questions and model answers

1. **What is the difference between lexical, syntax, and semantic errors?** An unexpected character is lexical; an invalid arrangement of valid tokens is syntactic; a dependency on an undeclared task is semantic.
2. **How is a cycle detected?** DFS marks tasks currently on its recursion path using `in_stack`. An edge to a task still on that path indicates a cycle, and `cycle_stack` helps print it.
3. **Why use topological sorting?** A prerequisite must appear before a dependent task. The queue starts with tasks having no remaining prerequisites and releases others as counts reach zero.
4. **Why are keyword rules before the identifier rule?** Flex chooses the longest match and uses rule order for ties. A reserved word matching the identifier pattern must return its keyword token.
5. **Does this execute scheduled jobs?** No. The implementation validates declarations and prints a simulated order. Script execution, timing, and real condition evaluation would require additional runtime components.

## Practice

Trace two tasks by hand using `tasks`, `in_degree`, and the queue. Add a dependency from the backup back to the report on paper and identify where DFS detects the cycle. Explain why a syntactically valid program can still fail.
