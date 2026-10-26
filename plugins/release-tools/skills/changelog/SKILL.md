---
name: changelog
description: Write the changelog entry for the changes since the last release. Use when the user asks for a changelog entry, release notes for CHANGELOG.md, or "what changed since the last tag".
license: MIT
---

# Changelog entry

1. From the project directory, run `scripts/commits.sh` (in this skill's folder) to list the commits since the last tag.
2. Read `references/format.md` in this skill's folder and group the changes the way it says.
3. Add the entry at the top of CHANGELOG.md for the version the user named, or under "Unreleased" if they named none. Create the file if there is none.

Describe what changed for the people who use the project, not how the code changed. Leave out merges and changes to CI.
