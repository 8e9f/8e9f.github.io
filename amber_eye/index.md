---
title: "Amber Eye: a Slay the Spire engine approaching top-level play"
---

# Amber Eye: a Slay the Spire engine approaching top-level play
{:.no_toc}

**Amber Eye** is an engine which plays Slay the Spire Ironclad A20H runs.
In a sample of 100 games, Amber Eye won 90, including a streak of 19.
The engine was trained using [AlphaZero](https://deepmind.google/blog/alphazero-shedding-new-light-on-chess-shogi-and-go/)-style techniques:
a neural network starts with almost no prior knowledge of game mechanics and learns to win through playing games and observing their outcomes.
Amber Eye was trained from scratch on my desktop computer for about one month, playing about 13.8 million games over that time.
Its playstyle is not quite like any other player I have watched,
and I believe that the ideas exhibited by its gameplay are worth further study.

* TOC
{:toc}

## Motivation

Why build an engine for Slay the Spire?

- It would be very cool for an engine to discover new macro ideas (or at least rediscover ideas used by humans), improving our understanding of the game.
- I actually don't know much about deep learning, and this project presented a good opportunity to learn.
Additionally, there are various interesting engineering challenges associated with implementing systems to train and run the model, as described below.
- I hope to rekindle some community interest and excitement around Spire 1, which I feel still has more to discover.

## Results

### Evaluation

I [have uploaded](https://youtube.com/playlist?list=PLBt3ZH12eKzw) the full set of 100 sample games for viewing.
Below are a few of the games I found interesting.
- [Run 80](https://youtu.be/TmoDgD4Ghf4) relies heavily on Unceasing Top strategies to get by with a deck with weak output.
- [Run 12](https://youtu.be/vMD71lvmqfk) is in a strong position throughout but plays an interesting loop with 3 Reckless Charges.
- [Run 96](https://youtu.be/pEH4aXzSzhA) bricks both boss relics and gets bailed out by the Act 3 shop.
It also shows the effects of playing Spire for 60 hours straight in the Masked Bandits fight.
- [Run 100](https://youtu.be/G2rmTCEqSZg) does a poor job preserving its resources, exacerbating a weak position.
- [Run 90](https://youtu.be/V1uVBcM0eJY) plays early game with Wild Strike+ and Fire Breathing as its damage.

The 100-game sample was played with the engine controlling a real game instance,
using a modified version of [CommunicationMod](https://github.com/ForgottenArbiter/CommunicationMod).
The engine was given a search budget of 262k positions per move (about 30-60 minutes per game).

The engine does not have knowledge of the seed or RNG state*[^leakedinfo].
I have ensured this to the best of my ability by only sharing visible or previously revealed state from the game with the engine.
Formally, the model aims to learn to play "ideal Spire",
an abstract version of the game where random outcomes are drawn truly at random, instead of from a seeded RNG.
This is the version that players generally agree is more interesting (optimally solving "real Spire" would involve reverse engineering the seed).
Both the training and evaluation settings use the RNGFix mod;
otherwise, the model would be able to learn certain aspects of [correlated RNG](https://forgottenarbiter.github.io/Correlated-Randomness/).

My first reaction when seeing the engine's gameplay was that it constantly made mistakes.
My second reaction was that it wins a lot despite this, so something interesting is happening.
I am a "mediocre" player myself (~30-40% Ironclad A20H winrate) so I am not particularly qualified to analyze its macro,
but I will do my best here to describe its overall approach to the game.

The engine appears to aim for hard forcing looping decks in the endgame
(exhaust down and play many Pommel, Shrug, Rage, Dropkick etc.),
although it does sometimes incorporate alternate win conditions (Snecko, strength scaling, Barricade).
It is perfectly happy to play greedily to enable looping strategies,
e.g. by picking Dark Embrace on floor 1 or buying The Abacus in the Act 1 early shop.
Notably, it is comfortable using Havoc or Sever Soul as core components of its exhaust engine
and is flexible about how it gets its loop payoff,
opening more paths to building this type of deck.

The engine's micro ability is usually rather terrible.
It bleeds HP and wastes damage or potions for no reason, often multiple times per run.
For example, in [run 51](https://youtu.be/5loL9tEnOOg&t=2301) of the sample,
it manages to throw a very strong position entering the Heart fight, with the help of a bad draw order.
(I replayed the fight and won easily.)
Occasionally, however, the model is able to pull off some flashy lines.

Overall, the general ideas are quite similar to the approaches used by good players,
but the details look very off in a way that I'm not able to understand.
Watching the engine's gameplay, I was very surprised that it was capable of reaching such a high winrate.

### Tier lists

The engine gives estimated winning probabilities for each position,
which we can use to generate Objectively Correct™ rankings for all items in the game.
We can directly measure the engine-estimated impact of each choice on winrate from a sample of logged games.

For the charts below, we define the *value* of a choice as the difference in win probability in percentage points between that choice and the next best option
(or skipping), or 0 if it is not the best choice.
The charts are sorted by average value.
The values were measured on ~43 000 self-play games played at the end of the training run.
Due to being played with a lower search budget, they have a slightly lower winrate (~85%) than the runs in the sample.
The lines on the chart surrounding the dots represent the 95% confidence interval.

{% include charts/neow.svg %}

The engine really likes option 3, as long as it is not rare colorless or max HP.
The downsides are generally rated as lose max HP > take damage > lose gold > curse.
If option 3 is not good, option 2 is usually pickable.

Boss swap is rated very poorly by the engine, never being picked and costing about 7-8 percentage points of winrate.
Most swaps were rated as negative; only Pandora's Box, Astrolabe, Runic Pyramid, and Fusion Hammer were clearly positive.

On the two-option blessing (Lament vs +10% max HP), the engine picks Lament almost every time,
but only slightly favours it in terms of value.
On average, the two-option blessing is rated about 2 percentage points worse for winrate than the full blessing.

{% include charts/cards_red.svg %}

All the premium exhaust cards are highly rated, as expected given the engine's playstyle.
Offering is in a tier of its own.
A significant portion of the card pool is rated as garbage and never picked.

{% include charts/cards_red_floor1.svg %}

The floor 1 card ratings don't appear too out of the ordinary.
The biggest surprises for me are Sever Soul and Reckless Charge as high as they are, Hemokinesis relatively far down the list,
and Fire Breathing being picked a significant fraction of the time.

In the sample runs, the engine appears to be significantly more willing to pick the garbage attack commons early than indicated on this chart.
In floors 1-6, Cleave was picked 4 times, Wild Strike 6 times, Thunderclap 4 times, and Sword Boomerang 2 times,
which is roughly double the rate expected from the self-play games.
This is the only major macro discrepancy I have noticed between the sample and the self-play games.
A possible explanation is that the larger search in the sample looks farther ahead into the next hallway fights and sees the value of the attacks.
Given this difference only appears with a deep search, the impact on winrate is likely marginal.

{% include charts/cards_colorless.svg %}

Colorless cards have a much smaller sample size.
Because the engine generally doesn't like picking rare colorless from Neow,
most of the rare card picks come from Sensory Stone, which probably underrates them in terms of strength
(by Act 3, most runs are in a strong position and have less room to increase their win chance).
Notably, Hand of Greed and The Bomb are not rated highly, which likely explains why it dislikes the Neow option.

{% include charts/relic_rewards.svg %}

The three books are highly rated, in agreement with how often the engine will acquire them.
The HP spent to get them is not included in this chart's rating, which inflates their placement a bit.
Pocketwatch far outclasses the rest of the non-boss relics.
Dead Branch is picked only 80% of the time but is still near the top of the chart,
while Bottled Flame is skipped 80% of the time at the bottom.
Horn Cleat, Toxic Egg, Bag of Preparation, and Centennial Puzzle outperform their rarity.

{% include charts/boss_relics.svg %}

To no one's surprise, Pyramid is literally off the charts in terms of value.
On average, getting Pyramid is the highest-impact choice in the whole run.
Unexpectedly to me, Pandora's Box and Runic Cube are the next-highest rated.
Generally, the engine is quite happy to pick Runic Cube even from the Act 1 boss reward.
It also doesn't care too much for energy: 31 of the games in the sample were played with no energy boss relic.

{% include charts/potion_rewards.svg %}

Fairy is rated far above all the other potions.
This is maybe not surprising; given the engine's horrible potion management,
it is better able to use the undrinkable Fairy in contrast to its tendency to waste the other drinkable potions.
The Fairy is used both as a short-term resource to be spent and as Act 4 insurance.
It is also a high-priority purchase in shops.

{% include charts/shop_relics.svg %}

Many shop relics are highly valued, especially The Abacus, which often turns into the core of the deck's block engine.
Despite a model limitation where the engine's visibility into the deck order is limited to the top two cards,
Frozen Eye is still often purchased.

{% include charts/shop_cards.svg %}

Shop card purchase rates roughly track their general rating.
The engine is quite willing to purchase removes even in Act 1.

{% include charts/shop_colorless.svg %}

Colorless card purchases are more niche.

{% include charts/shop_potions.svg %}

Potions are frequently bought (and thrown away on the next floor).
Fairy is the top-priority purchase in the shop on average.

{% include charts/events_act1.svg %}

For Scrap Ooze, the engine clicks at least once about 96% of the time.
Most of the time, it will click at least 3-4 times before giving up, depending on how much HP it has.
Interestingly, it often skips the Golden Idol, although this choice probably does not impact winrate significantly.
It never takes the upgrades in Shining Light; generally, it does not rate upgraded basics highly.

{% include charts/events_act2.svg %}

Notably, the engine virtually never upgrades all basics in Ancient Writing,
which is generally considered to be one of the strongest Act 2 event options.
In the context of the engine being very remove-pilled, however, this makes a bit more sense.
It also rarely takes Apparitions and Bites.
JAX tends to be taken with Runic Cube.
In The Joust, the engine tragically has little faith in Noodles.

{% include charts/events_act3.svg %}

For the Mysterious Sphere, the engine is usually in a strong position entering Act 3,
so it often doesn't consider the risk of fighting the double Orb Walkers worth it.
Mark of the Bloom is never taken.
It also takes the Madnesses a surprising amount.
I don't see the vision, as the Madnesses seem to typically not perform well in its decks,
but who am I to judge.

{% include charts/events_shrines.svg %}

The engine actually chooses to trade faces sometimes, which seems terrible to me,
but maybe it doesn't rate the bad faces as badly.

{% include charts/enemies.svg %}

HP lost in this chart refers to average net HP change, including healing from Burning Blood, Reaper etc.

The engine doesn't tend to take the Dead Adventurer Lagavulin fight in especially weak positions, as the chart might suggest.
Taking the fight too often may be one of the engine's blind spots.
More generally, the engine seems to prefer spreading out the risk more throughout the course of the run,
instead of concentrating it in Act 4.

{% include charts/keys.svg %}

"Never" covers games the engine loses.

The engine prefers to take the keys notably earlier than most players,
although this probably has a small overall impact on winrate.
In a large majority of games, it has already Recalled before entering Act 3.

{% include charts/campfire.svg %}

The rates here denote the percentage of time the option is taken versus the total number of rest site visits.
For Lift, Dig, and Toke, the percentage is over the rest sites where the relic is owned.

The engine doesn't mind resting a lot and generally only values a few key upgrades
(Pommel Strike, Feel No Pain, Body Slam, Havoc, Offering etc.)
When it owns a rest site relic, it usually makes use of it.

## Intermission

Now, on to the technical details of how I built the engine (with a bit of Spire trivia thrown in).
If you are not a giga nerd who is interested in the nitty gritty of how to build such an engine,
you can probably stop reading here or after the next couple of sections.
Check out the [runs from the sample](https://youtube.com/playlist?list=PLBt3ZH12eKzw),
and I will be working on extending the engine to the other three characters.

## AlphaZero

The [AlphaZero paper](https://arxiv.org/abs/1712.01815)
demonstrated top-performing engines for chess, go, and shogi,
but more importantly provided a general-purpose method to train computers to play a relatively broad family of games
(discrete action space, two-player, deterministic, perfect information).
The recipe works roughly as follows.

Apart from very simple games, it is impossible to exhaustively enumerate each game state and derive a perfect strategy from it.
Thus, since very early on, game-playing programs have combined position evaluation (estimate the value of the current state)
and search to improve the evaluation (simulate playing some moves to figure out the best line and get a more accurate judgement of the position).
Historically, the main research areas have been to make the value estimation more accurate and to improve the search algorithm.
AlphaZero's contribution is to bundle value and search into a self-improving loop driven by the learning of a neural network.

More specifically, the AlphaZero network takes as input a game position and predicts both its estimated value and a *policy*,
a probability distribution over the legal moves from the position, indicating what the network thinks is the best move.
The policy can be incorporated into the search algorithm to concentrate on moves which the network thinks are good.
The model generates its own training data in self-play, where it plays games against itself using the search;
labels are obtained from observing the games' outcomes and statistics from each search.

The two main ideas are that the value and policy predictions complement each other
(more accurate policy leads to improved search leads to more accurate value estimation and vice versa),
and that the network augmented by the search is a bit stronger than the network itself,
so the model can continually bootstrap itself by learning from the stronger search-improved targets.

Spire differs in many important ways from the setting in the AlphaZero paper, and so needs domain-specific adaptation:
- Spire is a single-player game.
    - In a two-player game, in self-play the model plays against an opponent that is just as bad as it, and so is able to receive training signal about won and lost games.
    In a complex single-player game like Spire, the model will start out by always losing, and will then learn that all positions are lost and "give up",
    never receiving signal that helps it make progress.
    - AlphaZero deliberately adds noise to decisions in self-play games, in order to be able to discover strategies its network may be underrating.
    This noise comes at a direct cost to play strength.
    For a two-player game, this is less important as the opponent's strength is reduced by the same noise.
    But for a single-player game, it becomes critical to find a favourable balance between exploration and play strength.
- Spire has a much longer time horizon than the games in the paper.
Typical chess, go, or shogi games end in fewer than 300 or so moves (by either player).
A typical game of Spire can run well over 1000 decisions, meaning the model has to be able to plan much farther ahead,
making it harder to propagate learnings from the end of the game back to the start,
and exacerbating the [value memorization](#value-memorization) problem.
- Spire has a considerable amount of randomness, with random rolls affecting the game state about every 2-3 decisions.
This adds sampling noise to the search, increases the amount of compute required to look ahead to a given depth, and reduces the signal that can be learned from the game's outcome as it is partially influenced by random chance.
- Chess, go, and shogi are all easily representable in a format that a neural network can process.
A position in Spire is considerably more diverse in its game state and action space, requiring more care in its encoding.
- The models in the AlphaZero paper were trained on a cluster of 5000 TPUs for self-play and 64 TPUs for training, over the course of up to 13 days (for go).
I have a single GPU available for both self-play and training, and while it is quite a bit faster than a single decade-old TPU,
I don't have the time to wait hundreds of years for the model to train.

While these are significant challenges, not all the solutions need to be invented from scratch,
as other projects such as [KataGo](https://github.com/lightvector/katago) and [Leela Chess Zero](https://lczero.org/)
have already improved on AlphaZero's method.
Most of the techniques that follow are adapted from followup research to AlphaZero.

## Throughput

Reinforcement learning is monstrously compute-hungry.
The model's improvement is driven by self-play, requiring a lot of compute to generate a tiny amount of value signal from the game outcomes.
As the model learns, its strength increases and thus the training target also changes, requiring more fresh self-play data, and so on.
Given this constraint, the guiding design principle motivating all of the following items was to make the learning as compute-efficient as possible.

In total, Amber Eye was trained from scratch in about one month, including time spent on experimentation.
I think that by applying learnings from the initial run, a model of equal strength could be trained in about one week.

## Simulator

When searching, Amber Eye uses its own reimplementation of the game engine,
built using [sts_lightspeed](https://github.com/gamerpuppy/sts_lightspeed) with some minor enhancements.
The implementation is quite fast, searching about 800 000 positions per second on small (CPU-bound) networks on my 8-core machine.

sts_lightspeed is mostly accurate to the real game, but contains some bugs,
which were discovered and patched over time by having Amber Eye play in the simulator and the real game in parallel and flag state divergences.
The patched simulator is faithful enough to not significantly impact the model's winrate,
but Spire itself has a lot of weird bugs which were not worth replicating:
- Several RNG rolls (Discovery, Courier restocks) are based on unseeded RNG and thus can vary.
This has no impact on the game from the perspective of "ideal Spire" and so doesn't actually affect the model's decision making.
- There are several timing-based bugs, including: 
    - Reaper can fail to heal when playing it ends the combat and there are enough queued animations from e.g. power effects.
    Fewer animations are required if Fast Mode is on.
    - There is a Corruption display bug where skills are shown to have nonzero cost and can only be played with sufficient energy,
    but don't actually use the energy.
    Some, but not all, of the skills generated from Chrysalis can be affected by this bug, but which ones depends on whether Fast Mode is on.
- When obtaining Searing Blow from Match and Keep with Molten Egg, somehow you get a "Searing Blow+2" which deals 17 damage
(Searing Blow+1 deals 16 and Searing Blow+2 deals 21).
I believe this also happens with Searing Blows created by e.g. Foreign Influence with Master Reality in play.
This is really funny, but also some nonsense that I don't want to rewrite the simulator to support.

The simulator gives a minor amount of help to the engine, mostly to reduce compute cost:
- in positions with one legal action, it is automatically taken;
- gold from rewards is automatically taken;
- actions are canonicalized (e.g. if multiple Defends are in hand, the simulator only exposes playing the first one); and
- buying Prismatic Shard is banned (I did not implement the other card colours, with my apologies to the Shard fans).

## Search

There are different options for what algorithm to use when performing the search.
They all use the network's outputs to guide the search, but differ in how they use the outputs to decide what move to explore next.
AlphaZero uses [PUCT](https://inria.hal.science/hal-00835352v1/document),
but Amber Eye uses [Gumbel AlphaZero](https://openreview.net/pdf?id=bERaNdoegnO).
The implementation details of the algorithms are not very important to describe here;
the main benefits of using Gumbel search are that it empirically performs better than PUCT at playing Slay the Spire at equal search budgets,
and that Gumbel search makes a much more favourable tradeoff between exploration and playing strength.

For PUCT to work well in AlphaZero, a significant amount of exploration noise has to be added to the network predictions,
in addition to randomly sampling some moves from the policy instead of taking the top choice.
With Gumbel search, I found it was sufficient to only apply the more modest Gumbel noise to the search and no sampling was needed.

Gumbel search also scales down to much lower search budgets than PUCT.
To generate policy targets, AlphaZero used the distribution of visit counts from search, which is very noisy for low visit counts.
Gumbel search also uses the value network's output to construct the policy target, reducing its variance.

### Randomness

To accurately model ideal Spire, the search must also not reveal or exploit hidden information,
e.g. draw pile order, future enemy intents, or relic queue order.
However, the search must still be able to progress past actions with random outcomes, or else it would barely be able to look ahead.

A simple way to address this is to sample random outcomes.
When stepping the simulator, it flags if any hidden information was revealed by this step,
which can be done by checking if any of the RNGs were used.
Some hidden information, like draw order or intent under Runic Dome, is randomized at an earlier step (e.g. when reshuffling the deck),
so these cases are also detected and resampled.
When a step reveals hidden information, its RNG is rerolled to sample a different random outcome consistent with the prior game state.

If we performed a fresh reroll every time a step was visited, the search would never be able to progress past that step.
Instead, the search adopts [double progressive widening](https://hal.science/hal-00542673v1/document) for random outcomes:
it puts a limit on the number of resamples per move which slowly grows as the move gets revisited.
If the move is visited when the random sample count is already at the limit,
the search proceeds deeper into one of the existing random outcomes rather than generating a new one.

### Transpositions

Playing Defend $$\rightarrow$$ Strike or Strike $$\rightarrow$$ Defend results in the same position[^order],
but a regular tree search would treat these lines as completely distinct,
wasting computation searching the same position in two different branches and adding noise from the two branches' differing random outcomes.
[Monte-Carlo Graph Search](https://github.com/lightvector/KataGo/blob/master/docs/GraphSearch.md)
addresses this by merging the identical positions and searching a DAG instead of a tree.
In the case of Spire, cycles are possible in the position graph (e.g. playing a "useless" infinite),
so we must additionally detect and cut these cycles when implementing the graph search.

Because of the complexity of its implementation, MCGS was *not* used in self-play games.
However, the benefit of MCGS grows as the search budget increases, so it was implemented to play the sample games.
Empirically, applying MCGS to Spire effectively cuts the average branching factor by about 1.2$$\times$$.
This has a modest benefit at the search budget used in self-play,
but compounds to about a 15$$\times$$ effective compute advantage versus tree search at the 262k-position search budget used in the sample.

## Model architecture

### Micro and macro

Spire positions can roughly be divided into combat and non-combat states (commonly referred to as "micro" and "macro", respectively).
The consequences of in-combat decisions are mostly local to that combat,
in that playing better lines saves HP for that fight and leads to a better post-fight position.
It is tempting, then, to also perform this division on the model side, and have separate networks handle combat and non-combat positions.
However, micro and macro are more intertwined than they look at first glance.
Resource use (potions), meta-scaling items (Feed), and relic setup (Incense Burner) obviously carry over to future combats,
but other combat decisions can depend more subtly on the overall game state.
For example, one might choose to play a higher-variance line which loses more HP on average if it opens a chance of highrolling and being able to path to an extra elite.
Because of this, optimally playing combats requires planning beyond just the combat.

For this reason, I consider it very important that a single network evaluates all game states.
A single network better applies the [bitter lesson](http://www.incompleteideas.net/IncIdeas/BitterLesson.html);
it can learn for itself how micro interacts with macro, avoiding the need to figure out how to stitch the networks together.

### Transformers

Amber Eye's network primarily consists of a transformer which processes the game state.
Most people's experience with transformers is with large language models, and so they are commonly associated with next-token prediction, but this is a bit of an oversimplification.
At a high level, a transformer is a type of neural network which can process variable-size sets of features
and which is very effective at learning interactions between them.
To enable this, the features are represented by points in an abstract high-dimensional embedding space.

More specifically, the data follows this computation path:
- The game state is encoded into tokens, which are points in the high-dimensional latent space.
For example, unique cards, event choices, relics, and enemies are encoded into one token each in the same embedding space.
Roughly speaking, the tokens can be thought of as the model's scratch space,
and part of its job is to learn how to organize the scratch space well so that similar concepts are close together.
For example, the model may learn to place "good earlygame damage card" along one of the dimensions, or whatever other concept that helps it predict.
- The set of tokens passes through a sequence of transformer blocks, each consisting of an attention and feed-forward layer.
Each transformer block modifies the token sequence.
    - The attention operator is [the famous one](https://arxiv.org/abs/1706.03762).
    It allows tokens to exchange information with each other
    (for example, comparing offered cards with the cards in your deck in a card reward screen, or your block cards in hand versus incoming damage).
    - The feed-forward operator has been studied since before machine learning existed.
    Its role in the transformer is roughly to remember relevant facts (for example, retrieving what cards synergize well, or recalling Defend blocks 5).
    It projects each token into a higher-dimensional space, applies a nonlinear activation function, then projects it back to the token space.
- After passing through the transformer layers, the tokens capture high-level interactions between the features.
The information from the tokens is then extracted by the heads, where the tokens are projected to produce the final predictions for value and policy.

### Feature encoding

To process the game state in the transformer, its information must first be preprocessed into tokens.
Each token's inputs consist of three parts:
- the token type (run metrics, player metrics, player status effects, monster, card, relic, potion, option, map);
- categorical features, up to 4 different IDs associated with the token e.g. card, relic, monster type, monster intent, card zone;
- numerical features, up to 64 floats for other numerical information (mostly for run and player tokens).

The token then passed into the transformer is the sum of its type embedding, its categorical embeddings,
and a type-specific projection of its features, all LayerNormed.
A game state has an average of about 40 tokens, mostly from cards and relics.

### Actions

A game like chess easily admits a fixed-size action space (source square $$\times$$ target square).
In Spire, many types of action are possible, and are contextual on the game state.
Thus, we also prepare a list of legal actions as an input, and the policy is directly computed over this list.
An action can be represented by the following:
- the action type (play card, use potion, end turn, take reward, buy shop item, visit map node, etc.);
- the source token (card being played, potion used, reward taken, item bought, etc.);
- the target token (targeted monster, potion offered to Ranwid, etc.); and
- up to 8 contextual features (e.g. if an attack is lethal, HP from event option).
These were probably not necessary as the information is already encoded in the source and target tokens.

The action list is used by the model to directly fetch the source and target tokens from the transformer's output and compute the hidden state for the policy.
More precisely, for each action $$a$$ we compute

$$
e_a = \text{emb}(k_a) + W_s\, h_{s_a} + W_t\, h_{t_a} + W_f\, f_a + W_c\, c
$$

where $$\text{emb}(k_a)$$ is the action kind embedding,
$$h_{s_a}, h_{t_a}$$ are the source and target output tokens respectively,
$$f_a$$ is the vector of action features,
$$c$$ is a pooled summary of other tokens,
and the $$W$$s are learned weight matrices.
If there is no source or target for an action, a "none" embedding is used instead.
The vector $$e_a$$ is then projected by a MLP to a logit, and the policy is obtained by taking the softmax of these logits over the action list.

This setup has the advantages of only needing to compute the policy over legal actions,
and automatically fetching the relevant tokens in the policy computation.

### Heads

The heads convert the processed token stream into the value and policy predictions.
Apart from the policy heads, each head consumes a summary consisting of the encoded Run token plus the average of other tokens processed by a shared summary MLP.
Each head is a two-layer MLP processing this summary to produce its prediction.
The prediction task and role of each head is discussed in the following section.

## Training

### Labels

In a pure AlphaZero setup, only the recorded outcome of the game and search policy would be used to train the network,
meaning we would only extract 1 bit of information for the value head to train on per game,
a terrible ratio given the tens of thousands of network evaluations spent to play the game.
However, we can make use of more information from the game to better train the network's internal representation.

The network has the following heads:
- The policy head, as described above.
- A soft policy head, which is trained to predict a flattened version of the search target.
This technique is adapted from KataGo.
For the policy target distribution $$\mathbf p$$, the soft policy head predicts $$\mathbf p^{1/4}$$.
The policy target tends to be very spiky, putting almost all its weight on the top move.
By predicting the flattened policy target the model is forced to learn more about the relative ranking of the non-top moves.
The soft policy is only used in training.
- The outcome head, predicting the run outcome.
Instead of just predicting win/loss, the outcome head also predicts the floor on which the run loses.
By training this classification task, more signal from the game is able to be used.
- The current-floor HP head, predicting the HP at the end of this floor, as a classification task over each HP value capped at 255.
- The per-floor HP head, predicting the HP for each floor from the current floor until the end of the run.
Unlike the current-floor HP head, the per-floor HP head predicts the HP value as a scalar, as using the same classification task would make the head too large.

### Reward shaping

At the start of training, playing games to purely maximize the network's predicted win chance quickly collapses into a state of learned helplessness.
The network is weak to start and cannot win any games,
so it quickly learns to predict that all states are completely lost and never improves.
Instead, we can make use of the other prediction heads to nudge the engine in a direction that makes progress
until it is able to win.
Formally, we define a utility function for the position based on the heads' predictions and ask the engine to maximize the utility
instead of the win chance.

In the beginning, the utility was defined in the following way.
Define the floor progress reward for a position $$s$$ as

$$
F(s) = \frac1{57} \left(\sum_{f=1}^{56} f \cdot P(\text{die on floor } f \mid s)\right) + P(\text{win} \mid s).
$$

Effectively, $$F(s)$$ gives the self-play agent a reward of $$1/57$$ when it advances a floor or wins.
The probabilities are estimated from the model's outcome head.
Similarly, we can define the HP reward as

$$
H(s) = \frac{1}{56}\left((\text{floor}(s)-1) + \sum_{f=\text{floor}(s)}^{56} \frac{\mathbb E[\text{HP on floor } f \mid s]}{255}\right),
$$

which rewards the model for preserving HP.
The utility function initially used was $$U(s) = F(s) + H(s)$$.

The model made consistent progress under this utility and won its first game after about 460 000 games played and 1.5 hours of training.
I was quite surprised that only these terms were sufficient,
as long fights still don't get any intermediate signal from the reward,
and long-term objectives like keys are hard to learn about.

After a few more hours, the model reached significant winrate under this utility.
I then updated the utility to $$P(\text{win} \mid s) + 0.25H(s)$$ for out-of-combat states and $$F(s) + 0.5H(s)$$ for in-combat states.
Empirically, these improved the performance in self-play,
and the changes can be interpreted as allowing focusing on building decks for the endgame in out-of-combat states.
I expected to eventually be able to entirely remove the other shaping terms and purely optimize for $$P(\text{win} \mid s)$$,
but this utility form continued to perform better even at the end of training.
One possible interpretation of this fact is that the model's estimation of $$P(\text{win} \mid s)$$ has some model error associated with it,
and the other shaping terms help mitigate the error, even though they introduce a bias which does not perfectly align with optimizing for winrate.

### Value memorization

Because training data is generated via self-play, training samples originating from the same game are correlated with each other.
This presents challenges for training the value network.
Samples from the same game all have the same outcome,
so the value network can quickly learn to memorize properties of the game (deck and relics) and use that to predict the game's result,
overfitting instead of properly generalizing.

To address this, first we apply [playout cap randomization](https://arxiv.org/html/1902.10565v5#S3) as in KataGo.
Moves are selected with probability $$p = 0.25$$ to undergo a full search with a larger number of simulations,
and the remainder of the moves are played using a fast search with much fewer simulations.
Only the full moves are used in training.
This increases the number of games that can be played per unit of compute, as we can spend less compute on the fast searches,
thus increasing the diversity of value labels that can be generated in the same amount of time.
While the overall production of training samples is reduced, the tradeoff is worth it to gain more value labels.
The fast moves also cost self-play strength, but this is also partially mitigated by not applying exploration noise to them.
Empirically, using a fast move budget $$1/8$$ the size of the full move budget cost roughly 1 percentage point of winrate.

Playout cap randomization partially mitigates the memorization problem,
but due to Spire's longer game length, significant memorization is still observed at $$p = 0.25$$.
$$p$$ could be lowered even further to reduce it,
but this increases the proportion of compute spent on fast moves which don't generate training data.
This tradeoff becomes quite unfavourable for small $$p$$.
Thus, I altered the value label itself as further mitigation.

The value labels were redefined as a combination of the search result and an exponential weighted average of future moves' search outcomes.
This injects diversity into the value labels to reduce overfitting,
although the labels from positions close to each other in the sequence of moves are still correlated.
It also corresponds well to the game itself, where the long-horizon outcome is noisy and dependent on future random rolls,
so it makes sense to downweight learning from outcomes far in the future.

This label definition also has the downside of having a low weight on the real outcome,
so the value estimate can remain persistently miscalibrated.
Notably, the engine significantly underestimates its own winrate,
and the underestimation increases in positions earlier in the game (where the end outcome is more distant).
The gap is 10 percentage points or more at the start of the game.
However, as long as the engine is able to score better positions as higher value,
the actual numerical value of the predicted winrate is irrelevant.

### Exploration

If the network learns that certain choices are bad early on and starts never picking them,
it will never get the signal it needs to correct misjudging these choices.
For example, midway through the training run, I noticed that the network rated Apotheosis as the worst card in the game by far,
because it never picked it and reconsidered.
Typical exploration strategies like sampling from the policy or adding uniform random noise still barely make the choices that need to be sampled,
and had a large cost to play strength.
To address this, I created a domain-specific exploration scheme inspired by the
[Quantile Balancing](https://arxiv.org/html/2607.24653v2#A3) technique used by Kimi K3.

For choices like card picks, event choices, Neow bonuses, and relics,
we first set a target percentage of games that each choice should appear in (I used 1-3%).
For items below the target, we can keep track in the gamelog of how far they are below the choice actually taken (in terms of policy logits).
We can then determine the bias that needs to be added to each of the items below the target such that the target pick count is reached.
In the search, the below-target items are forced to be kept as a final candidate,
any search that includes a below-target item becomes a full search,
and the bias is added after search completes to force the picking of unexplored items at roughly the target rate.

Instead of a hard threshold like in Quantile Balancing, the boosted item is picked with probability

$$\min(1,\,c\cdot\exp(-m/T)),$$

where $$c$$ is a parameter computed per item by binary search to hit the target pick rate,
$$m$$ is the gap between the boosted item and the top pick after the bias is added, and $$T = 4$$ is a temperature parameter.
Effectively, situations where the item is closer to being the top pick are more likely to be picked under this scheme,
but situations with larger gaps are also chosen occasionally to have more variety in the picks.

The forced picks successfully corrected the bias on items like Apotheosis,
and only affected ~0.2% of moves with a modest play strength cost of ~3 percentage points.

## Diminishing returns

There are many ways in which more compute can be traded for more playing strength.
Generally, you have to spend more compute than you get playing strength back,
so it saves a lot of time to avoid spending more compute when the model is weak and there are low-hanging fruit for improvement,
and only spend once it becomes necessary.
I applied this idea to several axes:
- Searching more positions per move makes the self-play slightly stronger[^searchscaling],
but is only needed towards the end of training when the network is so strong that its raw predictions approach the strength of the network with search,
inhibiting its ability to continue learning.
The majority of training used 64 positions per full move and 16 positions per fast move, and probably could have worked with even less compute.
The search budget was stepped up over time, ending at 256 positions per full and 32 positions per fast move.
For reference, AlphaZero uses 800 positions searched for every move.
- Larger networks can reach higher strength but require more data to train and are slower to evaluate.
I trained 3 networks in sequence with ~0.44M, ~1.2M, and ~5.4M parameters, achieving self-play winrates of ~55%, ~78%, and ~85% respectively.
Once the smaller network is trained, the larger network can quickly surpass its strength by using it as a teacher.
First, we can pretrain the larger network on the smaller network's generated games and training labels.
This is a commonly used technique, applied by e.g. KataGo and Leela Chess Zero.
Then, the smaller network can be evaluated on the large network's games to help it bootstrap more quickly.
I used [kickstarting](https://arxiv.org/abs/1803.03835), a simple form of on-policy distillation which mixes the teacher's prediction into the training labels, until the student's strength exceeded the teacher's.
- At the very start of training, I allowed the search to "savescum" by giving it knowledge of future RNG outcomes.
This improves play strength not just by having the advance knowledge, but also by avoiding wasting search budget on sampling RNG.
In the end, this did not end up being very useful and was turned off early,
because savescumming confers an absurdly large advantage to the player
(the tiny network quickly hit 88% winrate, and dropped back down to ~40% when savescumming was turned off).
I also found that optimal savescumming strategies did not correspond well to good strategies for the real game with randomness,
as they overly relied on finding out the outcomes from the search.

## Engineering

### Compact game representation

To train the model, we need to store a large replay buffer of training examples across a large number of games
to ensure each batch has a good mix of training data.
At the end of training, I used a replay buffer of 10M states, but a naive storage of the raw training data would take about 7-8 KB per state,
which is way too large for my machine.
I used the following compression techniques:
- Each training sample stores its labels for the outcome and HP distributions, per-floor HP, and policy.
The value distributions were quantized to an 8-bit fixed-point format,
which has more than good enough quality considering distribution values all lie between 0 and 1.
Only the nonzero interval of each value distribution was stored.
This reduced the size of the targets from ~1.5 KB/sample to ~120 bytes/sample.
- For storing the game logs on disk, each game can be stored as a seed and the sequence of actions taken in the game,
and can be reconstructed by replaying the actions from the start.
Each action can be packed into a 32-bit value, compressing the game log data itself down to 4 bytes per move.
For use in the replay buffer, we also need to store the training targets for each full search move, making for ~140 bytes per training state.
- The buffer contents need to be loaded into memory for fast access in training,
but the compressed disk format above takes a significant amount of compute to decompress it (replaying the game and encoding the token sequences),
making it unsuitable for this purpose.
However, we can still store the data efficiently by keeping a common dictionary of tokens which are referenced by index in replay buffer states.
Most tokens, especially cards and relics, are shared between many different states, leading to a high degree of reuse.
This compression scheme achieved about ~550 bytes/state.

### Efficient search

Search in an AlphaZero setting is difficult to efficiently implement as described.
In most abstract descriptions of search algorithms, the search tree/graph is expanded one node at a time, and each node needs a GPU evaluation,
leading to many roundtrips between CPU and GPU.
This naive approach also does not utilize the GPU properly, which is much more efficient at processing large batches in parallel.

In self-play, these issues can partially be addressed by playing many games simultaneously.
On my 8-core, 16-thread machine, I ran 14 worker threads each running hundreds of self-play games.
In each search, network evaluation requests were submitted into a queue,
and separate threads packed the requests from the queue into larger batches and passed them to the GPU.
The latency of GPU network evaluations is hidden by having many games in flight.

The large game count is still not enough to fully utilize the GPU,
and does not apply to the single-game setting,
so we additionally allow multiple searches in the same game tree using
[virtual visits](https://dke.maastrichtuniversity.nl/m.winands/documents/multithreadedMCTS2.pdf).
In self-play, up to 4 nodes per candidate move's subtree were allowed to be visited in parallel.
For the single-game evaluation, much more parallelism needs to be extracted,
and the strength penalty is not as large for a large simulation count,
so 2048 parallel nodes per candidate were allowed.
Both the above techniques are standard in AlphaZero-esque setups.

On the GPU side, the model is small enough that its weights in FP16 fit comfortably in my GPU's L2 cache,
and it has high arithmetic intensity, so it was relatively easy to get high compute utilization.
The kernel achieved about 75% MFU in self-play after discounting padded attention operations.

## Next steps

I am not sure why anyone would read this very long article, but hopefully it was informative.

Much more is still possible.
I have many ideas for further improving the engine's performance,
and of course the other three characters are still unexplored.
I am excited to continue developing the engine and sharing the results.

## Footnotes
{:.no_toc}

[^leakedinfo]: There are three situations I am aware of where hidden information is leaked to the engine.

    - When defeating the burning elite with Black Star, 
    any potion reward generated is dropped due to the limit of 5 rewards per reward screen,
    but the potion chance is still updated.
    The game passes the potion chance to the engine regardless.
    - When fighting The Maw with Incense Burner activating on turn 2,
    it is impossible to distinguish the small (5) and big (30) hit intents until later in the fight.
    However, the game directly passes the enemy intent to the engine whenever Runic Dome is not owned.
    This can also similarly happen for enemies whose attack damage is randomized (Louse, Darklings).
    - When in an event fight which has a relic reward (Colosseum, Mind Bloom etc.),
    the relic reward is determined at the start of the fight and removed from the relic pool.
    The information about the relic pool is passed to the engine,
    which could be used in principle by deducing that relics still in the pool will not be given as the reward.
    The network is unable to take advantage of this hidden information, however,
    as it is not given the relic pool as input.

    Each of these situations occur very rarely,
    and their knowledge gives a very small advantage to the player,
    so the overall impact on winrate is negligible.

[^order]: In the absence of other effects like Juggernaut.

[^searchscaling]: Towards the end of training, the observed scaling was very roughly 1 percentage point of winrate per quadrupling of search budget.