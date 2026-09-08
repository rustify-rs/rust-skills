# Contributing

## Adding a skill

1. **Open an issue first.** Describe the one axis of judgement the skill covers and two or
   three before/after pairs. If it overlaps an existing skill, say why it's separate.
2. A skill is one directory under `skills/<name>/` containing `SKILL.md`.
3. `SKILL.md` front matter:

   ```yaml
   ---
   name: rust-<topic>
   description: >
     What it reviews, and the trigger phrases. One paragraph. End with what it is NOT
     (e.g. "Not a lint pass").
   ---
   ```

## Quality bar

A skill ships only if it clears all four:

- **Not a clippy restatement.** If `cargo clippy` already flags it, it doesn't belong here
  unless the skill adds a design rationale clippy can't express.
- **Judgement, not commandments.** Every guideline states *why*, and gives the case where
  the opposite is correct.
- **Before / after.** Every point shows bad code and good code, both compilable in spirit.
- **Checklist.** The file ends with a `## Review checklist` of mechanical `- [ ]` items.

## Style

- No em dashes. No decorative horizontal rules in prose.
- Error messages, identifiers, and code follow the conventions the skill itself teaches.
- Keep `SKILL.md` under ~250 lines. Split into a second skill before it sprawls.

## Changing a skill from field use

Most edits after the initial version should come through the learnings loop, not direct
PRs:

1. A correction gets captured to `LEARNINGS.md` (`status: pending`) from real work.
2. `skill-maintainer` batches pending entries, decides promote / merge / reject behind a
   review gate, edits the rubric, bumps `.claude-plugin/plugin.json` `version`, and adds a
   `CHANGELOG.md` line.

A direct PR editing a rubric body should either reference a folded `LEARNINGS.md` entry or
justify why it bypasses the loop (typos, broken examples, formatting).

## Testing locally

```bash
claude --plugin-dir ./
```

Then run `/rust-skills:<name>` against a real file. Or copy a single skill:

```bash
cp -r skills/<name> ~/.claude/skills/
```

Run `claude plugin validate ./` before opening a PR.
