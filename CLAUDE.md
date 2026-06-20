# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Trading robots (Expert Advisors) for the **ProfitUltra** platform by Nelogica, written in **NTSL** (Nelogica Trading System Language). Target asset: **WIN** (mini-índice / Brazilian mini-index futures). Execution mode is **Tick a Tick**.

There is no compiler, package manager, or test runner — NTSL source (`.txt`) is copied into Profit's Strategy Editor, which verifies syntax and executes the strategy against live or historical market data. "Running" or "testing" an EA means the user does it inside Profit; Claude cannot execute the code.

## Repository layout

- `EAs/<EAName>/<EAName>.txt` — canonical NTSL source for each EA. This is the file that gets pasted into Profit.
- `EAs/<EAName>/<EAName>.md` / `<EAName>_documentacao.md` — human-readable strategy spec (logic, parameters, flow). **Read this before touching the `.txt`** — it explains the intent behind the code.
- `EAs/<EAName>/CONTEXTO_<EAName>.md` — session briefing for that EA (current state, open investigations, work in progress). Always read at the start of an EA-specific session.
- `docs/ManualNTSL.pdf` — the NTSL language reference. Consult it for syntax questions (e.g. sections 5.1–5.4 control flow, 19.3 `ConsoleLog`, 25.8 `Media`).
- `docs/Setups.docx`, `docs/Material Complementar*.pdf` — strategy reference material (Larry Williams / Stormer setups, etc.).
- `dados/*.csv` — exported trade logs from Profit (Brazilian CSV: `;` separator, Latin-1/Windows-1252 encoding, comma decimal, dates in `DD/MM/YYYY`).

Six EAs currently live here:

- **BoldGlad-CRTF** — moving-average crossover (arithmetic M1/M2) with breakout confirmation, consolidation-range detection, breakeven steps and trailing stop. The active development target.
- **BoldGlad-CM** — variant of BoldGlad-CRTF. Starts as a copy; specific changes to be indicated by the user.
- **BoldGlad-CML** — variant of BoldGlad-CM aimed at trending days: adds a dominant-trend filter (slow MA gating bias direction), reduced entries (anticipation triggers off by default + post-exit cooldown), and a restart-proof daily loss lock (internal points accounting + DailyResult seeding). Also has opening-candle market entry (P1) and re-entry-after-win (P2). Log tag `BG-CML:`.
- **BoldGlad-CML1** — variant of BoldGlad-CML with the trend filter removed and the entry-reduction removed (no N-candle pause — only the same-candle reentry block remains — and anticipation triggers back on by default). Everything else (P1, P2, faixa, internal daily accounting, trailing) is inherited unchanged. Also has a HiLo stop mode (`pStopHiLo`/`pPeriodoHilo`) that trails the lowest-low/highest-high of the last N candles per bar. Log tag `BG-CML1:`.
- **BoldGlad-M** — forked from BoldGlad-CML1 (12/06/2026), now a **tick-by-tick single-EMA system** (reworked 18/06/2026). Bidirectional, evaluated every tick (does not wait for candle close). Three MAs: M1=100 / M2=200 feed **only the consolidation range**; **M3 = EMA(21)** is the trade signal. **Entry is a candle-breakout stop order:** long requires the previous candle to be **positive** (`Close[1]>Open[1]`), M3 slope `M3[1]−M3[2]` **> `pInclinacao`(15)**, and the breakout level **above M3** → `BuyStop` at `High[1] + pDeslocamento(10)`; short is the mirror (negative candle, slope < −15, `SellShortStop` at `Low[1] − pDeslocamento`). Only one side arms at a time (the prior candle is positive XOR negative), tracked by `nLadoArmado`. Exits (any tick, unchanged): price crossing M3 against the position, stop gain (`pStopGainPts`, 1000), stop loss (`pStopLossPts`, 1000), time and daily lock. Re-entry only on the candle after an exit (`nBarraSaida` guard). After an **M3-cross exit specifically** (`bExitM3Pendente`→`bFiltroDistReentrada`, latched in block 11/read in 10), the breakout level is pushed to ≥ `pDistMinReentrada`(100) from M3 (`max(High[1]+desloc, M3+100)`); this floor does not apply after stop gain/loss/time/lock. (Superseded earlier same-day attempts: market entry by M3-cross, a frozen-M3 distance ref, and an M3±100 stop bracket — all changed per the trader.) Daily giveback lock is configurable via `pPercDevolucao` (default 20% of the day's peak, arms after `pLucroRef`) — replaced the old hardcoded 30%. Consolidation on by default (`pConsolicao=True`). Keeps the internal daily accounting. Log tag `BG-M:`. (Earlier same-day attempts — a two-MA close-based trend system and an EMA9/21 breakout-re-entry rework — were both superseded; the breakout one only traded once per day in a sideways tape.)
- **Larry9.2** — Setup 9.2 (Larry Williams / Stormer): EMA 9×21 trend filter + correction-candle pullback entry with configurable risk/reward multiplier.

## NTSL conventions used in this codebase

These are followed consistently across the EAs — keep them when editing:

- **Parameter names** start with lowercase `p`: `pPeriodoM1`, `pOffsetEntrada`, `pStopFixoPts`.
- **Variable names** are prefixed by type: `v` for Float, `b` for Boolean, `n` for Integer (e.g. `vM1`, `bCruzDetectado`, `nTendencia`).
- **Comments** are in Portuguese, wrapped in `{ }`.
- **Console logs** are prefixed with the EA's tag (`BG-CRTF:` for BoldGlad-CRTF, `BG-CM:` for BoldGlad-CM, `BG-CML:` for BoldGlad-CML, `BG-CML1:` for BoldGlad-CML1, `BG-M:` for BoldGlad-M) so they can be filtered in Profit's event window. Text must be ASCII-safe (no accents) — Profit's console renders them poorly.
- Code is structured into **numbered sections** (`{ 1. RESET }`, `{ 2. DETECCAO DE NOVA BARRA }`, …) — preserve the numbering when adding logic.
- **Header comment block.** Every EA `.txt` starts with a `{ }` comment block (ASCII-safe, Portuguese) that **summarizes** the EA: its **name** and the **main operational behavior — above all the entry and exit conditions of trades** — in a few lines. Keep it concise; the exhaustive description lives in the `*_documentacao.md`.
- **Input ordering.** `input` parameters are grouped by function with `{ --- <grupo> --- }` section comments and ordered to follow the trade flow: bias/signal generation → entry triggers/routes (offset, opening, re-entry) → stop/target & position management (stop, target, breakeven, trailing) → quantity → time window → daily risk/lock → auxiliary filters (consolidation range, etc.).

  These two apply to EAs **created or restructured from here on** — existing EAs may still have verbose headers or a different input order; don't retrofit them without the user asking.

## NTSL gotcha — `FloatToString` in `ConsoleLog`

In some Profit versions, `FloatToString` breaks when concatenated inside `ConsoleLog`. Concatenate Floats directly with `+`:

```
ConsoleLog("Preco = " + vPreco);            { OK }
ConsoleLog("Preco = " + FloatToString(vPreco, 0));  { may error }
```

This is documented in the NTSL manual's `ConsoleLog` example (section 19.3). When writing or editing logs, always use direct concatenation.

## NTSL gotcha — Function call parentheses for no-arg functions

Functions from section 16 of the manual (Calendar functions — `BarDuration`, `BarDurationF`, `CurrentDay`, `CurrentMonth`, `CurrentYear`, `CurrentTime`, `IsWeekend`, etc.) **always require parentheses**, even when called without arguments:

```
bTimeframeOK := (BarDuration() = ...);   { OK }
bTimeframeOK := (BarDuration = ...);     { SILENT runtime failure — Profit accepts the syntax but the comparison aborts mid-script and ALL subsequent code stops running. No error message in the events panel. }
```

This is different from built-in properties like `CurrentBar`, `Date`, `Time`, `Close`, `Open`, `High`, `Low`, `BuyPrice`, `SellPrice`, `HasPosition`, `IsBought`, `IsSold`, `LastBarOnChart` — those work without parentheses.

When unsure, **add the parentheses** — they never break a correct call but their absence may silently kill the entire script.

## NTSL gotcha — Integer-vs-Float comparison with `=`

NTSL does **not** auto-promote Integer to Float when comparing with `=`. Comparing the result of an integer-returning function (like `BarDuration()` → Integer) to a Float-typed input parameter (like `pTimeframe(1)` → stored as Float `1.0`) returns `False` even when the values are numerically identical.

**Do not "fix" this with `Round()`** — `Round()` in NTSL has signature `Round(Valor: Float): Integer`, so passing an Integer argument to it (e.g. `Round(BarDuration())`) silently fails in runtime and returns 0 or garbage. The "fix" then makes the bug worse.

**Correct fix:** use the Float variant of the function. NTSL provides paired functions for many cases:

| Integer variant (returns Integer) | Float variant (returns Float) |
|---|---|
| `BarDuration()` | `BarDurationF()` |

```
bTimeframeOK := (BarDuration() = pTimeframe);    { BUG — Integer = Float, returns False even when both are 1 }
bTimeframeOK := (Round(BarDuration()) = pTimeframe);  { WORSE — Round(Integer) fails silently, returns 0 }
bTimeframeOK := (BarDurationF() = pTimeframe);   { OK — Float = Float, native comparison }
```

Symptom of this bug: `bTimeframeOK` always `False`, EA blocks every operation, and the `Armacao bloqueada` diagnostic shows `TimeframeOK=False` even on a chart with the matching timeframe.

**General rule:** when comparing a function result against an input parameter with `=`, check the function signature in the manual. Pick the variant whose return type matches the input parameter's type (always Float for input params). Don't reach for `Round()`/`Floor()`/`Ceil()` until you've confirmed the inner type is Float.

## Working style the user expects

- **Explain in terms of market behavior, not code.** Describe what the robot *does* (or stops doing) in the market — not which variables change. Example: ✅ "When profit hits 200 points, the stop climbs one step." ❌ "`vStopAtual` is updated inside the conditional block when `pLucroAberto >= pDistAcionaBE`."
- **Informal, conversational Portuguese** is the default tone. The user is the trader/author, not a software team.
- When starting a session on a specific EA, the user will typically ask Claude to read that EA's `CONTEXTO_*.md` and `*_documentacao.md` first. Do that before proposing changes.

## Editing the `.txt` file

- Edits to the NTSL source must keep the numbered-section structure, the Portuguese `{ }` comments, and the variable/parameter prefix convention.
- There's no automated check — after editing, the user pastes into Profit's Strategy Editor and clicks "Verificar Sintaxe". If Claude makes non-trivial changes, flag that this syntax-check round-trip is needed.
- The `.txt` and the `.md` documentation should stay in sync. If behavior changes, update the corresponding section of `*_documentacao.md` as well.

## Protecting working code — REQUIRED before any `.txt` edit

The user has been burned multiple times by Claude breaking logic that was working. Before any non-trivial change to an EA's `.txt`:

1. **Announce the diff scope explicitly.** In the message *before* applying edits, list:
   - **Blocks/conditions that WILL be touched** (with block numbers).
   - **Blocks/conditions that WILL NOT be touched** (the surrounding logic that the user is relying on).
   - **Diagnostic logs (`BG-*-DIAG:`) that stay active in the affected flow** so regressions stay visible.
2. **Never remove a `BG-*-DIAG:` log without explicit confirmation from the user.** Even when the user complains about log noise, ask before deleting: "this log is noisy — can I remove it, or do you rely on it for diagnosis?". Removing diagnostics silently destroys the user's ability to see why a block was bypassed.
3. **Trace the affected candle's full flow before coding.** For changes touching blocks 8–14 of a BoldGlad EA, mentally walk through what happens on a candle that triggers the new logic, AND on a candle that doesn't — to verify no existing path is silently affected. Present that trace to the user before editing.
4. **Treat the `*_documentacao.md` as a contract.** If the doc says "the robot does X under condition Y", do not change the code to diverge from that contract without first calling out the divergence and getting confirmation.

These steps cost a few extra lines of prose but prevent silent regressions like the timeframe `BarDuration()` parens bug and the removed "Armacao bloqueada" diagnostic that made later debugging impossible.

## Git / commits

- Default branch: `main`. Commits and commit messages are in Portuguese, matching the existing `git log` style (e.g. `"Reorganizar repositorio em EAs/, docs/ e dados/"`).
- Only commit when the user asks.
