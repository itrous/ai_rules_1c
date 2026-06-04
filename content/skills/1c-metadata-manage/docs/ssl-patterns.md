# 1C SSL/БСП Subsystems Reference

For basic SSL usage (attribute access, user messages) — see `dev-standards-architecture.md §4 → "Data Access — Reference Attribute Access"` and `dev-standards-core.md §2`.

## When to Use

Invoke this skill when:
- Working with users and access rights
- Working with files and attachments
- Implementing print forms
- Managing background jobs
- Working with object versioning
- Sending emails
- Need common utility functions (arrays, structures, strings)

## Core Principle

**ALWAYS check if SSL has a solution before writing custom code.**

## SSL Search Workflow

When implementing new functionality:

1. **First, search SSL** — use `search` action `search_code` (semantic) / `find_code` (lexical) over the project's БСП modules (in `src/cf`) with keywords describing your need; **v8std** for БСП-related standards
   - Example: `search` action `search_code` "фоновое задание прогресс"
   - Example: `search` action `search_code` "копирование структуры"

2. **Check existing patterns** — use `search` action `find_code` (lexical) / `search_code` (semantic) to find how similar tasks are solved in the codebase

3. **Use SSL if available** — it's tested, optimized, and maintained

4. **Only then write custom code** — and document why SSL wasn't suitable

## Key SSL Modules

- **Пользователи** — users, roles, access rights
- **РаботаСФайлами** — file storage and attachments
- **УправлениеПечатью** — print forms
- **ДлительныеОперации** — background jobs with progress
- **ВерсионированиеОбъектов** — object history
- **РаботаСПочтовымиСообщениями** — email sending
- **ОбщегоНазначения** / **ОбщегоНазначенияКлиентСервер** — common utilities
- **СтроковыеФункцииКлиентСервер** — string functions

---

**Remember**: SSL is your first stop for common functionality. Writing custom code when SSL has a solution is technical debt.
