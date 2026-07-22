---
title: "They Accuse X of Manipulating the Algo. The Code Is Public. They Still Haven't Read It."
summary: "Brivael Le Pogam argues that accusations of algorithmic manipulation on X collapse under the open-sourced recommendation code: the real panic is loss of narrative monopoly, not opacity."
date: "2026-07-22"
link: "https://x.com/brivael/status/2078940564792426577"
author: "Brivael Le Pogam"
---
They Accuse X of Manipulating the Algo. The Code Is Public. They Still Haven't Read It.
by Brivael Le Pogam, posted on July 19th, 2026

On what the X source code really says — and on the panic it triggers

On July 15th, 2026, three things happened on the same day.

Elon Musk commented on polls placing Marine Le Pen in the lead and called her "the last hope of the country". The French left immediately cried foreign interference. And, that same day, Musk announced he would open-source the entirety of X's code — "without exception" — inviting third-party auditors to verify that the published code is exactly what runs in production.

Take a second to measure the scene. A man is accused of secretly manipulating a machine. The same day, the man pins the machine's blueprints to the wall and invites anyone to check that there is no false bottom.

That is not an awkward coincidence. It is the whole affair in a nutshell.

What the algorithm actually does

Let us start with the only terrain that does not lie: the code. The repository xai-org/x-algorithm is public, under the Apache 2.0 license, written 57 percent in Rust and 42 percent in Python, updated — according to the commitment X made in January 2026 — every four weeks. Here, without jargon, is how it builds your "For You" feed.

A conductor called Home Mixer receives your request. It fetches candidates — posts that might interest you — from two reservoirs.

The first, Thunder, holds posts from accounts you follow. It lives in memory, updates in real time, and answers in a fraction of a millisecond. That is your "in-network".

The second, Phoenix Retrieval, fishes in the global corpus of posts you do not follow but that might speak to you. Technically, it is a "two-tower" model: one tower turns you (your engagement history) into a digital fingerprint, the other turns each post into a fingerprint, and the system brings like closer to like. That is your "out-of-network", the door to discovery.

Candidates from both reservoirs are then enriched (text, media, author, verification status), then sifted through a series of filters: duplicates are removed, posts that are too old, your own posts, accounts you have blocked or muted, keywords you have silenced, what you have already seen.

Then comes the heart of the reactor: Phoenix, a transformer based on Grok. For each surviving post, it does not compute a vague "relevance". It predicts about fifteen distinct probabilities: probability that you will like, reply, repost, quote, click, dwell, follow the author… and also the probability that you will mark it "not interested", block, mute, or report.

The final score is a simple weighted sum of those probabilities. Positive actions count positively; negative actions — block, mute, report — count negatively, and push down content you would hate. Sort by score, keep the top of the basket, run one last safety filter (deleted content, spam, violence, gore), and serve you the feed.

That is all. It is neither magic nor a conspiracy. It is a machine that predicts what you will do, trained on what you have already done.

The point nobody wants to face: "No Hand-Engineered Features"

In the repository's design decisions, the very first one, numbered 1, is four words: No Hand-Engineered Features. No hand-tuned feature.

You have to understand why that is explosive.

Historically, a recommendation algorithm is an accumulation of human rules. "Boost verified accounts." "Penalize external links." "Downrank this type of content." "Amplify that other one." Each of those rules is a valve. And every valve is a place where a hand — an engineer's, a boss's, a regulator's, an ideologue's — can press quietly. That is exactly where manipulation lives when it exists: in the hidden heuristics.

What the repository says, black on white, is that they removed those relevance rules. The documentation is unambiguous: "we eliminated every hand-engineered feature and most of the system's heuristics". Relevance is no longer decided by a committee. It is learned, end to end, from your engagement sequence by the transformer.

In other words: the place where people traditionally suspect a thumb on the scale has been dismantled. Not hidden — dismantled, and the announcement of its dismantling is published.

Let us be honest all the way, because that is where the argument becomes airtight rather than naive: yes, filters remain. Safety, spam, violence, a content analysis service called Grox that applies platform rules. Every platform has them; it is inevitable. But here is the difference that changes everything: those filters are also in the repository. Line by line. Named. Readable. If tomorrow a political rule hid in the code, it would be on GitHub, forked 4,300 times, pinned in thirty seconds by the first researcher who looked, and turned into a global scandal before noon.

That completely reverses the trial against him. He is accused of opacity. He is the only platform in the world whose opacity is, literally, impossible.

The asymmetry that the "interference panic" cannot sustain

Now ask the only question that really matters. On which platform can you audit algorithmic manipulation?

Instagram? Black box. YouTube? Black box. TikTok? Black box. Twitch? Black box. Meta even announced cutting political content reach by default, without anyone being able to read a line of it. On all those platforms, the charge "the algorithm is suffocating this side" is rigorously unfalsifiable: you can neither prove it nor refute it, because the code is sealed.

And it is precisely the only platform that published its code — then announced it would publish all the rest, with third-party verification of production — that is dragged before a prosecutor for "algorithmic abuse".

Read that slowly. With Europol's help, they raided the offices of the most transparent platform in the history of social networks, over a manipulation that its own code lets anyone verify or disprove. While the real black boxes, the ones no judge can open, prosper quietly.

If you were sincerely looking for manipulation, you would start with the systems you cannot see. You fixate on the only one you can read. That is not an investigation into opacity. It is a punishment of transparency.

What this is really about: the end of a monopoly

So what is this about, if not the algorithm?

It is about distribution. For half a century, access to mass attention ran through a bottleneck: a few newsrooms, a few studio sets, a few editorialists. That bottleneck had a lean — no point denying it; the sociology of the journalism profession tilts massively one way, and everyone knows it. It was not necessarily a conspiracy; it was a milieu, a culture, an in-group. But the result was the same: a single, centralized filter decided what deserved to exist in the debate.

That bottleneck blew open.

An algorithmic feed learned from the signals of hundreds of millions of individuals has no editor-in-chief. It is a distributed order — Hayek would have recognized it instantly: dispersed information, aggregated by a mechanism no planner controls, producing a result nobody decreed. Facing that distributed order, the old world no longer has the hand. It no longer has the button.

Hence the vocabulary. "Reactionary internationale," said the head of state. The word is revealing: you do not use "internationale" to describe a disagreement of opinion; you use it to designate a coordinated conspiracy. That is René Girard's reflex: when mediation loses its power, it does not question itself — it looks for a scapegoat, a single culprit on whom to unload the anxiety of loss. Musk, X, the algorithm: the culprit is found, the ritual can begin.

Except you can no longer burn a sorcerer whose grimoire is public.

That is the true nature of the "interference panic". It is not fear of manipulation — they cannot even show the manipulation, since the code would contradict it. It is a deeper, more intimate fear of having lost the monopoly on the narrative. For decades, they were the algorithm. They decided what rose and what disappeared. Today an open system does it in their place, without asking their opinion, and they call that system "interference" — because in their grammar, everything that escapes their curation is an anomaly to correct.

Transparency as the final word

People will object that the French investigation also targets other grievances — Grok's behavior, problematic content. Fair enough. Those are separate questions, to be handled on their own ground, and I do not mix them here. But on the central charge, the one that launched the whole machine — the algorithm as a secret weapon of interference — the answer is already written, in Rust and in Python, published, forked, auditable, and soon verified in production by independent third parties.

The old world's line of defense rested on a premise: that algorithmic manipulation was a charge you could neither prove nor lift, and therefore could brandish indefinitely. That premise has just died. There is now a platform where the charge is verifiable — and therefore refutable. And a refutable charge that people keep hammering without ever verifying stops being a democratic concern. It becomes what it always was: an attempt to take back by law what was lost by technology.

The code is open. Soon, all the code will be. The question is no longer whether the algorithm is manipulated — the answer is one git clone away. The question is how much longer people will pretend otherwise, hoping nobody goes and reads.

Nobody is fooled anymore. And you do not inventory an open feed. You read it.
