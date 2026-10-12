---
name: bash_string_comparison_if
description: How to compare a variable with a string in bash if statements using [[ ]] or POSIX [ ], with pitfalls
tags:
- bash
- shell
- string
- comparison
- if
- posix
created_at: '2026-09-30T13:03:28.953171984+02:00'
---

# Bash string comparison in if statements

## Preferred: `[[ ]]` (bash builtin)

```bash
value="hello"

if [[ "$value" == "hello" ]]; then
    echo "match"
fi

if [[ "$value" != "world" ]]; then
    echo "not world"
fi
```

- `==` and `=` are equivalent for equality; `!=` for inequality.
- `[[ ]]` does not word-split or glob-expand unquoted variables, so it is safer than `[ ]`.
- Quoting `"$value"` is still recommended (empty values, spaces).

## POSIX sh: `[ ]` (always quote!)

```sh
if [ "$value" = "hello" ]; then
    echo "match"
fi
```

Unquoted `[ $value = "hello" ]` breaks when `$value` is empty or contains spaces.

## Extras

```bash
[[ -z "$value" ]]                      # empty string
[[ -n "$value" ]]                      # non-empty string
[[ "$value" == h* ]]                   # glob pattern match (unquoted right side)
[[ "$value" == "h*" ]]                 # literal "h*" comparison (quoted)
[[ "$value" == "foo" || "$value" == "bar" ]]   # OR
shopt -s nocasematch                   # case-insensitive comparisons
```

## Pitfalls

- `[ ]` with unquoted variables → word splitting / "unary operator expected" errors.
- In `[[ ]]`, the right side of `==` is a glob pattern unless quoted.
- Numeric comparison uses `-eq`, `-ne`, `-lt`, etc. — `==` on numbers compares them as strings ("01" != "1").
