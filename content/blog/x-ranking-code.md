---
title: "I read the code that decides who sees my posts"
date: 2026-11-07T23:30:00+05:30
slug: ""
description: "X open-sourced its For You ranker. A weighted sum, a few multipliers, a slot reserved for small accounts, and a Grok model grading replies under big ones. Also: one report does not cancel 468 likes."
keywords: ["recommender systems", "ranking", "x algorithm", "phoenix", "thompson sampling", "grok", "feed ranking"]
draft: true
tags: ["ml", "code-reading"]
math: false
toc: true
---

Some of my posts do fine. Others vanish without a trace. For a while my working theory was "the algorithm," which is a bit like blaming the weather. It explains everything and predicts nothing.

Then X published the code for its For You feed at [`xai-org/x-algorithm`][repo]. So I did the obvious thing and read it instead of guessing. This is what it actually rewards, in roughly the order I had questions about it, with a few places where I read it wrong first.

> **Scope:** everything here is from the repo at commit [`3b6919f`][commit] (7 Nov 2026). The numbers are the defaults in the code. The README says a cron job syncs those defaults to the main production values, but experiments override them for some users [1]. So read every number as "what most people probably get today", not as a law of physics. This is also not a growth guide. I'm interested in how the machine works.

## Where do the posts even come from?

A feed is built per request, in two halves. **Candidate sources** fetch a pool of posts that might be worth showing. Then **ranking** decides the order [2].

The sources are:

- **Thunder**, an in-memory store of recent posts from accounts you follow.
- **Phoenix retrieval**, a two-tower model. It embeds you and every post as vectors and returns the posts nearest to you.
- **SimClusters**, which groups accounts by who engages with what.
- A shared list of **popular posts**.

Before anything gets scored, a pile of filters runs. Two of them surprised me.

The first is that [anything older than 48 hours][age] is gone: `MAX_POST_AGE: u64 = 48 * 60 * 60`. For You has a two-day memory.

The second is that [replies and reposts from accounts you don't follow are dropped][oonreply] from For You entirely. The flag that would let some of them through is [`false` by default][oonparam]. So a reply only reaches strangers inside the conversation it was posted in. I'll come back to this, because X has a whole separate machine for those.

## How does one post get a score?

My mental model was a single "quality" number. That's not it. The ranker, also called Phoenix, is a transformer that reads your recent engagement history and predicts a separate **probability for each action** you might take on each candidate: like, reply, repost, share, block, report, and so on [3]. One detail I liked is that candidates [can't attend to each other][isolation]. A post's score doesn't depend on what else is in the batch, so scores can be cached.

> **Note:** I went in believing this was "the Grok-based ranker", because that's how people describe it. The [Phoenix README][phoenixreadme] is more careful. An *earlier* release shipped a sample transformer ported from Grok-1. This release ships the production model code, and nowhere does it say that model is Grok [4]. There is a Grok in this story, though. It just has a different job. Keep reading.

The probabilities are then collapsed into one number with a [plain weighted sum][readmescore]:

```text
Final Score = Σ (weight_i × P(action_i))
```

And [the actual multiply][apply] is about as boring as code gets:

```rust
fn apply(score: Option<f64>, weight: f64) -> f64 {
    score.unwrap_or(0.0) * weight
}
```

All the interesting decisions live in the weights. Here are [the defaults][weights]:

| Action | Weight | Action | Weight |
| --- | --- | --- | --- |
| share via copy link | **20.0** | like | 0.708 |
| reply | 5.0 | click into the post | 0.274 |
| quote | 5.0 | open a link | 0.2 |
| share via DM | 5.0 | profile click | **0.0** |
| follow the author | 4.0 | **report** | **−234** |
| share (generic) | 2.118 | **mute author** | **−58.8** |
| repost | 1.0 | **not interested** | **−53.07** |
| | | **block author** | **−31.2** |

Read as a statement of values, this is pretty clear. The most valuable thing you can do to a post is copy its link to send to someone. That's worth about 28 likes' worth of weight. A profile click is worth nothing *directly*, and it only matters because it can lead to a follow, which is worth 4. And the negative side is large.

Which brings me to the number everyone quotes.

## Does one report really cancel 468 likes?

No. The repo goes out of its way to say so, in the [README][myth] and in a comment sitting [right above the weights][mythcomment]:

```rust
// One common misinterpretation is that you can read these weight
// ratios as count equivalences, e.g. the incorrect statement that
// "one report cancels 468 likes" -- this is incorrect because the
// weights apply to the predicted probabilities rather than raw counts.

// And the baseline probability of a Report is more than 1000x lower
// than a Like, so it’s weighted more to allow the prediction to affect
// the final ranking at all.
```

The weights multiply *your predicted probability* of reporting this post, not a count of reports anyone has filed. Here's a toy viewer, using the comment's own "1000x" ratio: P(like) = 0.05, P(report) = 0.00005. The like term is 0.708 × 0.05 ≈ 0.035. The report term is −234 × 0.00005 ≈ −0.012. A big weight on a tiny probability is a modest number. The weight exists so that a rare event can register at all.

Two more things from the same comment. Predictions are personal, so reports from a group of bad actors mostly change what *people like them* see. And engagement only counts if the post was served in your Home timeline. Showing up from a group-chat link "has no ranking impact".

My favourite detail is where 468 came from. It's exactly 234 / 0.5, and 0.5 was the like weight in [the July diff][oldlike]. The default today is 0.708. So the myth is wrong, and it's also out of date. Even taken literally, it would now say one report cancels about 330 likes.

## What happens after the sum?

Two multipliers, applied in a separate service ([`vm-ranker`][vmrequest]) that home-mixer calls with `compute_value_model: true`.

**The out-of-network discount.** A post from someone you don't follow is multiplied by [0.75][oonfactor]. So, it turns out, is a reply or repost from someone you *do* follow ([`oon_applies`][oonapplies], with the flag [on by default][oonrescore]).

**Author diversity.** The params are [decay 0.5, floor 0.25][divparams]. On my first pass I read that as "your second post gets ×0.5 and everything after gets ×0.25". Clean, memorable, and wrong. Here is [the function][divfn]:

```rust
pub fn diversity_multiplier(decay_factor: f64, floor: f64, exponent: f64) -> f64 {
    (1.0 - floor) * decay_factor.powf(exponent) + floor
}
```

The decay applies to the part *above* the floor. So the multipliers for an author's 1st, 2nd, 3rd and 4th posts within one ranking request are:

| Post by same author | k | Multiplier |
| --- | --- | --- |
| 1st | 0 | 1.0 |
| 2nd | 1 | 0.625 |
| 3rd | 2 | 0.4375 |
| 4th | 3 | 0.34375 |

The curve approaches 0.25 but never reaches it. Also, [`author_pool_counts`][divcount] sorts by score first, so your *best* post is the one that gets 1.0. The function was fine. I was the bug.

After that, a [determinantal point process][readmecomponents] reorders the list, giving up a little score in exchange for neighbours that are less alike. A feed with ten near-identical posts in a row is a worse feed, even if each one scores well on its own.

## Is there any hope for small accounts?

This was the part I found most interesting. There's a dedicated **cold-start slot**. A post is [eligible][coldeligible] if it:

- is an original post, not a reply or a repost
- comes from an author with ≤ 50,000 followers
- is under 2 hours old
- has under 200 Home impressions
- isn't already in the bottom 3% of the ranked list

[The params][coldparams] say all of this, and the [filter][coldfilter] enforces it.

At most one such post per request gets lifted. Its score is raised to at least whatever the post at [rank index 15][coldtarget] scored (`ColdStartSlotMin = 15`, `ColdStartSlotMax = 16`, and the range is exclusive), so it lands around the 16th slot of your feed.

Picking *which* small post gets the slot is the fun part. It's a multi-armed bandit, solved with [Thompson sampling][coldts]:

```rust
let n = scale * candidate.view_count_on_home.unwrap_or(0) as f64;
let x = (candidate.fav_count.unwrap_or(0).max(0) as f64).min(n);
let alpha = alpha0 + x;
let beta = beta0 + (n - x).max(0.0);
Beta::new(alpha, beta).map(|d| d.sample(rng)).unwrap_or(0.5)
```

Each eligible post gets a Beta distribution over its like rate. The prior is [α₀ = 0.75, β₀ = 49.25][coldbeta], which is a 1.5% like rate weighted like 50 imaginary impressions. Each real impression and like updates it. The code draws one sample per post, keeps the [top 2][coldtopk], and gives the slot to whichever of those two the main model scores higher.

That design is quietly lovely. A brand-new post with zero data has a wide distribution, so it sometimes draws high and gets a chance. A post that has had 150 impressions and no likes has a narrow distribution sitting near zero, and it stops getting picked. Exploration is spent where the uncertainty is. So for a small account, the first ~200 impressions in the first two hours are effectively an audition, and the like *rate* is the score.

## What changed when X said it was boosting mutuals?

One line. In [`reply_weight_for`][replyweightfor], if the author is someone you mutually follow and the post is [an original, not a reply or a repost][bidieligible], the reply weight becomes `5.0 + 15.0 = 20.0`. Same model, same probability, a different price on replying to people you actually know.

The repo includes [a whole doc][bididoc] walking through how this shipped [5]. An A/B test on July 10 tried boosts of 5, 10, 15 and 20. On July 13 they rolled out 20 broadly. On July 24 they cut it to 15 because, during the World Cup, people weren't seeing enough discussion from accounts they didn't follow. It's a nice reminder that each of these constants is a decision someone argued about.

## Then where does Grok come in?

In replies. Specifically, replies under big accounts.

When a reply lands in a thread where the parent's or the root's author has [more than 250,000 followers][replyfilter], it goes to a reply ranker. The code skips smaller threads with the reason `"low_blast_radius"`, which is a great name. The ranker is [Grok 4 mini, with a fast Grok 4.1 fallback][replymodel], and it [must return a score from 0 to 3][replyrange]. The prompt itself is [withheld "to reduce gameability"][prompt]. The inputs, though, are right there in [`thread.py`][signals]:

```python
# grox/core/lm/thread.py, trimmed
if user.num_replies_last_24hrs is not None:
    lines.append(
        f"  - Num Replies Last 24 Hours: {user.num_replies_last_24hrs:.0f}"
    )
# ...
if user.num_legit_blocks_received_last_24hrs is not None:
    lines.append(
        f"  - Num Legit Blocks Received Last 24 Hours: {user.num_legit_blocks_received_last_24hrs:.1f}"
    )
if post.is_pasted is not None:
    lines.append(f"  - Reply Was Pasted: {post.is_pasted}")
```

The full list, in plain English:

- **your profile:** [name, handle, bio, location][userrender], and your follower count
- **your recent behaviour:** how many replies you posted in the last 24 hours, and how many "legit" blocks you received in that time
- **how the reply was made:** whether it was **pasted**, and whether your account has missing client events
- **context:** your top country, account language, and any risky safety label
- **the thread itself:** up to [10 ancestor posts and 2 media items each][threadlimits], plus the root author's bio

So the judge isn't only reading the reply. It's reading how the reply was made.

Two write-side details. A new score [only replaces the old one if it's lower][onlylower], so a reply can be re-judged down but never back up. And a score of exactly 0 attaches a `RiskyHighVizReply` label, [unless][exempt] your user-cred score (PageRank over the follow graph and engagement edges) is at least 66, or you have a grey badge.

> **Note:** I assumed that label meant "your reply is hidden". The code is narrower than that. The [rule][rule] that reads it has audience `NonFollowers` and action drop. But in this repo, it's wired into exactly two places: the [reply and mention notifications][levels]. So the code shows that a zero-scored reply doesn't *notify* people who don't follow you. How the conversation view orders or hides replies isn't in the repo at all. Its [safety levels][safetylevels] cover Home, immersive video, quotes and notifications. I'd rather say "not published" than guess.

## So what does it actually reward?

Strip out the plumbing and the design is pretty legible. The ranker pays for posts you'd send to a specific person, and for replies between people who know each other. It's patient with small accounts, giving their new posts a short, honest audition. And it's wary of anything that makes people want to mute, block or report. Apart from that one bandit, every term is a prediction about *you*, the viewer, not a tally of what happened to the post elsewhere.

Most of the folklore I'd absorbed about this either misreads a ratio or treats a probability as a count. The actual code is a weighted sum, some multipliers, a bandit, and one LLM with a rubric. It's less mysterious than the weather and much better commented.

Anyway, I'll be posting the link to this exactly once. The floor is 0.25 and I have some self-respect. Thanks for the read, and bye!

## References

All code links are pinned to commit [`3b6919f`][commit] of `xai-org/x-algorithm`.

[1] [README, Experiments and Configuration: "we run cron scripts that set the defaults in this repository's code to be the primary production values".][readmeconfig]

[2] [README, Overview and request path.][readmeoverview]

[3] [README, Scoring and Ranking: the list of predicted actions and the weighted sum.][readmescore]

[4] [Phoenix README: the production ranking and retrieval code, and its relation to the earlier Grok-1 port.][phoenixreadme]

[5] [docs/BIDIRECTIONAL_BOOST_CHANGE.md: how the mutual-follow reply boost was tested, shipped and tuned.][bididoc]

[repo]: https://github.com/xai-org/x-algorithm
[commit]: https://github.com/xai-org/x-algorithm/tree/3b6919f87532e57063bb3f1e85c2f509323b513e
[readmeconfig]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/README.md?plain=1#L428-L434
[readmeoverview]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/README.md?plain=1#L78-L90
[readmescore]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/README.md?plain=1#L350-L368
[readmecomponents]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/README.md?plain=1#L295-L301
[isolation]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/README.md?plain=1#L484-L486
[myth]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/README.md?plain=1#L54
[phoenixreadme]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/phoenix/README.md?plain=1#L9-L13
[age]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/params/config.rs#L39
[oonreply]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/filters/oon_retweet_reply_filter.rs#L9-L30
[oonparam]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/params/param.rs#L67-L72
[weights]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/params/param.rs#L404-L533
[mythcomment]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/params/param.rs#L375-L403
[apply]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/xai-value-model/scoring.rs#L57-L59
[oldlike]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/docs/BIDIRECTIONAL_BOOST_CHANGE.md?plain=1#L294
[vmrequest]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/scorers/vm_ranker_request.rs#L38-L47
[oonfactor]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/vm-ranker/params.rs#L211-L222
[oonapplies]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/xai-value-model/scoring.rs#L284-L293
[oonrescore]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/vm-ranker/params.rs#L193-L198
[divparams]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/vm-ranker/params.rs#L175-L192
[divfn]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/xai-value-model/scoring.rs#L261-L263
[divcount]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/xai-value-model/scoring.rs#L265-L282
[coldeligible]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/scorers/author_cold_start.rs#L188-L193
[coldparams]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/params/param.rs#L608-L644
[coldfilter]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/scorers/author_cold_start.rs#L293-L309
[coldtarget]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/scorers/author_cold_start.rs#L216-L226
[coldts]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/scorers/author_cold_start.rs#L243-L283
[coldbeta]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/params/param.rs#L658-L669
[coldtopk]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/home-mixer/params/param.rs#L670-L675
[replyweightfor]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/xai-value-model/weights.rs#L86-L93
[bidieligible]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/xai-value-model/inputs.rs#L19-L23
[bididoc]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/docs/BIDIRECTIONAL_BOOST_CHANGE.md?plain=1#L5-L11
[replyfilter]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/grox/flows/reply_spam/task_filter.py#L185-L226
[replymodel]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/grox/flows/reply_spam/classifier_reply_ranking.py#L33-L42
[replyrange]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/grox/flows/reply_spam/classifier_reply_ranking.py#L164-L170
[prompt]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/grox/flows/reply_spam/prompts.py#L11
[signals]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/grox/core/lm/thread.py#L36-L66
[userrender]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/grox/core/lm/user.py#L18-L24
[threadlimits]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/grox/core/lm/thread.py#L9-L10
[onlylower]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/grox/flows/reply_spam/task_write.py#L124-L141
[exempt]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/grox/flows/reply_spam/task_write.py#L39-L66
[rule]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/visibility-filtering/rules/tweet_rules.rs#L1022-L1027
[levels]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/visibility-filtering/rules/notification_kind.rs#L16-L17
[safetylevels]: https://github.com/xai-org/x-algorithm/blob/3b6919f87532e57063bb3f1e85c2f509323b513e/visibility-filtering/rules/safety_level.rs#L21-L31
