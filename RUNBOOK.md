# Daily run — instructions for Claude

The 9 AM scheduled task follows this file. Edit it to change behaviour; changes apply from the next run.

Recipients: rushikeshbhaskar7@gmail.com, patelsavan5553@gmail.com

## 1. Pick up new books
For each folder in `books/` with no entries in `queue.json`: append one item per numbered chapter PDF
(`<n>._<Title>.pdf`, sorted by n; skip front matter like About_the_Author / Table_of_Contents).
Book name = folder name in title case. A chapter over 45 pages becomes two items (`part` "1/2" / "2/2", `pages` split in half).

## 2. Choose today's item
`item = queue[progress.next]`. If `next >= len(queue)`, send one short email saying all books are finished
and suggesting adding a new PDF to `books/<book-slug>/`, then stop.

## 3. Make sure the summary exists
If `item.summary` is missing:
- `pdftotext -layout <pdf>` (use `-f/-l` with `item.pages` when set) and read the whole text.
- Write the summary in your own words (no quoted passages over a short phrase), **~10 min read**, matching the
  structure and depth of `summaries/ai-native-software-engineering/01-the-ai-native-engineer.md`:
  TL;DR · Key concepts (explained, with the chapter's examples/code described) · Perspective & trade-offs ·
  Apply it (2–3 concrete actions) · Think about it (2 reflection questions).
- Reader: full-stack dev (Node/TS/Next.js microservices, Kafka, Redis, Postgres, Docker, Python), learning K8s/DevOps.
  Relate examples to that stack where it helps.

## 4. Email it
Gmail `send_message` to all recipients (in `to`):
- Subject: `📘 <Book> — Ch <n>: <Title>` (+ ` (part x/2)` when split)
- `htmlBody`: the summary rendered (python `markdown` with `tables`; leave a blank line before every list) as clean HTML (headings, lists, tables, `<code>`), max-width ~680px, readable on phone.
  Footer: `Next: <next item title>` and position `<k>/<total>`.
- If the email fails, do NOT advance progress; finish and report the error.

## 5. Record progress
`progress.next += 1`, append `{index, title, date}` to `progress.sent`. Commit
`summary: <book> ch<n>` (include any new summary file) and push to `main`.
