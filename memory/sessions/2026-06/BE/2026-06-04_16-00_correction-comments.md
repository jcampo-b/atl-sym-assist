# Correction — DEVSYM-319 comment noise

## What was wrong
Code review found two categories of comment noise in DEVSYM-319 implementation.

## What the correct approach is

### 1. Numbered step comments inside methods
Comments like `// 1. Fetch current task`, `// 2. Resolve current lifecycle state`
restate what the code already says. They add no value. If a method needs a map
to be understood, the method is too long — split it. If it's the right length,
the code speaks for itself.

### 2. Duplicate TODOs
Stub rules had the same TODO in both the docblock and the inline comment.
One is enough. Keep the inline TODO inside the method body. Remove it from the docblock.

## Rule to apply going forward
- Only comment when the WHY is non-obvious — not the WHAT.
- Never restate what the code already expresses.
- Never duplicate a TODO in both docblock and inline. One location only: inside the method body.
- An empty class body does not need a comment placeholder.
