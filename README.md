# book-summaries

Daily 9 AM (UK) email with a summary of the next chapter, run by a Claude scheduled task.

- `books/<book-slug>/` — chapter PDFs (`<n>._<Title>.pdf`). **Keep this repo private** — books are licensed content.
- `summaries/` — one Markdown summary per chapter (written on first send, editable).
- `queue.json` — reading order. `progress.json` — `next` = index of tomorrow's chapter.
- `RUNBOOK.md` — what the daily run does.

**Add a book:** push its chapter PDFs into a new `books/<book-slug>/` folder. The next run queues it after the current book.
**Skip / repeat a chapter:** edit `progress.next`.
