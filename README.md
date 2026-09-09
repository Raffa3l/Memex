# Memex

[Deutsch](README.de.md) · English

A personal knowledge system that an LLM builds and maintains. Markdown as storage, a single HTML file as the
view, one Python script for PDFs. No framework, no database, no vector search.

The name Memex refers to Vannevar Bush's idea from 1945: a personal, curated store of knowledge in which the
connections between documents are as valuable as the documents themselves. Bush could not solve who does the
maintenance. The LLM can.

Memex is the whole: inbox, raw sources, schema, scripts, view. The wiki is its core, the collection of pages the
LLM writes and maintains from the sources. Whenever this file says "wiki", it means that layer.

This is an idea file. It is meant to be pasted into an LLM agent, whichever one you use: Claude Code, Codex,
Gemini CLI, OpenCode, Aider or anything else. It describes how I built my system, why, and what the three scripts
do. Your agent can build its own version from it in one or two sessions. The file stands on its own and needs
nothing else.

## The idea

The usual way to work with an LLM over documents is lookup: you upload files, the model searches for matching
passages on every question and answers from them. That works, but nothing accumulates. Every question starts
from zero. A question that connects five documents has to be reassembled by the model every single time.

The other way: the LLM builds a persistent wiki from the sources. Every new source is read once, condensed and
worked into existing pages. Cross-references, contradictions and summaries are already there when the question
arrives. Answers worth keeping are filed as pages into the wiki. The wiki gets richer with every source and every
question.

I do not write the wiki myself. I supply sources, ask questions and read. The LLM does the bookkeeping:
summarising, linking, filing, maintaining the index and the log. People abandon wikis because the maintenance
grows faster than the value. An LLM does not get tired, never forgets a cross-reference, and can touch fifteen
pages in one pass.

The pattern is not tied to a field. My sources are technical handbooks of over a thousand pages, government
reports and research reports. It works for any area where knowledge builds up over months from many documents:
research, reading a book, standards and regulations, project knowledge, hobbies.

## Four decisions

I wanted something that runs in a browser, needs no installation and gets by with as little tooling as possible.
That led to four decisions:

1. **Markdown is the truth.** The LLM writes Markdown, nothing else. The HTML file is generated from it and
   never touched by hand.
2. **One HTML file is the view.** A short Python script assembles all wiki pages into a single `index.html`
   with navigation by page type, backlinks, full-text search and formulas. It runs by double-click from a folder
   and just the same on a web host. No static-site generator, because they need a server and bring hundreds of
   packages.
3. **PDFs become Markdown only.** Docling converts, a script splits at headings. No JSON, no images, no
   splitting by text length. The original PDF stays next to the Markdown so the LLM can look at the page itself
   for formulas and figures.
4. **One Python environment, three scripts.** Convert, build, check. Nothing else.

## Structure

Three layers with clear ownership, plus the view:

```
memex/
├── <schema>.md        Schema: rules, page types, workflows. The agent reads it at every start.
├── inbox/             Inbox: unprocessed PDFs. Empty means: everything processed.
├── raw/<source>/      Raw sources: PDF, full text, index, sections. Read only.
├── wiki/              The wiki: index.md, log.md, home.md and all pages. Flat, no subfolders.
├── tools/             convert.py (PDF → raw), build.py (wiki → site), lint.py (health check)
└── site/index.html    The whole view in one file.
```

**Raw sources.** You drop them in, the LLM reads, nobody edits. Every source gets a short key (`handbook-xy`,
`standard-123`) and a folder. It holds the PDF, a `_fulltext.md` with page markers as an archive, an `_index.md`
with the chapter list, and one file per section with title, source and PDF page range in the front matter.
Books with hundreds of sections also get a `_sections.md` that the LLM searches directly, so the index stays
short.

**Wiki.** The LLM writes, you read. Flat, one file per page, the file name is the link name. Every page starts
with a front matter block: type (`source`, `concept`, `entity`, `project`, `answer`, `meta`), maturity
(`seedling`, `growing`, `evergreen`), date and the keys of the sources it rests on. Links as `[[filename]]`.
Citations with work, section and PDF page.

```yaml
---
title: Page title
type: concept
status: growing
updated: 2026-09-09
sources: [handbook-xy, standard-123]
---
```

**Schema.** The most important file. Every agent has its own file name for it, which it reads automatically at
start. It tells the LLM what lives where, what pages look like and which steps each workflow has. It grows with
experience; every change to it gets a log entry.

**Two special files.** `wiki/index.md` is the catalogue: every page with a one-line description, grouped by
type. The LLM reads it before every answer and updates it on every change. `wiki/log.md` is the history:
append-only, one entry per operation with date and kind (`## [2026-09-09] ingest | Title`), followed by two to
five lines. At a few hundred pages the index replaces any search infrastructure; the log gives the LLM the
context of the last sessions.

## The three scripts

All tooling is three Python files, a few hundred lines in total, written by the agent in one session. The only
dependencies: `docling` (IBM, MIT licence) for PDFs and `markdown` for the view. This section is written for the
agent and is technical accordingly.

**`convert.py`: PDF → `raw/<key>/`.** Called with PDF path, key and title.

1. Docling with `do_ocr=False` (when the PDFs have a text layer), tables in "accurate" mode, formula
   enrichment off, device `auto`, no images.
2. Export of the whole document as Markdown with `page_break_placeholder`. The marker only appears between
   non-empty pages, so the page sequence is derived from `doc.iterate_items()` (page number of each element in
   order). Result: `_fulltext.md` with `<!-- PAGE n -->` before every page. With it the split can be repeated in
   seconds at any time (`--split-only`).
3. Splitting: every heading becomes an atom with the text up to the next heading. Depth from the numbering
   ("2" → 2, "2.4" → 3, "2.4.5" → 4, "1.2.7-6" → 5); unnumbered headings count as siblings of the last numbered
   one; enumerations like "2. Example" do not count. Atoms are grouped at headings down to `--level` (default 2).
   Groups above `--max-chars` (40,000) are split recursively at deeper headings, if necessary at blank lines by
   size. Groups below `--min-chars` (2,500) move to their neighbour: tables of contents and short chapter heads
   forward into the following section, everything else onto the previous one. Index, glossaries and advertisers
   are dropped.
4. Writing: `NNN_title.md` with front matter `title`, `source`, `pages` (PDF page range), H1 and text. Control
   characters from broken formulas are replaced and the file marked `formulas_damaged: true`. `_index.md` with a
   header (work, PDF path, page count, date, parameters), chapter list (the topmost numbered level with at least
   ten entries) and the section table; above 200 sections the table moves to `_sections.md`.
5. The PDF is moved from `inbox/` to `raw/<key>/`.

**`build.py`: `wiki/*.md` → `site/index.html`.** Reads all pages, parses the front matter with a small parser
of its own (keys, lists in square brackets), protects formulas (`$…$`, `$$…$$`) from the Markdown parser with
pure-ASCII placeholders, replaces `[[target|text]]` with links to `#target` (resolved by file name or title,
missing targets as a dashed hint), converts Markdown with the extensions tables, footnotes and code blocks,
computes backlinks. Output: one HTML file with embedded CSS and JavaScript. Sidebar by type, every page as a
hidden `<section id="slug">`, page switching via `hashchange`, search as a substring filter over the text of all
pages with a snippet, meta line (type · maturity · date · sources), backlinks footer, KaTeX from a CDN only when
a page contains a `$`, light and dark design via `prefers-color-scheme`. Reports missing link targets.

**`lint.py`.** Checks every wiki page: front matter present, type and maturity from the allowed set, date in
`YYYY-MM-DD`, file name lowercase-ascii-hyphen, source keys with an existing folder, dead links, orphans (pages
only linked from the index or the log), pages missing from the index, log entries off the standard format.
Errors exit with 1, hints are text only.

## Workflows

**Inbox.** I drop a PDF into `inbox/` and say "new file in inbox". The LLM proposes key and title, converts,
checks the source index and moves the PDF into its source folder.

**Ingest.** "ingest handbook-xy". The LLM reads the source index and sections, discusses the key takeaways with
me, writes a source page, creates or updates concept and entity pages, flags contradictions with existing pages,
updates index and log, builds the view. One source may touch ten to fifteen pages. Books are not ingested in one
go. They get a source page with a chapter overview; chapters are read when a question needs them. That way the
wiki grows where you work.

**Query.** I ask. The LLM reads `wiki/index.md` first, then the matching pages, the raw sources if needed.
Answer with citations. If the answer is worth keeping (a comparison, an analysis, a basis for a decision), it is
filed as an `answer` page in the wiki and linked from the concept pages concerned. That way explorations do not
vanish into the chat history.

**Lint.** `tools/lint.py` finds dead links, orphans, pages without sources and pages missing from the index.
On top the LLM checks contradictions between pages, stale claims, concepts without a page of their own and
missing cross-references. Findings land as a task list on the home page.

**End of session.** Build the view, check the log. If you version the wiki, record the state now with a note by
kind of operation (`ingest:`, `query:`, `lint:`, `schema:`).

## PDF to Markdown: what I learned

This was the most laborious part, and the mistakes are instructive.

- **Do not split by text length.** My first attempt with a splitter by token count turned one book into tens of
  thousands of files, often one paragraph per file. Its index was as long as the book and useless as an entry
  point. Now I export the whole document once as Markdown and split at headings. The goal: sections an LLM reads
  in one go, a few hundred files per book, an index that fits on one screen.
- **Docling puts all headings on one level.** The hierarchy is in the numbering. The script derives the depth
  from it, splits oversized sections finer, attaches small ones to their neighbours and drops index and
  glossary. Docling often does not recognise unnumbered book parts as headings; the chapter list makes up for
  it.
- **Formula enrichment off.** Docling can read formulas as images and turn them into LaTeX. On a machine
  without a graphics card that takes all night for a thick book and is not finished in the morning. Without
  enrichment the same book converts in a coffee break. Formulas from the PDF text layer are often broken anyway.
  Rule in the schema: for formulas the LLM reads the PDF page directly. That is why the PDF stays in the source
  folder.
- **Export once, not per page.** Page-by-page export gets slower with every page and costs more time on a book
  than the conversion itself. Export with a page marker in one pass takes seconds. The page mapping comes from
  the order of the document elements and is exact.
- **All page numbers are PDF pages**, not the printed pagination. Every index says so, so that citations stay
  traceable.

Without text recognition (OCR) and without formula enrichment, conversion is fast: a report in seconds, a
handbook of thousands of pages in minutes.

## The view

One HTML file of a few hundred kilobytes that grows with the wiki. Sidebar by type, search over all page texts
(key `/`), meta line with type, maturity, date and sources, backlinks under every page, formulas via KaTeX,
light and dark design following the system setting. Page switching via `#pagename`, so no server.

Limits I accept: no graph view of the links (backlinks are enough for me), no editing in the browser (the LLM
writes), formulas need internet once for KaTeX.

The file runs by double-click from any folder, including a synced cloud drive. Because it is a single static
file, it can be put on any web host behind a login when you want it on the go. The raw sources do not belong
there, because they may contain copyrighted works. The wiki itself contains only short quotations.

## What I deliberately left out

- No JSON, no second document format, no image export.
- No static-site generator, no Node.
- No vector search. Index plus search in the view are enough up to several hundred pages, then a small search
  script.
- No folder tree in the wiki. Flat, with the type in the front matter. Fewer paths, fewer dead links.
- No daily notes, no project management until I need them. The pattern is open to it.

## Build your own

1. Create a folder, copy this file into it and give it to your agent. Tell it your field, your sources and
   the language the wiki should be in.
2. Have the agent write the schema first, in the file it reads at start: layers and folders, page types and
   front matter, the four workflows with checklists, citation rules, end of session. The schema is the most
   important file and grows with every session.
3. Have it build the three scripts from the description above and test them on a small PDF. Python 3.10 or
   newer in a virtual environment outside cloud drives, with `docling` and `markdown` in it.
4. First PDF into `inbox/`, "new file in inbox". Look at the source index: are the sections a usable size, is
   the chapter list plausible? If not, repeat the split with a different heading level or minimum size. That
   takes seconds from the stored full text, without converting the PDF again.
5. Build the view, open `site/index.html` in the browser. The HTML file always shows the state of the last
   build. So the agent runs the script at the end of every session, otherwise the new pages are missing from
   the view. That belongs in the schema as a fixed step.
6. Ingest the first source, ask the first question, file the first answer as a page. After a few sessions,
   revise the schema. It becomes yours.

## Note

This file describes a pattern, not a standard. The keys, the page types, the section sizes, versioning and
publishing: all of it is adapted to my case and replaceable. The agent should read the file and build, together
with you, the version that fits your sources, your tooling and the way you work.
