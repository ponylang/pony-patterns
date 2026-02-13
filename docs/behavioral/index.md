---
hide:
  - toc
---

# Behavioral Patterns

How should you organize your actor when its behavior depends on its current state? You could track the state with a flag and check it in every behavior, but that gets old fast. The patterns in this chapter offer better approaches, using Pony's type system to make the structure of your actor's behavior explicit and compiler-checked.
