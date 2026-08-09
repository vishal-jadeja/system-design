# System Design

Working notes for a system design study track — one design per day, read-first, then implement.

Notes are Obsidian markdown. Diagrams are Mermaid, inline (GitHub renders them). No build step.

## Layout

```
System Design.md    hub / index
concepts/           one note per concept (load balancing, consistent hashing, ...)
designs/            one note per system (001-url-shortener, ...)
templates/          skeletons used for every new note
```

## Method

1. Read the source chapter.
2. Write the design note from `templates/Design Template.md` — requirements, back-of-envelope, API, data model, HLD, deep dives, trade-offs, failure modes.
3. Any concept I can't explain cold gets stubbed in `concepts/` and linked.
4. Implement what's implementable. The note links the repo and commit.

Trade-offs section is the point. A design note without it is a transcription, not a design.

## Reading it outside Obsidian

`[[wikilinks]]` render as plain text on GitHub. Files are named to match, so links are still navigable by hand.
