---
name: li-publish
description: >-
  Publish, schedule or queue a finished LinkedIn post (text, image, video or
  PDF document carousel) to the user's personal profile through the PostZen
  MCP tools, verify it actually went out, and log it. Use when the user says
  "post it", "publish this", "schedule it for Tuesday", "put it in the
  queue", "connect my LinkedIn", or when another skill in this pack has a
  copy-ready block and the user wants it posted rather than pasted.
---

# li-publish

The one skill in this pack that touches LinkedIn. Every other skill writes
and hands off here. This one checks the account, gets the media in, builds the
`createPost` call, gets the user's yes, sends it, and then checks that it
really published, because the success message does not prove that.

PostZen is an approved LinkedIn partner, so this is the official route to a
personal profile, not browser automation. It posts to the member's own
profile only. Company pages are not available yet.

Without PostZen, this skill has nothing to call. Say so, print the copy-ready
block the content skill produced, and the user posts by hand. That path is
not a failure, it is how the pack worked before PostZen existed.

## Step 0: is PostZen connected?

Look for the PostZen MCP tools in this session: `listAccounts`, `createPost`,
`createMediaPresign`. If they are there, PostZen is connected. If they are
not, the user has three ways in:

1. **Plugin install.** The plugin ships `.mcp.json`, which registers the
   `postzen` server at `https://mcp.postzen.dev/mcp`. Run `/mcp`, pick
   `postzen`, then Authenticate.
2. **Manual skills copy.** Add the server first:
   `claude mcp add --transport http postzen https://mcp.postzen.dev/mcp`,
   then `/mcp`, `postzen`, Authenticate.
3. **An API key instead of OAuth.** If the user already has a PostZen key:
   `claude mcp add --transport http postzen https://mcp.postzen.dev/mcp --header "Authorization: Bearer pzn_live_..."`.
   Never ask for the key in chat and never paste it into a file you write.

Authenticate opens the browser at `app.postzen.dev/connect/mcp`. The user signs
in to PostZen there, nothing is typed into Claude. Two choices on that screen:

- **Read & Write or Read Only.** Publishing and scheduling need Read & Write.
- **Profile access.** All profiles, or selected ones.

Approval mints an API key named `MCP: <client name>`, revocable at
`app.postzen.dev/api-keys`. The consent screen expires after 10 minutes, so
if the user wandered off, run `/mcp` again.

**Connecting LinkedIn.** PostZen connects the member's personal profile.
There is no page picker, because company pages are not available yet.

1. `listProfiles` to get a `profileId` (there is usually one, the default).
2. `createConnectUrl({ platform: "linkedin", profileId })`. It returns
   `authUrl` and `state`. The link is good for 10 minutes.
3. The user opens `authUrl` and approves on LinkedIn. The redirect lands on
   PostZen's own callback, which finishes the connection. `completeConnect`
   is for integrations that catch the OAuth code on their own redirect, which
   this flow does not, so skip it.
4. `listAccounts({ platform: "linkedin", profileId })` and check `status` is
   `connected`.

**The free plan**, so the user is not surprised: $0, 2 connected accounts,
20 platform-posts per UTC calendar month, 60 API requests a minute.
Published, scheduled, queued and publishing posts count against the 20.
Drafts, failed and canceled ones do not. Over the account limit,
`createConnectUrl` returns a `402` with `freeTierExceeded`. Over the post
limit, `createPost` returns a `402` with `freePostLimitExceeded`. Paid plans
remove the post cap.

Never ask for or handle the user's LinkedIn password or tokens. The browser
flow is the only path.

## Step 1: find the account

```
listAccounts({ platform: "linkedin" })
```

Use `accounts[i]._id` as `accountId` everywhere. LinkedIn has no handle here:
`username` and `displayName` both hold the member's name. Show it back as
"Jane Doe (LinkedIn)", never as "@jane". If there is more than one connected
LinkedIn account, ask which. Never guess an `accountId` and never type one
from memory.

Check `status`. Anything other than `connected` (`needsReauth`,
`disconnected`, `disabled`) means reconnect first: run `createConnectUrl`
again for the same profile. Reconnecting the same member updates the same
account. A post aimed at an account that is not connected is rejected with
"<name> needs to be reconnected before publishing".

## Step 2: get the media in

A LinkedIn post carries one kind of media, or none:

| kind | media | rules |
| --- | --- | --- |
| text | none | up to 3,000 characters |
| image | 1 to 10 images, in `mediaItems` order | GIFs count as images. 2 or more become a multi-image post |
| video | exactly 1 | MP4 only, up to 500 MB |
| document | exactly 1 PDF | up to 100 MB. This is the carousel format |

Mixing kinds fails: "LinkedIn posts cannot mix images, video, and documents."
Ten images is the API's ceiling, so a bigger set is two posts or a PDF.

**Option A: a public URL.** For images and video only. Any direct `https`
link to the file goes straight into `mediaItems[].url`. PostZen downloads and
re-hosts it. Limit 100 MB. Google Drive, Dropbox, OneDrive and iCloud share
links return a web page, not a file, so they fail. A PDF cannot come in by
URL at all; use Option B.

**Option B: a local file.** Three steps:

```
createMediaPresign({ filename: "playbook.pdf", contentType: "application/pdf", size: 2481733 })
```

returns `uploadUrl`, `publicUrl`, `key`, `type`. Then the bytes go up with a
plain HTTP PUT, which the MCP cannot do, so run it in the shell:

```bash
curl -X PUT --upload-file playbook.pdf -H "Content-Type: application/pdf" "<uploadUrl>"
```

Then `publicUrl` goes into `mediaItems[].url`. Do the PUT right away; the
upload URL is short-lived. If the PUT never ran, `createPost` fails with
"Media upload is not complete for <url>". `size` is the byte count of the
file, which you can read with `stat` or `wc -c`.

Accepted `contentType` values: `image/jpeg`, `image/jpg`, `image/png`,
`image/webp`, `image/gif`, `video/mp4`, `video/mpeg`, `video/quicktime`,
`video/avi`, `video/x-msvideo`, `video/webm`, `video/x-m4v`,
`application/pdf`. That list is wider than LinkedIn's: a `.mov` uploads fine
and is then rejected with "LinkedIn videos must be MP4." Convert first.

**Documents are PDF only.** Word and PowerPoint files cannot get in through
the API. Export to PDF, then upload.

**Format advice.** PostZen hands LinkedIn the bytes as stored and converts
nothing. LinkedIn documents JPG, PNG and GIF for images, so turn a WebP into
a JPEG locally first (ImageMagick, `sips` on macOS, Pillow). Do not tell the
user PostZen will convert it, because it will not.

**Alt text works.** `mediaItems[].title` on an image is sent to LinkedIn as
its alt text. Write one per image.

**Limits PostZen checks:** text length (3,000), the media mix, video format
and the 500 MB cap, document size (100 MB), and up to 10 `mediaItems`. Images
over 8 MB only produce a warning, and LinkedIn may still reject them.

**Limits PostZen does not check, so you do:** video length (LinkedIn wants
3 seconds to 10 minutes) and document pages (LinkedIn caps at 300).

A presigned upload that never ends up in a post is deleted after about 24
hours. Upload, then post, the same day.

## Step 3: build the call

One `createPost` per post. The post text goes in `content`. The LinkedIn
target goes in `platforms[]` with `platform: "linkedin"`, the `accountId`
from Step 1, and `settings`.

The `settings` keys that are safe to send:

| key | values | rules |
| --- | --- | --- |
| `visibility` | exactly `"PUBLIC"` or `"CONNECTIONS"` | default `PUBLIC`. Any other value is quietly treated as `PUBLIC`, so spell it exactly and echo it back to the user |
| `documentTitle` | up to 200 characters | **always set it on a document post.** Without it LinkedIn shows the upload's filename, extension and all ("Q3 Report.pdf") |
| `videoTitle` | up to 200 characters | video posts only |
| `reshareUrl` | a public post link (the kind with `activity-<digits>` in it) or a `urn:li:activity:`, `urn:li:share:` or `urn:li:ugcPost:` URN | reshares someone's post with the user's text on top. No media allowed with it |
| `disableLinkPreview` | `true` | keeps a link in the text as plain text instead of a link card |

**Never send these:**

- `organizationUrn`, or any alias of it (`organizationId`, `organization_id`,
  `organization_urn`). Company-page posting is not available yet, and sending
  it can break the connection: LinkedIn refuses, and PostZen marks the
  account as needing a reconnect.
- `firstComment`. It is not available yet. It is accepted, then fails
  silently after publish, so the post goes out with no comment and nobody is
  told. One over 1,250 characters rejects the whole post. If the post relies
  on a link in the first comment (`/li-post` puts links there), the user adds
  that comment by hand once the post is live. Say so in the gate.
- `geoRestrictionCountries`. It only works for company pages and is rejected
  on a personal post.

**What the text can and cannot do.**

- No mentions. `@Name` goes out as plain text, it does not tag anyone. A post
  that has to tag people is a manual post.
- Hashtags go out as typed text. Whether LinkedIn turns them into clickable
  hashtags on this path is not confirmed, so check the first one in the feed.
- Line breaks are kept.
- Link cards: with no media, no reshare and no `disableLinkPreview`, PostZen
  attaches a card for the first link in the text. Its title is the site's
  hostname. There is no custom card title or image.

An example, a scheduled document post:

```json
{
  "content": "We cut proposal time from 5 hours to 20 minutes.\n\nThe whole process is in the PDF. Slide 9 is the one people screenshot.",
  "mediaItems": [{ "url": "https://media.postzen.dev/.../playbook.pdf" }],
  "platforms": [{
    "platform": "linkedin",
    "accountId": "<_id from listAccounts>",
    "settings": {
      "visibility": "PUBLIC",
      "documentTitle": "The 20-minute proposal playbook"
    }
  }],
  "scheduledFor": "2026-10-13T08:15:00-04:00",
  "x-request-id": "li-doc-2026-10-13-proposals"
}
```

`x-request-id` is an idempotency key. If a call times out and you cannot
tell whether it landed, repeat it with the same value: PostZen returns the
original post under `existingPost` instead of creating a second one. It does
not re-run a publish that failed.

## Step 4: pick exactly one mode

| mode | set | notes |
| --- | --- | --- |
| **draft** | `isDraft: true` | saved in PostZen, nothing goes to LinkedIn. `platforms` is optional. No confirmation needed |
| **now** | `publishNow: true` | publishes inside the request |
| **schedule** | `scheduledFor: "<ISO 8601>"` | at least 60 seconds in the future. An offset (`-04:00`) or `Z` both work. Always write the offset for the user's timezone rather than converting to UTC in your head |
| **queue** | `queuedFromProfile: "<profileId>"`, optional `queueId` | PostZen claims the profile's next free slot and returns it as `scheduledFor`. The profile needs a queue with slots; `/li-plan` sets those up |

Two modes in one call is a 400. No mode at all is an error too. Never call
`getNextQueueSlot` or `previewQueue` and feed the answer back as
`scheduledFor`; they are previews and reserve nothing. Use queue mode.

**When to post.** PostZen has no LinkedIn engagement data yet (analytics are
waiting on LinkedIn's approval), so `getBestTimeToPost` will most likely come
back with no slots for LinkedIn. Do not present an empty answer as data. Use
the time from `/li-plan`, or the user's own call, and label any general
guidance as general.

Every time shown to the user carries a timezone. Silent UTC is how a post
goes out at 3 AM.

## Step 5: the gate

Before `createPost` with `publishNow` or `scheduledFor` or
`queuedFromProfile`, show all of this and get an explicit yes in this
conversation:

```
PUBLISH  ·  Jane Doe (LinkedIn, personal profile)

type:        document post
media:       playbook.pdf, 2.4 MB, 11 pages, title "The 20-minute proposal playbook"
text:        "We cut proposal time from 5 hours to 20 minutes." (126 chars)
visibility:  PUBLIC
when:        Tue Oct 13, 8:15 AM EDT (2026-10-13T08:15:00-04:00)
by hand:     nothing

Reply "yes" to schedule it, or tell me what to change.
```

`text` shows the first line and the full character count; show the whole
post above the block if the user has not seen this exact version. `by hand`
lists anything the user still has to do in LinkedIn, such as a first comment
with the link.

"Looks good" about the draft earlier in the conversation is not a yes to
this. The yes is to this block, with this account, this visibility and this
time. Drafts (`isDraft: true`) do not need it.

If the user wants to change anything, change it and show the block again.
Do not send a version they have not seen.

## Step 6: verify, because the message lies by omission

`createPost` returns `{ post, message }`. The `message` is chosen by mode,
not by outcome: "Post published successfully" comes back even when LinkedIn
refused the post, because the provider error is caught and logged and the
request still returns 201. So:

1. Read `post.platforms[].status`. It is one of `draft`, `scheduled`,
   `pending`, `publishing`, `published`, `failed`, `canceled`.
2. If `failed`, read `post.platforms[].error` and tell the user the actual
   text. Do not retry blindly. If the error says the outcome is unknown
   (`provider_outcome_unknown`), the post may already be on LinkedIn: ask
   the user to check their feed before anything is sent again. Otherwise fix
   the cause, show the gate again, and send a new `createPost` with a new
   `x-request-id`.
3. If `publishing` or `pending`, keep checking with
   `getPost({ postId: post._id })` about every 30 seconds. Video is the
   usual reason: LinkedIn processes it, and PostZen gives up after 20
   minutes with `container_timeout`. Poll, do not resend. Tell the user
   what you are waiting on rather than going quiet.
4. If `published`, take `platformPostUrl`
   (`https://www.linkedin.com/feed/update/<urn>/`) and give it to the user.
   `platformPostId` is the LinkedIn URN.
5. If `scheduled`, repeat the time back with its timezone and stop. The post
   goes out later; nothing to verify yet.

If the error points at the connection (an auth failure), the account will
show `needsReauth`. Reconnect (Step 0), then send again after a fresh yes.

A `402` with `freePostLimitExceeded` means the month's 20 are used. Say so,
save the post as a draft if the user wants, and leave the upgrade decision
to them.

## Step 7: log it

On `published` or `scheduled`, append one line to
`~/.claude/linkedin/log.md`. `/li-audit` reads it to match hook formulas to
results and `/li-plan` reads it to avoid repeating a theme:

```
2026-10-13  DOCUMENT  #17 Time Anchor  "We cut proposal time from 5 hours to 20 minutes."  postzen:<post._id>  https://www.linkedin.com/feed/update/urn:li:share:XXXX/
```

Date, format (TEXT, IMAGE, VIDEO, DOCUMENT, RESHARE), hook formula if the
content skill named one, the first line, the PostZen post id, and the URL if
there is one yet. Create the file if it does not exist. If `/li-post`
already logged this post when the draft was approved, add
`postzen:<post._id>` and the URL to that line rather than writing a second
one; one post, one line.

## Afterwards

- **Change a scheduled post:** `updatePost({ postId, ... })`. Omitted fields
  keep their values. Passing `platforms` replaces every target and all its
  settings, so resend `visibility`, `documentTitle` and the rest in full.
  Works on `draft`, `scheduled`, `queued`, `failed`, `partially_failed` and
  `canceled`. Published and publishing posts cannot be edited.
- **Cancel a scheduled post:** `deletePost({ postId })`. Same statuses.
  Nothing PostZen does edits or removes a post that is already on LinkedIn.
  That is done on LinkedIn.
- **See what is scheduled:** `listPosts({ platform: "linkedin", accountId,
  status: "scheduled", sortBy: "scheduledFor" })`. Queued posts show up here
  too. It covers posts created in PostZen only.

## What this skill cannot do

Say so when asked rather than attempting a workaround.

**Not available yet, waiting on LinkedIn's approval:**

- Analytics of any kind: impressions, reactions, follower counts, best
  times. `/li-audit` stays paste-based.
- Reading or replying to comments and reactions. `/li-reply` stays
  paste-based.
- The inbox and DMs, invites included.
- First comments.
- Posting to a company page.

**Not offered by this route:**

- Mentions or tagging people.
- Polls, articles, newsletters and events.
- Word or PowerPoint documents (export to PDF), or a document by public URL.
- More than 10 images in one post.
- Custom link-card titles or images.
- Editing or deleting a post after it is published.
- Checking video length or PDF page count before LinkedIn does.

## Cautions

- Publishing is a real, outward-facing action. Never call `createPost` with
  `publishNow: true`, a `scheduledFor`, or `queuedFromProfile` without the
  user's explicit yes to the exact content, account, visibility and time in
  this conversation.
- Never invent `accountId` or `profileId` values. They come from
  `listAccounts` and `listProfiles` in this session.
- Never send `organizationUrn` or `firstComment`, even if the user asks for
  a company-page post or a first comment. Explain that they are not
  available yet.
- If a `createPost` call fails, report the actual error. A blind retry can
  double-post if the first attempt landed.
- "Post published successfully" is a mode label. `platforms[].status` is
  the truth.
- One post, one call. For a week of posts, `/li-repurpose` collects one
  confirmation that lists every post, then this skill sends them one at a
  time and verifies each.
