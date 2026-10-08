---
name: li-plan
description: >-
  Build the week on LinkedIn - what to post, when to post it, and who to engage
  with. Use when the user says "plan my week", "what should I post", "content
  calendar", "I have nothing to post about", or wants a posting schedule and an
  engagement list.
---

# li-plan

The control room. Everything else in this pack executes; this decides what
gets executed. Run it once a week, on the same day.

## Input

If `~/.claude/linkedin/voice.md` and `log.md` exist, read them - the plan
should not repeat a theme from the last fortnight. If they do not exist, ask
for four things and write them down:

1. What the user sells, and to whom.
2. The three or four themes they want to be known for.
3. What actually happened this week - a client call, a number, a mistake, a
   thing they built, an argument they had. This is where posts come from.
4. Ten to twenty people or companies worth being visible to.

## What to post

Four posts a week beats seven. Consistency is a floor, not a target, and the
fifth post of a week is almost always the weak one that drags the average.

Mix across the week, never two of the same type back to back:

| type | share | job |
| --- | --- | --- |
| **Proof** | 1 per week | something that happened, with a number |
| **Opinion** | 1 per week | a position that could lose you followers |
| **Teach** | 1 per week | one thing the reader can do today |
| **Story** | 1 per fortnight | a scene with dialogue and a cost |
| **Offer** | 1 per fortnight | what you sell, said plainly, no apology |

For each slot give: the theme, the specific angle drawn from what actually
happened this week, and the hook formula number from `li-post/hooks.json` that
fits it. Not a topic - an angle. "AI" is not a plan. "The proposal we lost
because our AI draft had an em dash in it" is a post.

## When to post

Post when the user's audience is at a desk. For a B2B audience in one
timezone, Tuesday to Thursday, 7:30-9:30am local, is the working default, with
Monday afternoon and Friday morning as the second tier. Weekends are for
personal-story posts or nothing.

But state this plainly: **the day and hour matter far less than whether the
first line is good.** If the user is optimising posting times before their
hooks work, they are polishing the wrong thing, and you should say so.

Anchor the times to their audience's timezone, not the user's, if those
differ.

### With PostZen connected

If the PostZen MCP tools are in this session, two things improve and one
does not.

**What is already booked.** `listPosts({ platform: "linkedin", accountId,
status: "scheduled", sortBy: "scheduledFor" })` shows every scheduled post,
queued ones included, so the plan does not double up a day. Only posts
created in PostZen appear here.

**Queue slots.** If the user wants a standing schedule rather than a time per
post, PostZen's queue does that: posts go in with `queuedFromProfile` and
take the next free slot.

- `listProfiles` for the `profileId`, then `listQueueSlots({ profileId, all:
  true })` to see what exists, including the next five free instants.
- `createQueueSlot({ profileId, timezone: "America/New_York", slots: [{
  dayOfWeek: 2, time: "08:15" }, { dayOfWeek: 4, time: "08:00" }, {
  dayOfWeek: 5, time: "08:30" }], name: "LinkedIn" })` creates a whole queue,
  not one slot. `dayOfWeek` is 0 for Sunday, `timezone` is an IANA name. The
  first queue on a profile becomes its default. Show the slots with the
  timezone and get a yes before creating.
- `updateQueueSlot({ profileId, timezone, slots, queueId })` **replaces every
  slot** in that queue; it is not a nudge to one slot. Send the full list
  each time. Posts already placed keep their times unless
  `reshuffleExisting: true`.
- `deleteQueueSlot` **without a `queueId` deletes every queue on the
  profile.** Never call it without one, and confirm with the user first.
- `previewQueue({ profileId })` returns the upcoming free instants, not the
  posts in them, and reserves nothing. For posts, use `listPosts` as above.

**What does not change: the timing.** PostZen has no LinkedIn engagement data
yet (analytics are waiting on LinkedIn's approval), so it cannot tell you when
this account's audience responds. `getBestTimeToPost` will most likely come
back empty for LinkedIn. The times in the plan are still the general default
above, and say so. They are not personalised.

## Who to engage with

Build a list of 10, split three ways:

- **5 reach** - people with an audience the user wants, whose posts they can
  genuinely add to. Comment before they have 20 comments or nobody sees it.
- **3 peers** - same level, same field. This is the group that reciprocates.
- **2 buyers** - people who could actually buy. Comment on their posts for
  weeks before any DM, and never pitch in a comment.

20 minutes a day, before posting, not after. Comments on other people's posts
are what makes the user's own post land.

## Output

```
WEEK OF SEP 8

MON  engage only  (20 min, list below)
TUE  8:15am  PROOF    #17 Time Anchor   - the 5hr -> 20min proposal rebuild
WED  engage only
THU  8:00am  OPINION  #1  Contrarian    - why we killed the discovery call
FRI  8:30am  TEACH    #21 Direct Value  - the 4-line reopen email, given away
SAT  -
SUN  4:00pm  STORY    #9  Cold Open     - the "going with someone cheaper" email

ENGAGE  (5 reach / 3 peers / 2 buyers)
  ...

Say "write Tuesday" and I will draft it.
```

Write the plan to `~/.claude/linkedin/plan.md` so the other skills can read it.
Writing a time into the plan schedules nothing. The user runs the plan, or
`/li-publish` schedules each post once it is written and approved.
