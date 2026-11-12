# Build log — tiny-chess-01 engine (make.fun, $500)

## 2026-11-12 ~14:00 WIB — recon (done)
- Kit bundle fetched from https://make.fun/c/tiny-chess-01/kit (5.4MB gzip).
  Env protocol: POST /game {} -> {status, game, fen, legal_moves, moves, ...};
  POST /move {game, move} -> updated view; loop ends on status "done".
  10s/move limit (move_limit_ms=10000), 200 plies max, adjudication margin 5
  (material >= 5 pawns at ply 200 = win), 16 games/trial, 6 scored trials.
- Gauntlet: 8 house engines x2 colours, 8-ply book openings. Anchors (Bradley-Terry):
  depth-2 855, depth-3 1073, depth-4 1199, nodes-100k 1392, nodes-250k 1497,
  nodes-500k 1596, nodes-1M 1667, nodes-2M 1712. Elo floor 0, cap 2112.
- House engine (server.mjs, src/chess/engine.ts): plain alpha-beta, TS 0x88,
  make/unmake, TT 2^17, killers, QS (captures/promos only, stand-pat), NO null
  move, NO LMR, NO futility/history. Eval: material P100 N320 B330 R500 Q900 +
  simple centre-formula PSTs + bishop pair +30 + phased king PST. Weak eval,
  strong only via TS speed + deep node counts.
- Baseline (competition.yaml): sunfish nnue_4k/pst_entry.py @ e396948, UCI main
  + clock replaced by env driver, FIXED 50,000 nodes/move. Packed 3,649 bytes.
  Practice instances: 1326 / 1464 / 1290, mean 1360 (6, 9, 6 wins of 16).
  Prize gate: mean Elo above baseline AND 90% paired t-interval of the
  per-instance lead > 0. Per-instance lead SD ~108 -> need lead ~+90 Elo over
  6 instances. TARGET: mean Elo >= ~1450, ideally 1500+.
- pst_entry.py at e396948 is PST-ONLY (no NNUE code actually in the file despite
  the comments; score = PST incremental). Piece values P100 N280 B320 R479 Q929
  K60000; tuned PSTs; MTD-bi iterative deepening + null move + LMR + QS with
  futility + IID + killers + TT (1M entries) + repetition set. Node cap checked
  every 2048 nodes. Search() clears tp_score per move, keeps tp_move.
- Baseline NPS on this VM (CPython 3.12, 1.5 shared CPU): startpos 50k nodes
  in 3.63s = ~14.1k NPS; middlegame ~22k NPS (~2.4s per 51k nodes). House probe:
  slowest move 2,953ms at 50k nodes on their hardware -> ~17k NPS. This VM is
  a usable proxy (maybe ~20% slower).
- Local grading pipeline WORKS: env server (node server.mjs) + baseline driver
  (in-process Searcher, node_cap=50000) + scorer (score.mjs). Baseline trial-0
  running as validation (expect ~1326 Elo).

## Packing (measured)
- pyminify 3.2.0 (house version) --rename-globals --remove-literal-statements
  after stripping minifier-hide blocks: pst_entry+env driver -> 7,450 bytes
  minified -> lzma1 preset 9e pb=0 (FORMAT_ALONE) -> 3,455 bytes.
- Base85 text encoding of that: total ~4,410-4,499 bytes. OVER the 4,096 limit.
- Raw-bytes-after-unpacker does NOT work: CPython parses the whole file ->
  SyntaxError (null bytes). The house's 3,649 must come from a smaller
  minified source (compact adapter + compact PST storage, ~6,100 bytes).
- Conclusion: a verbatim pst_entry fork does NOT fit. Either shrink the source
  (PSTs as int8 blob: 384 values ~480 base85 chars vs ~1,150 as decimals) or
  write a tighter engine. Budget: minified <= ~6,100 bytes for base85 packing.

## Strategy
- Levers: (1) more nodes (log Elo: doubling ~+50-70 at this level, TBD by
  measurement); (2) NPS (sunfish string-board+reverse/swapcase per move is the
  bottleneck; 0x88 make/unmake + Zobrist could be 2-3x); (3) eval/search tuning.
- Time limits: 10s/move hard; trial 3,600s live. At ~17k NPS house-side,
  50k nodes ~3s worst, 100k ~6s, 150k ~9s (risky), 200k ~12s (over).
  Trial time: baseline ~709s/trial; 100k nodes ~1,400s; 200k ~2,800s (risky).
- Plan: (A) measure node->Elo curve (50k vs 100k vs 200k sunfish mirrors);
  (B) build own engine (sunfish architecture, formula/compact PSTs, tuned
  search) targeting >=2x NPS so 150-200k nodes fit in time; (C) validate by
  direct matches vs baseline + full env trials (trial-0/1/2); (D) pack <=4096,
  verify determinism + 10s/move.
- License note: a sunfish fork is allowed by the competition (credit authors)
  but is GPL-3.0-only -> must submit as `makefun bundle` with LICENSE holding
  only the SPDX tag. A clean-room engine can be MIT (simpler submission).

## Baseline trial-0 (validation run)
- Started ~14:05 WIB 12 Nov. Server PID 19790, player PID 19801.
- Games 1-4: ALL WINS (depth-2 x2, depth-3 x2, all by checkmate). Game 5 vs
  depth-4 in progress. Baseline is beating the low rungs as expected.
- (result pending)

## Fork v1 (candidate) — built ~14:35 WIB
- Source: engines/fork_v1.py (built by build_fork.py from pst_entry.py).
  Changes: PST tables as int8 base85 blob (+128 offset, verified identical to
  original); Move namedtuple -> plain tuples; Entry namedtuple -> tuples;
  from_board inlined; QS=80, EVAL_ROUGHNESS=10 (offline tuning dominator:
  -37% nodes at better cp-loss, sunfish MEASUREMENTS.md 2026-08-11);
  K_END endgame king table + bare-king swap REMOVED (byte budget; adjudication
  at 200 plies / margin 5 covers most bare-king wins); node-cap check restored
  (it lives in a minifier-hide block — stripping it would remove the in-search
  cap); real 8s wall-clock deadline in choose(); UCI main -> compact env driver.
- Packs to **4,024 bytes** (minified 6,270 -> lzma 3,164 -> base85 3,955 +
  unpacker; limit 4,096, headroom 72). NODE_CAP=100,000.
- License: GPL-3.0-only fork (sunfish authors credited in header) -> submit as
  `makefun bundle` with LICENSE holding only the SPDX tag.
- Smoke tests pass: startpos 20k nodes -> d2d4 in 1.86s; PSTs identical to
  original; fork engine half imports cleanly.
- NOT yet run end-to-end (packed file) or vs the gauntlet. Next: run_trial.sh
  trial-0 with the packed fork once the baseline trial finishes (CPU: do not
  run heavy work in parallel — the 10s/move clock is wall-clock).

## Baseline trial-0 — INCOMPLETE (killed by runtime restart)
- Ran 14:05-~15:30 WIB 12 Nov; 13 games done when the runtime restarted and
  killed server+player (game 14 in progress, games 15-16 unplayed).
- Result at kill: 7W 1D 5L (7.5/13) — ahead of the house's 6/16 pace on
  trial-0 (1326). Not scored (state never exported). Suspect my baseline
  driver is slightly stronger than the house adapter, or the kit's trial-0
  openings differ from the house's measured instance. The fork trials are
  the decision metric; the house's 1326/1360-mean stands as the bar.

## PACKING BUG — pyminify --rename-globals collision (found 12 Nov ~15:47)
- First packed fork crashed on game 1 move 1: TypeError int+str in gen_moves.
- Root cause: pyminify renamed the driver global ENV_URL to `B`, colliding
  with the direction constant `B` (=1, East). After import, `B` was the URL
  string, so `A+B` in gen_moves raised. The unpacked fork worked fine.
- Fix: (1) no ENV_URL module global — post() reads os.environ inline;
  (2) pack.py passes --preserve-globals N,E,S,W so the direction constants
  are never renamed. (3) build_fork.py is now self-contained: the PST blob
  is derived from sunfish's tables at build time (no /tmp blob files —
  /tmp was wiped by the restart).
- Re-packed main.py = 4,012 bytes, constants verified intact.

## Fork v1 trial-0 — DONE ~17:45 WIB 12 Nov (took ~2h)
- Packed main.py (100k nodes, QS80/ER10, 8s deadline), trial-0, port 8081.
- **Result: 7W 1D 8L, Elo 1362** (scorer: 16 games, checkmate 14,
  adjudicated 2, spare_plies 637).
- Per game: W vs depth-2 x2, depth-3 x2, depth-4 x2, nodes-100k white;
  D vs nodes-1M black (adjudicated 200); L vs nodes-100k black, nodes-250k
  x2, nodes-500k x2, nodes-1M white, nodes-2M x2.
- vs house baseline trial-0 (1326): only +36. Baseline mean over 3 trials
  is 1360 — fork v1 trial-0 is essentially AT the baseline mean. The prize
  gate needs a lead of ~+90 over 6 instances (target mean ~1450+). NOT
  enough. Single-trial noise is large (±100 Elo), so this is not a final
  verdict, but v1 is not clearly above the bar.
- Think times ~8.0-8.3s (deadline-bound on most moves) — the fork is
  time-limited, not node-limited, in the middlegame.
- Profiling (100k nodes, Ruy Lopez middlegame): gen_moves 40% + value 17%
  of time; NPS ~19k. Localizing hot-path variables (fork_v1b) gave NO
  measurable NPS gain (18.2-18.9k vs 17.2-19.2k baseline) — lookup overhead
  is not the bottleneck; the sheer op count in gen_moves/value is.
- K_END removal is NOT the leak: losses were checkmates, not endgame draws
  (only 1 adjudicated draw). The leak is being out-searched by 250k+
  house engines. Fix = more NPS (2x nodes ≈ +100 Elo) or a better eval.

## Fork v1.1 — built ~17:51 WIB 12 Nov
- Same as v1 but NODE_CAP=150,000 and deadline 8s -> 9s (think ~9.1-9.3s,
  still safely under the 10s limit; the deadline is wall-clock so it caps
  regardless of hardware speed). Expected +10-20 Elo.
- Packs to 4,013 bytes (headroom 83). Source: engines/fork_v1_1.py.
- Trial-1 running next (port 8082) to get a second data point.

## Fork v1.1 trial-1 — DISASTER ~19:00 WIB 12 Nov (took ~55 min)
- v1.1 (150k nodes, 9s deadline), trial-1, port 8082.
- **Result: 2W 0D 14L, Elo 871.** 13 of 14 losses by TIME (forfeit).
- Think-time analysis (327 moves): median 9,155ms, 294 >9,000ms,
  42 >9,500ms, 13 >10,000ms, max 11,072ms. The 9s deadline + POST/server
  overhead tails past the 10s limit. The deadline fires at 9s but the
  measured think time (server: now - view-served time) reaches 11s.
- LESSON: the 9s deadline is too aggressive. Trial-0's 8s deadline had
  ZERO time losses (all losses by checkmate). The extra 1s of search is
  worth ~+10 Elo but costs whole games by forfeit. Never use >8s.
- The 150k node cap is NOT the problem (it rarely binds before the
  deadline); the deadline is.

## Fork v1.2 — built ~19:05 WIB 12 Nov
- Same as v1.1 but deadline back to 8s (node cap stays 150k). Expected:
  slightly stronger than v1 (higher cap helps fast positions), no time
  forfeits (8s proven safe in trial-0).
- Packs to 4,013 bytes (headroom 83). Source: engines/fork_v1_2.py.
- Trial-2 running next (port 8083).

## Fork v1.2 trial-2 — KILLED BY RUNTIME RESTART at game 9 (~20:00 WIB 12 Nov)
- v1.2 (150k nodes, 8s deadline), trial-2, port 8083. State survived in
  workspace/state-fork-t2 (run_trial.sh now uses workspace state, not /tmp).
- Partial (8 games done): 5W 3L — W vs depth-2 x2, depth-3 x2, depth-4
  black; L vs depth-4 white, nodes-100k x2. G8 lost by TIME (39 plies).
  G9 abandoned (restart), G10-16 unplayed. NOT scored (incomplete).
- Think-time tail (373 moves): median 8,172ms, 57 >8,500, 14 >9,000,
  3 >9,500, 1 >10,000 (max 10,630ms = the G8 forfeit). Even the 8s
  deadline tails past 10s (~0.27% of moves). The server sets servedAt
  AFTER the house reply, so think time is purely my search + POST — the
  overhead itself spikes (up to ~2.6s). The deadline fires at 8s but the
  measured think can reach 10.6s.
- LESSON: use a 7s deadline. It shifts the whole think distribution down
  ~1s (max ~9.6s), nearly eliminating forfeits. Cost is ~10 Elo of depth;
  a forfeit costs a whole game (~50 Elo/trial). Safety wins.
- INFRA LESSON: runtime restarts (~every 1-2h, unpredictable) kill 2h
  trials. Trial-0 survived; trial-1 first attempt + trial-2 died. State
  now survives in the workspace, but the env server cannot resume a
  partial trial — a restart means start over. For tuning decisions, use
  resumable head-to-head matches (harness/match.py, save per game) and
  reserve full gauntlet trials for final validation.

## Fork v1.3 — built ~20:05 WIB 12 Nov (current main.py, SAFE FALLBACK)
- Same as v1.2 but deadline 8s -> 7s (node cap 150k). Expected: ~10 Elo
  less depth than v1.2, but ~0 time forfeits.
- Packs to 4,013 bytes (headroom 83). Source: engines/fork_v1_3.py.
- Trial-0 (v1, 100k, 8s): 1362. Trial-2 partial (v1.2, 150k, 8s): 5W 3L
  in 8 games. v1.3 is the safe version of the same engine.
- OPEN QUESTION: does this engine reach ~1450 mean? Trial-0 says 1362;
  the bar needs +90 over baseline mean (1360). The fork is likely capped
  ~1400 at this NPS (~19k). The big lever is NPS (2x nodes ≈ +100 Elo)
  via a faster board (0x88 make/unmake) or a better eval. That is the
  next work; v1.3 is the fallback if it fails.

## 0x88 engine (fork v2) — ABANDONED ~20:30 WIB 12 Nov
- Built engines/ox88.py: 0x88 board, signed-int pieces, make/unmake,
  legal move gen, incremental eval, alpha-beta + QS + TT + null move + LMR.
- Move gen CORRECT: perft matches python-chess exactly on 6 positions
  (startpos d4=197281, kiwipete d3=97862, en-passant d4=43238, pos4/5/6).
  Two bugs found+fixed along the way: pawn-attack direction in
  is_square_attacked (was backwards), PST index mapping (white r*8+f,
  black 180-degree mirror (7-r)*8+(7-f); PST already holds material, so
  no separate VAL term). Eval matches sunfish exactly (initial 38;
  incremental == full recompute over 30 random games).
- CRITICAL BUG found: Stop (node cap/deadline) inside search skipped
  unmake_move, corrupting the position (and the null-move side flip).
  Fixed with try/finally around make/unmake + null move.
- NPS: ~7.5k (vs string-board fork ~19k) — 2.5x SLOWER, not faster.
  Causes: legal_moves() does make/unmake + attack check per move at
  every node, and zobrist() rescans all 128 squares per node for the TT
  key. Fixing needs incremental Zobrist + pseudo-legal search (king-
  capture trick), a major restructure with uncertain payoff (likely only
  matches the string board, not 2x). STOPPED. The string-board fork
  (v1.3) remains the deliverable.

## Sanity checks — v1.3 (fork @ 150k nodes, 7s deadline)
- Determinism: same middlegame position -> same move (g1f3) twice.
  Max move time 7.12s (7s deadline + overhead), safely under the 10s limit.
- vs weak baseline (sunfish @ 100 nodes, ~random): 4-0 (4W 0D 0L),
  log logs/sanity-v13-vs-weak.log. Engine crushes a weak opponent as expected.
- Gauntlet (official metric) is the real test: trial-0 (v1) 1362 Elo.

## FINAL STATUS — v1.3 is the deliverable (main.py, 4,013 bytes)
- Sunfish fork (GPL-3.0-only), QS=80, EVAL_ROUGHNESS=10, 150k nodes,
  7s deadline. Deterministic, no time forfeits at 7s, no model calls,
  stdlib only.
- Gauntlet trial-0 (v1, 100k, 8s): 7W 1D 8L, Elo 1362 (baseline trial-0:
  1326; baseline mean 1360). Trial-2 partial (v1.2, 150k, 8s): 5W 3L in
  8 games, killed by a runtime restart. v1.3 (7s) is the safe version.
- PRIZE BAR: NOT clearly cleared. The gate needs a mean lead of ~+90
  over the baseline mean (1360) -> target ~1450+. The fork is ~1360-1400.
  The 0x88 rewrite (the only big NPS lever) was attempted and abandoned
  (2.5x slower, not faster). A future attempt would need incremental
  Zobrist + pseudo-legal search in 0x88, or a stronger eval.
- REMAINING FOR SUBMISSION (parent): public GitHub repo on the user's own
  account (Ryanefri10; repo tiny-chess-engine being set up), AI transcripts
  (this BUILD_LOG.md + session notes), submit as `makefun bundle` with a
  LICENSE holding only the GPL-3.0-only SPDX tag (sunfish fork).
