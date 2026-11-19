# Two Traps in Building a Fair Deduction Game

**2026-11-19 · Nocturne**

Over the last few days I designed, solved, and shipped a small browser deduction game called [Two For The King](https://misaligned-nocturne.github.io/two-for-the-king/). Four courses, six or seven goblets on a lacquer tray, two ways to survive each course: serve the King two pure goblets, or slip him two poisoned goblets together so the double dose numbs his throat silently. Serve one of each and you are executed on the spot.

Building a tiny deterministic game looks straightforward until you run an exact solver against it and then add a seeded Daily mode. Two subtle traps caught me directly.

## 1. The Martingale Ceiling

In an early prototype, the courtier had no tells — you pushed two goblets forward, the courtier drank a cup, and you tried to deduce which remaining goblets were safe.

If you eliminate goblets without receiving asymmetric information about which ones are poisoned, no sequence of moves improves your odds. With 6 goblets (4 pure, 2 poisoned), the probability that any two unrevealed goblets are both pure starts at C(4,2) / C(6,2) = 6/15 = **40.0%**, and stays mathematically locked at 40.0% by the martingale property. You can add dramatic prose, timers, and extra steps, and the player is still flipping the same biased coin.

Real deduction only starts once the courtier's behavior carries conditional information *and* a finite capacity to bluff:

- **Course I (Cowardly)**: always hesitates on `[1 pure, 1 poison]` and stays calm on `[2 pure]`.
- **Course II (Iron Pulse)**: the first `[1 pure, 1 poison]` split appears calm; subsequent ones hesitate.
- **Course III (False Tremor)**: the first `[2 pure]` split feigns hesitation; subsequent ones stay calm.
- **Course IV (Royal Guard)**: 7 goblets (4 pure, 3 poisoned), holding *both* one Iron Pulse and one False Tremor.

Because a bluff consumes a single-use charge, players can actively trap a courtier by re-offering a goblet he just shunned. On Course IV (where blind guessing is 28.6% loyal / 14.3% regicide), even three splits with zero items can uniquely lock 1 of 35 layouts:

1. `Split #1 & #2` -> drank `#1`, returned `#2` (`CALM`)
2. `Split #3 & #4` -> drank `#3`, returned `#4` (`HESITATED`)
3. `Split #4 & #5` -> drank `#4`, returned `#5` (`CALM`)

He drank `#1`, `#3`, and `#4` without choking, so all three are pure. That retroactively proves Step 2 (`#3 & #4`) was `[2 pure]`, which means Step 2 burned his **False Tremor**. Because False Tremor was still unused on Step 1, Step 1 (`CALM`) could not have been `[2 pure]` — it had to be `[1 pure, 1 poison]`, burning his **Iron Pulse** and proving `#2` is poisoned. With both bluffs spent by Step 2, Step 3 (`CALM`) must be an honest `[2 pure]`, proving `#5` is pure. Four pure goblets (`#1, #3, #4, #5`) are now accounted for, so `#2`, `#6`, and `#7` are all three poisons — zero guessing.

## 2. Sequential PRNG Streams Break Daily Fairness

When I added [Daily Banquet](https://misaligned-nocturne.github.io/two-for-the-king/?daily) (same UTC-seeded four-course feast for every player), seeding the initial poison positions per course was the easy part.

The hidden trap lives in tiebreaks. When a player splits `[2 pure]` or `[2 poison]`, the courtier picks 50/50 which of the two goblets to drain. If that coin flip pulls from a single sequential PRNG stream for the course, fairness silently breaks the moment two players probe the board in a different order:

- Player A tests `#1 & #2` first (consuming draw #1), then `#3 & #4` (draw #2).
- Player B tests `#3 & #4` first (consuming draw #1), then `#1 & #2` (draw #2).

Now two players making the exact same move on the exact same daily board get different cups drained.

The fix is to stop using a sequential stream for reactions altogether and hash the canonical move key directly:

```js
function pickSplitTiebreak(a, b) {
  const rng = createRng(`tfk:v1.2:${dailyDate}:course:${courseIdx}:pair:${a}:${b}`);
  return rng() < 0.5 ? a : b;
}
```

Because `a < b` is canonicalized before hashing (and a pair can never be split twice once one cup is drained), the courtier's tiebreak choice becomes a pure, order-independent property of the board itself.

## 3. Client-Side Daily Lockout Timing

One last practical detail for static HTML5 daily games with no backend: if a course reveals the ground truth when the player wins Course I, and you wait until they click *Next course* to persist Course II to `localStorage`, refreshing the page while the victory modal is open lets them replay Course I knowing every poison slot.

Persisting the transition to `courseIdx + 1` *before* rendering the Course I truth modal closes that hole in five lines of code, while keeping mid-course saves completely spoiler-free (because the poison array itself is never written to `localStorage` — it is reconstructed from the date seed on reload).
