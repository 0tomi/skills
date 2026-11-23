---
name: reduce-slop
description: Reduces slop by reviewing the code with good practices.
---

Review the code suggested by the user according to the following criteria:

* Follow the YAGNI principle.
* Follow SOLID principles where applicable.
* Enforce SRP: keep presentation decoupled from persistence and database queries.
* Avoid high cyclomatic complexity unless it is justified.
* Do not introduce unnecessary technical debt.
* Look for overengineering, unnecessary wrappers, unnecessary abstractions, unnecessary functions, and anything that could be implemented more simply.
* Look for duplicated code and dead code.

