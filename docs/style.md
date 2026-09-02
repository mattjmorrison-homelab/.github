# Doc and comment style

READMEs, doc comments, and code comments describe the system as it is now
— they are not changelogs. Git history and commit messages are where the
story of how something changed belongs, if anywhere.

- Don't narrate how something got to its current state ("first attempt
  did X, that broke because Y, so then Z" / "confirmed live" / "second bug
  found"). State the current design and the reason for it.
- Don't document absences or rejected alternatives ("no X annotations
  anywhere, plain Y instead"). Only document what a mechanism *is* and the
  real constraint that shapes it. A rejected alternative not worth a
  commit message isn't worth a "not X" clause in a doc either.
- Be concise. Get to the point with the minimum text needed to explain why
  something is the way it is.
