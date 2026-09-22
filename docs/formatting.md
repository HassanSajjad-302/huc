# HUC Default Formatting

Status: default style for HUC 0.1; a formatter is not implemented yet

This guide gives HUC one familiar layout across projects. It is not an extra
set of compiler rules. Different indentation, line wrapping, or operator
spacing remains valid when it preserves the tokens and their meaning.
Projects may enforce this style with a future formatter without making it a
requirement for compiling HUC.

The [language specification](language-specification.md) and
[grammar](huc.ebnf) define required punctuation. In particular, declarations
use `name: Type`, control-flow bodies require braces, and statements retain
their semicolons. Digit separators and trailing commas are optional syntax.

## Indentation and blocks

Use four spaces per indentation level, not tabs. Aim for at most 100 columns;
do not change string contents or split an indivisible token to fit that width.
Remove trailing whitespace and end each file with a newline.

Put an opening brace on the same line as the completed header. Put a closing
brace on its own line, except for an empty block or a following `else`:

```huc
if (ready) {
    run();
} else if (waiting) {
    wait();
} else {
    stop();
}
```

Use a blank line between top-level declarations. Avoid aligning unrelated
declarations into columns; renaming one should not require reformatting all
the others.

## Spaces and declarations

- Use one space after a comma and after a declaration colon, none before them.
- Use spaces around binary operators, assignment, and `->` in a return type.
- Keep prefix operators next to their operands, such as `!ready`, `*pointer`,
  and `&value`. Binary `&` still gets surrounding spaces: `left & right`.
- Keep calls tight: `run(value)`, not `run ( value )`.
- Put a space after a control-flow keyword: `if (ready)`.
- Keep type suffixes attached: `Widget*` and `Widget#`.
- Put binding `mod` before the name and pointee `mod` inside the type.

```huc
let mod owner: mod Widget# = new Widget(42);
let observer: Widget* = owner;

fn inspect(widget: Widget*) -> i32 {
    return widget->value;
}
```

These are presentation choices, not restrictions on operator spacing.
For example, `a+b`, `a + b`, and `a+ b` are all valid binary additions in HUC.

## Lists and trailing commas

Keep a short list on one line without a trailing comma. When a list wraps,
normally put each item on its own line and include a trailing comma:

```huc
process(first, second);

process(
    first,
    second,
);

fn combine(
    left: i32,
    right: i32,
) -> i32 {
    return left + right;
}
```

Use the same convention for phase binders, phase requests, specialization
patterns, and constructor initializer lists. In a multiline constructor
initializer list, put the body brace on the next line after the final comma:

```huc
fn init(x: i32, y: i32)
    : x(x),
      y(y),
{
}
```

The compiler accepts either trailing-comma choice on one line or several
lines. An empty list stays `()`, never `(,)`. A parenthesized condition or a
grouped expression is not a list and does not gain a trailing comma.

## Numbers and comments

Use digit separators when they help the reader: `1_000_000`, `0xFFFF_0000`,
or `0b1010_0011`. Plain `1000000` remains fine. There is no compulsory digit
count or grouping size, and a formatter should preserve the author's valid
grouping rather than guess whether digits represent a count, mask, or code.

Prefer `//` comments, with one space after `//`. An inline comment gets one
space after the code. Keep existing non-nesting `/* ... */` comments available.
Formatting must preserve comment boundaries and string/character contents.

This guide does not introduce interpolation, implicit returns, destructuring,
optional semicolons, omitted type annotations, or another declaration order.
