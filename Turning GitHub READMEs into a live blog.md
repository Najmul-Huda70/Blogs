# Turning GitHub READMEs into a live blog — without a database

While building my portfolio, I kept running into the same annoying loop: I'd
write a project's `README.md` on GitHub, then copy-paste a version of the
same write-up into MongoDB so it could show up on my website. Two sources of
truth for the same content — and every time I updated one, I'd forget to
update the other.

The fix ended up being the most interesting part of the whole build: **stop
storing content in the database at all. Store the GitHub URL, and fetch the
markdown live.**

## The idea

Instead of a `content` field in my `Work` or `BlogPost` document, I just
store a `sourceUrl` — a normal `github.com/.../blob/...` link to a README.
When the portfolio renders that project or post, it:

1. Converts the GitHub `blob` URL into a raw content URL
2. Fetches the raw markdown at request time
3. Optionally pulls out just the section it needs (like an "Overview" or
   "Features" heading)
4. Renders it as HTML on the page

That's it. Update the README on GitHub, and the portfolio reflects it on
the next request — no redeploy, no database write, no second copy of the
content to maintain.

## The code

The core of it is two small functions.

**Converting a GitHub blob URL to a raw URL**, since `github.com` pages are
HTML wrappers around the actual file — you need the `raw.githubusercontent.com`
version to fetch plain markdown:

```typescript
export function toRawGithubUrl(url: string): string {
  const u = new URL(url);
  if (u.hostname === "raw.githubusercontent.com") return url;

  if (u.hostname === "github.com") {
    const parts = u.pathname.split("/").filter(Boolean);
    const [user, repo, blob, branch, ...fileParts] = parts;
    if (blob === "blob") {
      return `https://raw.githubusercontent.com/${user}/${repo}/${branch}/${fileParts.join("/")}`;
    }
  }
  throw new Error("Unrecognized GitHub URL format");
}
```

**Extracting just one section of a README**, so a project card can show
only the "Overview" heading instead of the entire file:

```typescript
export function extractSection(markdown: string, heading: string): string {
  const lines = markdown.split("\n");
  const startIdx = lines.findIndex((l) => l.trim() === heading);
  if (startIdx === -1) return markdown;

  const level = heading.match(/^#+/)?.[0].length ?? 2;
  let endIdx = lines.length;
  for (let i = startIdx + 1; i < lines.length; i++) {
    const m = lines[i].match(/^#+/);
    if (m && m[0].length <= level) {
      endIdx = i;
      break;
    }
  }
  return lines.slice(startIdx, endIdx).join("\n");
}
```

The heading-level check is the part I like most — it stops at the next
heading of the *same or higher* level, so a `## Overview` section correctly
includes any `### Subsection` underneath it, but stops as soon as another
`##` heading starts.

## Why this felt worth it

- **One source of truth.** The README is already the natural place to
  document a project. Now it's the *only* place.
- **Git history for free.** Every edit to my project write-ups is
  version-controlled, without me building any of that myself.
- **Zero-friction updates.** Fix a typo in a README on my phone through
  GitHub's web editor, and it's live on the portfolio on the next page load.
- **The database stays tiny.** `Work` and `BlogPost` documents only hold
  metadata (title, tags, category, `sourceUrl`) — the heavy content lives
  where it's actually authored.

## Trade-offs I'm accepting

Nothing is free, and I want to be honest about the parts of this that need
care:

- **Every page render is a live fetch** to `raw.githubusercontent.com`
  unless I add caching (Next.js `fetch` with `revalidate` handles this
  well for me).
- **GitHub's rate limits** apply to unauthenticated raw fetches — fine for
  a personal portfolio's traffic, but something to watch if traffic grows.
- **No offline fallback** if GitHub is down, though for a low-traffic
  personal site that risk is acceptable.
- **Markdown → HTML still needs a renderer** on the frontend (I'm using a
  markdown-to-HTML library rather than writing my own parser).

## What's next

The next step is caching fetched markdown at build/revalidate time so
project pages don't hit GitHub on every single visitor, and adding a
fallback that shows the last successfully fetched version if a request
fails. But even in its current, simple form, this pattern removed an
entire category of "forgot to update the database" bugs from my workflow —
and that's exactly the kind of problem worth solving once, structurally,
instead of remembering to do it manually every time.

[more details](https://markdown-tools.com/react-markdown)

[more details](https://hannadrehman.com/blog/enhancing-your-react-markdown-experience-with-syntax-highlighting)
