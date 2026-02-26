---
name: write-pattern
description: Guide for writing new pattern pages in the pony-patterns documentation. Load when adding a new pattern or a new pattern category.
disable-model-invocation: false
---

# Writing Pony Patterns

Follow these conventions when adding new patterns to the pony-patterns documentation.

## Tone

Write casually, like you're explaining something to a friend. Be direct and concrete. Avoid being mechanical or listy ("it provides X, Y, and Z"). When something might seem magical or foreign to the reader, take a moment to explain where it comes from and what it does. Don't assume readers already know the standard library or advanced type system features.

Avoid excessive em dashes. Use periods, colons, semicolons, commas, and parentheses for variety.

## File Format

Every pattern page starts with this front matter:

```yaml
---
hide:
  - toc
---
```

The page structure is always:

- `# Pattern Name` (H1)
- `## Problem` (H2)
- `## Solution` (H2)
- `## Discussion` (H2)

No other H2 headings. Favor flowing narrative prose in Discussion over H3 subsections. If the discussion is long enough that distinct topics genuinely need signposting, subsections are acceptable, but the default should be connected prose with transitions between ideas.

## Problem Section

Start from the user's concrete need, not an abstract statement about software engineering. Frame it as something they're trying to do: "Your system needs X", "You want to enforce Y", "You have a value that could be one of several things".

Show a code example of the naive or problematic approach. After the code, explain why it falls short. Connect the prose to the code by naming specific functions or types from the example so the reader can follow along.

## Solution Section

Open with what the solution gives you. Talk about the guarantees and benefits before diving into the mechanics. The reader should understand why this approach is better before they see how it works.

Build up incrementally. Don't dump a wall of code and expect the reader to figure it out. Introduce one concept at a time with a small code snippet, then explain what's going on in it. Walk through the moving parts. If something could look magical ("where did `MakeConstrained` come from?"), explain it.

After walking through all the pieces, show a complete runnable program that ties everything together. This is the reference the reader can copy and adapt.

## Discussion Section

Favor flowing narrative over a list of bullet points under separate headings. Connect your points with transitions so the discussion reads as a coherent piece rather than a FAQ. Cover things like:

- Why the solution works the way it does (design constraints, language limitations)
- Limitations and things to watch out for
- How this pattern relates to other patterns in the collection (link to them)

When linking to related patterns, make sure the cross-references go both ways. If the new pattern references an existing one, update the existing pattern to reference the new one too.

## Workflow

### Creating the pattern file

Create the markdown file in the appropriate category directory under `docs/`. If no existing category fits, create a new one (see below).

### Category index pages

If the pattern belongs in a new category, create a `docs/<category>/index.md` with the same front matter (`hide: toc`), an H1 like "Category Name Patterns", and a brief paragraph describing what the category covers.

### Updating navigation

Add the pattern to the `nav` section in `mkdocs.yml`. Categories are listed alphabetically. Each category has an Overview entry pointing to its index page, followed by its patterns.

```yaml
  - Category Name Patterns:
    - Overview: 'category-name/index.md'
    - Pattern Name: 'category-name/pattern-name.md'
```

### Cross-references

When the new pattern relates to existing patterns, add links in both directions. The new pattern should link to the related ones in its Discussion, and the related patterns should be updated to mention the new one.

### Spell checking

The cspell config (`cspell.json`) excludes fenced code blocks and inline code from spell checking, so Pony identifiers in code won't be flagged. If your prose uses unusual words, check `.cspell/project-words.txt` and `.cspell/pony-terms.txt`. Add words only if they're genuinely needed and not already covered.

### Verification

After writing:

1. Run cspell to check spelling. If `npx` is not available, use Docker: `docker run --rm -v "$(pwd):/workdir" -w /workdir ghcr.io/streetsidesoftware/cspell:latest "docs/<category>/*.md"` plus any modified files.
2. Run `mkdocs build --strict` to verify the site builds without broken links. If mkdocs is not installed, create a venv: `python3 -m venv .venv && .venv/bin/pip install mkdocs mkdocs-material mkdocs-htmlproofer-plugin` then use `.venv/bin/mkdocs build --strict`. The `.venv` directory is already in `.gitignore`.
3. Confirm the nav ordering looks right.
