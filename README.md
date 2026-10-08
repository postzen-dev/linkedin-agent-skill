# The LinkedIn agent skill

Twelve Claude skills for running a LinkedIn account. Connect [PostZen](https://www.postzen.dev) to publish and schedule posts on your personal profile through LinkedIn's official API.

The skills write posts from 21 hook formulas, draft comments for other people's posts and replies for the thread under yours, score your profile out of 100, and plan the week. `/li-publish` sends a finished post to LinkedIn and checks that it went out.

The humanizer is what makes the rest usable. It strips em dashes, stock phrases and invisible watermark characters from a draft, then scores what is left on five checks before you see it.

**Nothing gets posted until you say yes.** Each skill shows you the exact text, account, visibility and time first.

## Install

Install the plugin in Claude Code:

```text
/plugin marketplace add postzen-dev/linkedin-agent-skill
/plugin install postzen-linkedin
```

The plugin registers the PostZen MCP server. Run `/mcp`, select `postzen`, and choose Authenticate. Sign in on the PostZen page that opens in your browser. Select **Read & Write** to enable publishing.

Or copy the skills by hand:

```bash
git clone https://github.com/postzen-dev/linkedin-agent-skill.git
cp -r linkedin-agent-skill/skills/li-* ~/.claude/skills/
claude mcp add --transport http postzen https://mcp.postzen.dev/mcp
```

Then run `/mcp`, select `postzen`, and authenticate.

For a project-local install, copy the same folders into your repo's `.claude/skills/`.

You can also paste a single `SKILL.md` at the top of a chat to use it as a mode. The writing works without Claude Code, but the two Python tools and PostZen are unavailable, and the Python tools are most of the point of `/li-human`.

**Connect LinkedIn:** Say "connect my LinkedIn". `/li-publish` asks PostZen for a connect link. Open it and approve on LinkedIn. It connects your personal profile. PostZen's free plan covers 2 connected accounts and 20 posts a month.

PostZen is optional. Every writing skill works without it, and you paste the result yourself.

Spend ten minutes on `templates/voice.md`. Copy it to `~/.claude/linkedin/voice.md` and fill it in, or paste three of your own posts into Claude and say "write my voice.md from these". Every skill reads that file. Skip it and everything comes out sounding like everyone else.

## The twelve skills

| Command | What it does | With PostZen connected |
| --- | --- | --- |
| `/li-post` | One idea into a post. Three hook options from [21 formulas](skills/li-post/hooks.json), one full draft, humanized before you see it. | Hands the post to `/li-publish`. |
| `/li-comment` | Comments on other people's posts. Nine types, picked by what the post actually is. Never "Great post!". | Unchanged. You post comments by hand. |
| `/li-reply` | The thread under your own post. Sorts every comment into lead / substance / peer / support / noise, then writes in that order. | Unchanged for now. You paste the comments in. |
| `/li-profile` | Scores your profile against a [12-part rubric](skills/li-profile/rubric.json) out of 100, then rewrites in fix-first order. | Unchanged. You paste each section in. |
| `/li-plan` | The week: what to post, when, and the 10 people to engage with. Writes `~/.claude/linkedin/plan.md`. | Shows what is already scheduled and sets up queue slots. |
| `/li-human` | The humanizer. Two scripts that actually run. See below. | Unchanged. |
| `/li-carousel` | Document posts. Slide-by-slide copy, the cover that earns the swipe, and the PDF. | Publishes the PDF as a document post. |
| `/li-repurpose` | One video, newsletter or transcript into a week of posts that each stand alone. | Schedules the week after one confirmation. |
| `/li-dm` | The 200-character invite note, the first message, and the two follow-ups. Two. | Unchanged. You send them by hand. |
| `/li-inbox` | Triages the inbox into lead / recruiter / peer / ask / spam, and names the tell that gave a sequence away. | Unchanged for now. You paste the messages in. |
| `/li-audit` | Post-mortem on what you have published. Ranks by engagement rate and reach multiple, not impressions. | Unchanged for now. You paste the analytics in. |
| `/li-publish` | Handles LinkedIn writes. Takes the media, prepares `createPost`, asks for approval, then verifies and logs the post. | Provides the PostZen publishing workflow. |

## The humanizer

`/li-human` ships two Python scripts with no dependencies. They run on your machine, on your text, and nothing is uploaded.

```bash
python3 humanize.py draft.txt --report      # clean it, show every change
python3 detect.py draft.txt                  # score it, five checks
python3 detect.py before.txt after.txt       # prove the delta
```

The cleaner handles:

- **Invisible characters:** zero-width spaces and joiners, word joiners, soft hyphens, byte-order marks, Unicode tag characters, non-breaking and narrow spaces. Your keyboard doesn't type these. They survive copy-paste and don't show in any editor.
- **Typography:** em dashes to commas, en dashes to hyphens, curly quotes to straight quotes, ellipses to three dots.
- **Stock wording:** 113 words and phrases swapped for plain English ("delve", "leverage", "robust", "testament to", "in today's fast-paced world", "let that sink in"), with capitalisation kept and URLs left alone. Edit the list in [`slop.json`](skills/li-human/slop.json).

The detector flags sentence shapes that need a rewrite: "It's not just X, it's Y", rule-of-three triads, one-word rhetorical questions, hashtag walls, reflex engagement bait, and uniform sentence length. It flags them rather than fixing them, because changing the shape of a sentence takes judgement a regex doesn't have.

The five checks, scored 0 to 100, higher is more human:

| Check | What it measures |
| --- | --- |
| BURSTINESS | sentence-length variation. Models write even. |
| SPECIFICITY | numbers, names and concrete markers per 100 words |
| SLOP DENSITY | lexicon hits per 100 words |
| FINGERPRINT | invisible characters, em dashes, curly quotes per 1,000 |
| VOICE | contractions, person, structural tells |

The verdict weights the mean at 60% and the **weakest single check** at 40%, because a detector only needs one signal to fire.

A deliberately terrible draft scored:

```text
  BURSTINESS    ##################......  73.0
  SPECIFICITY   ######################## 100.0
  SLOP DENSITY  ........................   0.0    19 stock terms, 24.1 per 100 words
  FINGERPRINT   ........................   0.0    1 invisible, 1 em dash, 3 curly quote
  VOICE         ########................  33.3    3 structural tells
  ------------------------------------------------------------
  HUMAN SCORE   ######..................  24.8   FLAGGED
```

After `humanize.py`, with the flagged structures still unrewritten:

```text
  HUMAN SCORE   #################.......  69.7   REVIEW    (+44.9)
```

The last stretch to PASS is the rewrite the script leaves to you.

## PostZen integration

PostZen is a social media API with a hosted MCP server and an approved LinkedIn partner app. It holds the tokens and the LinkedIn approval, so Claude can post to your personal profile through the official API without you building any of that.

With PostZen, the skills can:

- **Publish, schedule, queue or draft** text posts, posts with up to 10 images (alt text included), a single MP4 video, and PDF document posts (carousels). `/li-publish` reads back the post's status to confirm it went out, because a "published successfully" message describes the requested mode, not the result.
- **Set visibility** to anyone or connections only, title a document or video, reshare a post with your text on top, or turn off the link card.
- **Run a queue:** posts take the next free slot in a weekly schedule you set once.

Not available yet, waiting on LinkedIn's approval:

- Analytics of any kind, so `/li-audit` still runs on pasted numbers and `/li-plan`'s posting times are general, not personalised.
- Reading or replying to comments and reactions.
- The inbox, DMs and invites.
- First comments. If a post's link belongs in the first comment, you add it by hand.
- Posting to a company page. Everything goes to your personal profile.

Not offered by this route: mentions (an `@Name` stays plain text), polls, articles and newsletters, Word or PowerPoint documents (export to PDF), and editing or deleting a post after it is published.

## The fine print

**The five checks are local heuristics, not detector APIs.** They are modelled on the signals public detectors look for, and they run entirely on your machine. They are not GPTZero, Originality, Copyleaks, Winston or Turnitin, they don't call those services, and they can't promise their verdicts. Fixing what they measure tends to move those numbers, because both are measuring the same underlying things. That is the whole claim. Nobody can honestly sell "undetectable".

**The invisible-character pass is real and narrow.** It removes the zero-width and format characters that end up in generated text and survive a copy-paste. That is a genuine, checkable fingerprint. It isn't a claim about defeating cryptographic watermarking, and this repo doesn't make one.

**Nothing here fabricates.** No invented metrics, clients or outcomes go out under your name. If a draft needs a number you haven't given, it comes back with `{{your number}}` in it and a flag, every time.

**No browser automation.** Posting goes through PostZen's approved partner app or through your own hands. Nothing here drives the LinkedIn site with a browser or scrapes it, which [LinkedIn's User Agreement](https://www.linkedin.com/legal/user-agreement) prohibits.

## Files

```text
.mcp.json                        registers the PostZen MCP server for the plugin
skills/li-publish/SKILL.md       the only skill that touches LinkedIn
skills/li-post/hooks.json        21 hook formulas: template, example, what it is for, how it gets ruined
skills/li-human/slop.json        the lexicon: 113 terms, 17 invisible classes, 11 structural tells
skills/li-human/humanize.py      the three cleaning passes
skills/li-human/detect.py        the five-check panel
skills/li-profile/rubric.json    the 100-point profile score
templates/voice.md               your voice profile. Fill this in first.
```

## Credits

This pack is a fork of Jake Schincariol's [linkedin-agent-skill](https://github.com/Jakeschincariol/linkedin-agent-skill), licensed under MIT. He wrote the original eleven skills, the hook formulas, the humanizer and the profile rubric.

PostZen added `/li-publish` and the publishing layer across the other skills.

## License

MIT.
