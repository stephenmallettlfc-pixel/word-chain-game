/**
 * Word Chain — Hint System
 * ========================
 * Drop this file into your project and call initHints() after your puzzle loads.
 *
 * INTEGRATION STEPS:
 * 1. Import/include this file in your main JS bundle.
 * 2. Call: initHints({ start, end, wordList, getCurrentChain, onHintUsed })
 *    - start:           string  — start word (e.g. "CAT")
 *    - end:             string  — target word (e.g. "DOG")
 *    - wordList:        Set     — your full valid word set (same one used for validation)
 *    - getCurrentChain: fn      — returns the player's current word chain array e.g. ["CAT","COT"]
 *    - onHintUsed:      fn(n)   — called with total hints used; use to add penalty steps to score
 * 3. Call: renderHintButton(containerEl) to inject the hint button into the DOM.
 * 4. Call: resetHints() when a new puzzle starts.
 *
 * HINT TIERS (each costs +1 step penalty):
 *   Tier 1 — Reveals which letter position changes next (e.g. "Change the 2nd letter")
 *   Tier 2 — Reveals the actual next word on the optimal path
 *   Tier 3 — Shows the full remaining optimal path
 */

// ─── BFS Solver ────────────────────────────────────────────────────────────────

/**
 * Find the shortest path from `start` to `end` using BFS.
 * Returns an array of words including start and end, or null if no path exists.
 */
function bfsSolve(start, end, wordList) {
  if (start === end) return [start];

  const queue = [[start]];
  const visited = new Set([start]);

  while (queue.length > 0) {
    const path = queue.shift();
    const current = path[path.length - 1];

    const neighbours = getNeighbours(current, wordList);
    for (const neighbour of neighbours) {
      if (neighbour === end) return [...path, neighbour];
      if (!visited.has(neighbour)) {
        visited.add(neighbour);
        queue.push([...path, neighbour]);
      }
    }
  }

  return null; // no path found
}

/**
 * Get all valid one-letter neighbours of `word` from `wordList`.
 */
function getNeighbours(word, wordList) {
  const results = [];
  const letters = 'abcdefghijklmnopqrstuvwxyz';
  const w = word.toLowerCase();

  for (let i = 0; i < w.length; i++) {
    for (const letter of letters) {
      if (letter === w[i]) continue;
      const candidate = w.slice(0, i) + letter + w.slice(i + 1);
      if (wordList.has(candidate)) results.push(candidate);
    }
  }

  return results;
}

// ─── Hint State ────────────────────────────────────────────────────────────────

let _config = null;
let _hintsUsed = 0;
let _optimalPath = null; // full BFS solution from start→end
let _solverCache = {}; // cache partial paths from any word→end

/**
 * Initialise (or re-initialise) the hint system for a new puzzle.
 */
function initHints({ start, end, wordList, getCurrentChain, onHintUsed }) {
  _config = { start, end: end.toLowerCase(), wordList, getCurrentChain, onHintUsed };
  _hintsUsed = 0;
  _solverCache = {};

  // Kick off BFS in the background so first hint is fast
  setTimeout(() => {
    _optimalPath = bfsSolve(start.toLowerCase(), end.toLowerCase(), wordList);
  }, 0);
}

/**
 * Reset hint counter (call between puzzles).
 */
function resetHints() {
  _hintsUsed = 0;
  _solverCache = {};
  _optimalPath = null;
}

// ─── Hint Logic ────────────────────────────────────────────────────────────────

/**
 * Get the next hint. Returns a hint object:
 * {
 *   tier: 1 | 2 | 3,
 *   message: string,         // human-readable hint text
 *   nextWord: string | null, // the actual next word (tier 2+)
 *   remainingPath: string[]  // full remaining path (tier 3)
 * }
 */
function getNextHint() {
  if (!_config) throw new Error('initHints() must be called first');

  const { end, wordList, getCurrentChain } = _config;
  const chain = getCurrentChain();
  const currentWord = chain[chain.length - 1].toLowerCase();

  // Find the optimal path from current word to end
  const cacheKey = currentWord;
  if (!_solverCache[cacheKey]) {
    _solverCache[cacheKey] = bfsSolve(currentWord, end, wordList);
  }
  const pathFromHere = _solverCache[cacheKey];

  if (!pathFromHere || pathFromHere.length < 2) {
    return {
      tier: 0,
      message: "You're already at the target!",
      nextWord: null,
      remainingPath: [],
    };
  }

  const nextWord = pathFromHere[1]; // optimal next step
  const tier = Math.min(_hintsUsed + 1, 3);
  _hintsUsed++;
  if (_config.onHintUsed) _config.onHintUsed(_hintsUsed);

  // Find which position changes
  const changedPos = findChangedPosition(currentWord, nextWord);
  const posLabel = ordinal(changedPos + 1);

  let message;
  switch (tier) {
    case 1:
      message = `Try changing the <strong>${posLabel} letter</strong>.`;
      break;
    case 2:
      message = `Next word: <strong>${nextWord.toUpperCase()}</strong>`;
      break;
    case 3: {
      const rest = pathFromHere.slice(1).map(w => w.toUpperCase()).join(' → ');
      message = `Remaining path: <strong>${rest}</strong>`;
      break;
    }
    default:
      message = `Next word: <strong>${nextWord.toUpperCase()}</strong>`;
  }

  return {
    tier,
    message,
    nextWord,
    remainingPath: pathFromHere.slice(1),
  };
}

function findChangedPosition(a, b) {
  for (let i = 0; i < a.length; i++) {
    if (a[i] !== b[i]) return i;
  }
  return 0;
}

function ordinal(n) {
  const s = ['th', 'st', 'nd', 'rd'];
  const v = n % 100;
  return n + (s[(v - 20) % 10] || s[v] || s[0]);
}

// ─── UI ────────────────────────────────────────────────────────────────────────

/**
 * Inject the hint button + tooltip into a container element.
 * Styling uses CSS custom properties so it inherits your game's theme.
 *
 * @param {HTMLElement} containerEl — element to append the hint UI into
 */
function renderHintButton(containerEl) {
  // Inject styles once
  if (!document.getElementById('wc-hint-styles')) {
    const style = document.createElement('style');
    style.id = 'wc-hint-styles';
    style.textContent = `
      .wc-hint-wrap {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 8px;
        margin-top: 12px;
      }

      .wc-hint-btn {
        display: inline-flex;
        align-items: center;
        gap: 6px;
        padding: 8px 18px;
        border: 1.5px solid var(--color-border, #d1d5db);
        border-radius: 999px;
        background: var(--color-surface, #fff);
        color: var(--color-text-muted, #6b7280);
        font-family: inherit;
        font-size: 0.875rem;
        font-weight: 500;
        cursor: pointer;
        transition: border-color 0.15s, color 0.15s, background 0.15s;
        user-select: none;
      }

      .wc-hint-btn:hover {
        border-color: var(--color-accent, #f59e0b);
        color: var(--color-accent, #f59e0b);
      }

      .wc-hint-btn:active {
        transform: scale(0.97);
      }

      .wc-hint-btn svg {
        width: 16px;
        height: 16px;
        flex-shrink: 0;
      }

      .wc-hint-penalty {
        font-size: 0.75rem;
        color: var(--color-text-muted, #9ca3af);
      }

      .wc-hint-penalty strong {
        color: var(--color-accent, #f59e0b);
      }

      .wc-hint-bubble {
        display: none;
        padding: 10px 16px;
        background: var(--color-surface-alt, #fef9ec);
        border: 1.5px solid var(--color-accent, #f59e0b);
        border-radius: 10px;
        font-size: 0.9rem;
        color: var(--color-text, #374151);
        text-align: center;
        max-width: 280px;
        animation: wc-hint-pop 0.2s ease;
      }

      .wc-hint-bubble.visible {
        display: block;
      }

      .wc-hint-tier-badge {
        display: inline-block;
        font-size: 0.7rem;
        font-weight: 700;
        text-transform: uppercase;
        letter-spacing: 0.05em;
        color: var(--color-accent, #f59e0b);
        margin-bottom: 3px;
      }

      @keyframes wc-hint-pop {
        from { opacity: 0; transform: translateY(-4px); }
        to   { opacity: 1; transform: translateY(0); }
      }
    `;
    document.head.appendChild(style);
  }

  const wrap = document.createElement('div');
  wrap.className = 'wc-hint-wrap';
  wrap.innerHTML = `
    <button class="wc-hint-btn" id="wc-hint-btn" aria-label="Get a hint">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <circle cx="12" cy="12" r="10"/>
        <path d="M9.09 9a3 3 0 0 1 5.83 1c0 2-3 3-3 3"/>
        <circle cx="12" cy="17" r=".5" fill="currentColor"/>
      </svg>
      Hint
    </button>
    <span class="wc-hint-penalty" id="wc-hint-penalty" style="display:none">
      +<strong><span id="wc-hint-count">0</span></strong> step penalty
    </span>
    <div class="wc-hint-bubble" id="wc-hint-bubble">
      <span class="wc-hint-tier-badge" id="wc-hint-tier"></span>
      <div id="wc-hint-text"></div>
    </div>
  `;

  containerEl.appendChild(wrap);

  document.getElementById('wc-hint-btn').addEventListener('click', () => {
    const hint = getNextHint();
    const bubble = document.getElementById('wc-hint-bubble');
    const tierBadge = document.getElementById('wc-hint-tier');
    const hintText = document.getElementById('wc-hint-text');
    const penaltyEl = document.getElementById('wc-hint-penalty');
    const countEl = document.getElementById('wc-hint-count');

    const tierLabels = { 1: 'Hint — Position', 2: 'Hint — Word', 3: 'Hint — Full Path' };
    tierBadge.textContent = tierLabels[hint.tier] || 'Hint';
    hintText.innerHTML = hint.message;

    bubble.classList.add('visible');
    // re-trigger animation
    bubble.style.animation = 'none';
    bubble.offsetHeight; // reflow
    bubble.style.animation = '';

    countEl.textContent = _hintsUsed;
    penaltyEl.style.display = 'block';
  });
}

// ─── Exports ───────────────────────────────────────────────────────────────────

// If using ES modules:
// export { initHints, resetHints, renderHintButton, getNextHint, bfsSolve };

// If using CommonJS / bundler:
if (typeof module !== 'undefined') {
  module.exports = { initHints, resetHints, renderHintButton, getNextHint, bfsSolve };
}

// If loaded as a plain script tag, expose on window:
if (typeof window !== 'undefined') {
  window.WordChainHints = { initHints, resetHints, renderHintButton, getNextHint, bfsSolve };
}
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Word Chain</title>
<link href="https://fonts.googleapis.com/css2?family=DM+Serif+Display&family=DM+Mono:wght@400;500&family=DM+Sans:wght@400;500&display=swap" rel="stylesheet" />
<style>
  :root {
    --bg: #faf9f6; --surface: #ffffff; --border: #e8e4dc; --text: #1a1814;
    --text-muted: #8a8680; --text-hint: #b8b4ae;
    --amber-bg: #fef3e2; --amber-border: #f5c842; --amber-text: #7a5c00;
    --blue-bg: #edf4ff; --blue-border: #91b8f5; --blue-text: #1a3f7a;
    --pink-bg: #fdeef5; --pink-border: #e8a0c8; --pink-text: #7a1f4a;
    --green-bg: #edf7ee; --green-border: #7ecb84; --green-text: #1a5c22;
    --red-text: #c0392b; --radius: 10px; --radius-lg: 16px;
  }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { background: var(--bg); color: var(--text); font-family: 'DM Sans', sans-serif; min-height: 100vh; display: flex; flex-direction: column; align-items: center; }
  header { width: 100%; border-bottom: 1px solid var(--border); background: var(--surface); padding: 1rem 1.5rem; display: flex; align-items: center; justify-content: center; gap: 10px; }
  .logo { font-family: 'DM Serif Display', serif; font-size: 26px; letter-spacing: -0.5px; color: var(--text); }
  .logo-sub { font-size: 12px; color: var(--text-muted); letter-spacing: 0.08em; text-transform: uppercase; }
  .help-btn { margin-left: auto; padding: 6px 14px; border-radius: var(--radius); border: 1px solid var(--border); background: var(--surface); color: var(--text-muted); font-size: 13px; font-family: 'DM Sans', sans-serif; cursor: pointer; }
  .help-btn:hover { background: var(--bg); color: var(--text); }
  .overlay { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.4); z-index: 100; align-items: center; justify-content: center; }
  .overlay.open { display: flex; }
  .modal { background: var(--surface); border-radius: var(--radius-lg); padding: 2rem; max-width: 400px; width: 90%; position: relative; }
  .modal-title { font-family: 'DM Serif Display', serif; font-size: 22px; color: var(--text); margin-bottom: 1rem; }
  .modal-close { position: absolute; top: 1rem; right: 1rem; background: none; border: none; font-size: 20px; color: var(--text-muted); cursor: pointer; line-height: 1; }
  .modal-close:hover { color: var(--text); }
  .rule-block { margin-bottom: 1rem; }
  .rule-block p { font-size: 14px; color: var(--text-muted); line-height: 1.7; margin-bottom: 0.5rem; }
  .example { display: flex; flex-wrap: wrap; gap: 6px; align-items: center; margin: 0.75rem 0; }
  .ex-word { background: var(--blue-bg); border-radius: var(--radius); padding: 5px 11px; color: var(--blue-text); font-weight: 500; font-family: 'DM Mono', monospace; font-size: 14px; letter-spacing: 0.06em; }
  .ex-word.start { background: var(--amber-bg); color: var(--amber-text); }
  .ex-word.end { background: var(--green-bg); color: var(--green-text); }
  .ex-arr { color: var(--text-hint); font-size: 11px; }
  .level-pills { display: flex; gap: 8px; margin: 0.75rem 0; }
  .pill { padding: 5px 12px; border-radius: 20px; font-size: 12px; font-weight: 500; }
  .pill.easy { background: var(--amber-bg); color: var(--amber-text); }
  .pill.medium { background: var(--blue-bg); color: var(--blue-text); }
  .pill.hard { background: var(--pink-bg); color: var(--pink-text); }
  .got-it { width: 100%; padding: 12px; border-radius: var(--radius); border: 1px solid var(--border); background: var(--bg); color: var(--text); font-size: 15px; font-family: 'DM Sans', sans-serif; font-weight: 500; cursor: pointer; margin-top: 1.25rem; }
  .got-it:hover { background: var(--border); }
  .game { width: 100%; max-width: 480px; padding: 1.5rem 1rem 3rem; }
  .level-tabs { display: flex; gap: 8px; margin-bottom: 1.5rem; }
  .ltab { flex: 1; padding: 10px 8px; border-radius: var(--radius); border: 1px solid var(--border); background: var(--surface); color: var(--text-muted); font-size: 13px; font-family: 'DM Sans', sans-serif; cursor: pointer; text-align: center; transition: all 0.15s; }
  .ltab:hover { background: var(--bg); }
  .ltab .lname { font-size: 13px; font-weight: 500; }
  .ltab .lword { font-size: 11px; margin-top: 2px; opacity: 0.7; }
  .ltab.active-easy { background: var(--amber-bg); border-color: var(--amber-border); color: var(--amber-text); }
  .ltab.active-medium { background: var(--blue-bg); border-color: var(--blue-border); color: var(--blue-text); }
  .ltab.active-hard { background: var(--pink-bg); border-color: var(--pink-border); color: var(--pink-text); }
  .bridge { background: var(--surface); border: 1px solid var(--border); border-radius: var(--radius-lg); padding: 1.25rem; margin-bottom: 1.25rem; text-align: center; }
  .blabel { font-size: 11px; color: var(--text-hint); letter-spacing: 0.08em; text-transform: uppercase; margin-bottom: 10px; }
  .bwords { display: flex; align-items: center; justify-content: center; gap: 14px; }
  .wbadge { background: var(--bg); border: 1px solid var(--border); border-radius: var(--radius); padding: 10px 20px; font-family: 'DM Mono', monospace; font-size: 22px; font-weight: 500; color: var(--text); letter-spacing: 0.1em; }
  .barrow { font-size: 20px; color: var(--text-hint); }
  .stats { display: flex; gap: 10px; margin-bottom: 1.25rem; }
  .stat { flex: 1; background: var(--surface); border: 1px solid var(--border); border-radius: var(--radius); padding: 12px; text-align: center; }
  .sv { font-size: 22px; font-weight: 500; color: var(--text); font-family: 'DM Mono', monospace; }
  .sl { font-size: 11px; color: var(--text-muted); margin-top: 3px; letter-spacing: 0.04em; }
  .chain-area { margin-bottom: 1rem; }
  .clabel { font-size: 11px; color: var(--text-hint); letter-spacing: 0.08em; text-transform: uppercase; margin-bottom: 10px; }
  .cdisp { display: flex; flex-wrap: wrap; gap: 6px; min-height: 40px; align-items: center; }
  .cw { display: flex; align-items: center; gap: 5px; }
  .cw .w { background: var(--blue-bg); border-radius: var(--radius); padding: 5px 11px; color: var(--blue-text); font-weight: 500; font-family: 'DM Mono', monospace; font-size: 14px; letter-spacing: 0.06em; }
  .cw.start .w { background: var(--amber-bg); color: var(--amber-text); }
  .cw.done .w { background: var(--green-bg); color: var(--green-text); }
  .cw .arr { color: var(--text-hint); font-size: 11px; }
  .irow { display: flex; gap: 8px; margin-bottom: 6px; }
  .irow input { flex: 1; padding: 11px 14px; font-size: 16px; border-radius: var(--radius); border: 1px solid var(--border); background: var(--surface); color: var(--text); font-family: 'DM Mono', monospace; letter-spacing: 0.1em; text-transform: uppercase; outline: none; transition: border-color 0.15s; }
  .irow input:focus { border-color: #91b8f5; box-shadow: 0 0 0 3px rgba(145,184,245,0.2); }
  .irow button { padding: 11px 18px; border-radius: var(--radius); border: 1px solid var(--border); background: var(--surface); color: var(--text); font-size: 14px; font-family: 'DM Sans', sans-serif; font-weight: 500; cursor: pointer; transition: background 0.1s; }
  .irow button:hover { background: var(--bg); }
  .hbar { font-size: 13px; color: var(--text-muted); min-height: 20px; margin-bottom: 8px; line-height: 1.5; }
  .hbar.err { color: var(--red-text); }
  .hbar.ok { color: var(--green-text); }
  .undo-btn { font-size: 12px; color: var(--text-hint); background: none; border: none; cursor: pointer; text-decoration: underline; display: block; text-align: right; margin-bottom: 12px; font-family: 'DM Sans', sans-serif; }
  .undo-btn:hover { color: var(--text-muted); }
  .rule { font-size: 12px; color: var(--text-hint); text-align: center; line-height: 1.7; }
  .winbox { background: var(--green-bg); border: 1px solid var(--green-border); border-radius: var(--radius-lg); padding: 1.5rem; text-align: center; margin-bottom: 1rem; }
  .wt { font-family: 'DM Serif Display', serif; font-size: 22px; color: var(--green-text); margin-bottom: 6px; }
  .ws { font-size: 14px; color: #2d7a35; margin-bottom: 1rem; }
  .win-chain { font-family: 'DM Mono', monospace; font-size: 13px; color: var(--green-text); margin-bottom: 1rem; line-height: 1.8; }
  .share-btn { display: inline-block; padding: 11px 24px; border-radius: var(--radius); border: 1px solid var(--green-border); background: var(--surface); color: var(--green-text); font-size: 14px; font-family: 'DM Sans', sans-serif; font-weight: 500; cursor: pointer; transition: background 0.1s; margin-bottom: 0.75rem; width: 100%; }
  .share-btn:hover { background: var(--bg); }
  .share-copied { font-size: 12px; color: var(--green-text); text-align: center; min-height: 18px; margin-bottom: 0.5rem; }
  .newbtn { width: 100%; padding: 13px; border-radius: var(--radius); border: 1px solid var(--border); background: var(--surface); color: var(--text); font-size: 15px; font-family: 'DM Sans', sans-serif; font-weight: 500; cursor: pointer; transition: background 0.1s; }
  .newbtn:hover { background: var(--bg); }
  footer { margin-top: auto; padding: 1.5rem; font-size: 12px; color: var(--text-hint); text-align: center; }
</style>
</head>
<body>

<div class="overlay" id="overlay" onclick="closeHelp(event)">
  <div class="modal">
    <button class="modal-close" onclick="closeHelpBtn()">x</button>
    <div class="modal-title">How to play</div>
    <div class="rule-block">
      <p>Connect the two words by changing one letter at a time. Every step must be a real word.</p>
      <div class="example">
        <span class="ex-word start">COLD</span>
        <span class="ex-arr">→</span>
        <span class="ex-word">CORD</span>
        <span class="ex-arr">→</span>
        <span class="ex-word">WORD</span>
        <span class="ex-arr">→</span>
        <span class="ex-word">WARD</span>
        <span class="ex-arr">→</span>
        <span class="ex-word end">WARM</span>
      </div>
      <p>Each step changes exactly one letter in the same position. Try to reach the target in as few steps as possible — that's your par!</p>
    </div>
    <div class="rule-block">
      <p>Choose your difficulty:</p>
      <div class="level-pills">
        <span class="pill easy">Easy — 3 letters</span>
        <span class="pill medium">Medium — 4 letters</span>
        <span class="pill hard">Hard — 5 letters</span>
      </div>
    </div>
    <div class="rule-block">
      <p>Stuck? Use the undo button to go back a step and try a different route.</p>
      <p>When you finish, copy your result and share it with friends!</p>
    </div>
    <button class="got-it" onclick="closeHelpBtn()">Got it — let's play!</button>
  </div>
</div>

<header>
  <span class="logo">Word Chain</span>
  <span class="logo-sub">Daily puzzle</span>
  <button class="help-btn" onclick="openHelp()">How to play</button>
</header>

<div class="game">
  <div class="level-tabs">
    <button class="ltab" id="tab-easy" onclick="setLevel('easy')"><div class="lname">Easy</div><div class="lword">3 letters</div></button>
    <button class="ltab" id="tab-medium" onclick="setLevel('medium')"><div class="lname">Medium</div><div class="lword">4 letters</div></button>
    <button class="ltab" id="tab-hard" onclick="setLevel('hard')"><div class="lname">Hard</div><div class="lword">5 letters</div></button>
  </div>
  <div class="bridge">
    <div class="blabel">Connect these words</div>
    <div class="bwords">
      <div class="wbadge" id="start-badge">CAT</div>
      <div class="barrow">→</div>
      <div class="wbadge" id="end-badge">DOG</div>
    </div>
  </div>
  <div class="stats">
    <div class="stat"><div class="sv" id="steps-v">0</div><div class="sl">steps</div></div>
    <div class="stat"><div class="sv" id="par-v">4</div><div class="sl">par</div></div>
    <div class="stat"><div class="sv" id="best-v">—</div><div class="sl">best</div></div>
  </div>
  <div class="chain-area">
    <div class="clabel">Your chain</div>
    <div class="cdisp" id="cdisp"></div>
  </div>
  <div id="game-input">
    <div class="irow">
      <input type="text" id="wi" placeholder="next word..." autocomplete="off" autocapitalize="characters" spellcheck="false" />
      <button onclick="submit()">Add</button>
    </div>
    <button class="undo-btn" onclick="undo()">undo last</button>
    <div class="hbar" id="hbar">Change one letter from <strong id="cur-hint">CAT</strong></div>
    <div class="rule" id="rule-txt">All 3-letter words · one letter changes per step</div>
  </div>
  <div id="win-area" style="display:none">
    <div class="winbox">
      <div class="wt" id="wt">Chain complete!</div>
      <div class="ws" id="ws"></div>
      <div class="win-chain" id="win-chain"></div>
      <button class="share-btn" onclick="shareResult()">Copy result to share</button>
      <div class="share-copied" id="share-copied"></div>
    </div>
    <button class="newbtn" onclick="nextPuzzle()">Next puzzle →</button>
  </div>
</div>

<footer>Word Chain &mdash; a daily word puzzle &mdash; wordchain-puzzle.com</footer>

<script>
const LEVELS={easy:{wordLen:3,puzzles:[{s:'CAT',e:'DOG',par:4},{s:'HOT',e:'ICE',par:4},{s:'SUN',e:'DIM',par:4},{s:'BAT',e:'FLY',par:4},{s:'MAN',e:'BOY',par:3},{s:'CUP',e:'POT',par:3},{s:'HAT',e:'WIG',par:4},{s:'BIG',e:'FAT',par:4},{s:'WET',e:'DRY',par:4},{s:'OLD',e:'NEW',par:4}],words:new Set(['CAT','BAT','BIT','BOT','BOG','DOG','HOT','HOG','HIT','HID','AID','ICE','SUN','GUN','GUT','CUP','CUT','CAP','MAP','MAT','MAD','BAD','BAG','FAT','FLY','FIT','SIT','SAT','HAT','HAD','HAS','WAS','WIG','WIN','TIN','TAN','MAN','MEN','HEN','HEM','HAM','JAM','JAR','BAR','BAN','BOY','COY','COT','POT','POP','MOP','MOB','MOD','SOW','SOB','SOD','SOT','SET','NET','MET','WET','BET','BED','RED','ROD','ROT','RAT','RAN','RAM','DIM','DIP','TIP','TOP','TON','TOT','TOE','FOE','FOG','LOG','LOT','LOW','BOW','ROW','ROB','RUB','RUT','NUT','NOT','NEW','SEW','SEA','PEA','PEG','LEG','LET','LED','FED','FEW','DEW','DEN','PEN','PIN','PIE','TIE','DIE','LIE','LIP','SIP','ZIP','ZAP','LAP','LAD','LAG','NAG','NAP','SAP','SAG','SAW','PAW','LAW','WAX','WAR','FAN','PAN','PAD','OLD','ODD','ANT','APE','AGE','ACE','OWL','GEL','GAP','GAB','CAB','CAD','CAN','COB','COD','COG','COP','COT','COW','CUB','TAB','TAD','TAG','TAP','TAR','TON','TAX','YAM','YAP','YEW','YET','DRY','BIG','WIG','PIG','JIG','RIG','DIG','FIG','HOD','NOD','GOD','BOD','POD','ADO','AGO','ALE','ALL','ARC','ARE','ARK','ARM','ART','ASH','ASK','ATE','AWE','AXE','AYE','BEE','BOO','BUG','BUN','BUS','BUT','BUY','CAR','COO','CRY','CUE','DAB','DAD','DAM','DAY','DID','DOE','DUB','DUG','DUN','DUO','DYE','EAR','EAT','EEL','EGG','ELF','ELK','ELM','EMU','ERA','EVE','EWE','FAD','FAR','FAX','FAY','FEE','FEN','FEY','FIB','FIN','FIR','FIX','FLU','FOB','FRY','FUN','FUR','GAG','GAL','GAR','GAY','GEE','GEM','GET','GIN','GNU','GOB','GOT','GUM','GUY','GYM','HAG','HAP','HAW','HAY','HEP','HER','HEW','HEY','HIM','HIP','HIS','HOB','HOE','HOP','HOW','HUB','HUE','HUG','HUM','HUT','ICY','ILL','IMP','INK','INN','ION','IRE','ITS','IVY','JAB','JAG','JAW','JAY','JET','JIB','JOB','JOG','JOT','JOY','JUG','JUT','KEG','KID','KIT','LAB','LAX','LAY','LEA','LID','LIT','LOO','LOP','LUG','MAG','MAW','MAY','MEW','MID','MIX','MOO','MUD','MUG','NAB','NAY','NIB','NIT','NOB','NOG','NOR','NOW','NUN','OAF','OAK','OAR','OAT','ODE','OHM','OIL','OPT','ORB','ORE','OUR','OUT','OWE','OWN','PAL','PAP','PAR','PAT','PAX','PAY','PEE','PEP','PER','PEW','PLY','PUB','PUD','PUG','PUN','PUP','PUS','PUT','RAG','RAP','RAW','RAY','REB','REF','REP','RIB','RID','RIM','RIP','ROC','ROE','RUG','RUM','RUN','RYE','SAC','SAD','SEC','SEE','SEG','SHE','SHY','SIN','SIR','SIS','SKI','SKY','SLY','SON','SOP','SOY','SPA','SPY','STY','SUB','SUE','SUM','SUP','TAO','TAT','TAV','TAW','TEA','TEE','THO','THY','TIC','TOD','TOG','TOM','TOO','TOW','TOY','TRY','TUB','TUG','TUM','TUN','TWO','UGH','UMP','URN','USE','VAN','VAR','VAT','VIA','VIE','VOW','WAD','WAG','WAN','WAT','WAW','WAY','WEB','WED','WEE','WEN','WIS','WIT','WOE','WOK','WON','WOO','WOP','WOT','WOW','WRY','YAK','YAW','YEA','YEN','YEP','YES','YIN','YOB','YOU','YUK','YUM','ZAG','ZAX','ZED','ZEE','ZEN','ZIG','ZIT','ZOO'])},medium:{wordLen:4,puzzles:[{s:'COLD',e:'WARM',par:4},{s:'CATS',e:'DOGS',par:5},{s:'HAND',e:'FOOT',par:5},{s:'LOVE',e:'HATE',par:4},{s:'HEAD',e:'TAIL',par:5},{s:'BOOK',e:'READ',par:4},{s:'DARK',e:'GLOW',par:5},{s:'SALT',e:'LIME',par:4},{s:'FIRE',e:'COOL',par:4},{s:'BLUE',e:'PINK',par:5},{s:'MIND',e:'BODY',par:5},{s:'RAIN',e:'SNOW',par:5}],words:new Set(['ABLE','ACID','AGED','AIDE','AIMS','AKIN','ALTO','AMID','ANTE','ANTI','ANTS','APEX','ARCH','AREA','ARID','ARMY','ARTS','ASKS','ATOM','AUNT','AURA','AUTO','AVID','AXES','AXLE','BABE','BACK','BAIL','BAIT','BAKE','BALD','BALE','BALL','BALM','BAND','BANE','BANG','BANK','BARD','BARE','BARK','BARN','BASE','BASH','BASS','BATH','BATS','BAWL','BEAD','BEAK','BEAM','BEAN','BEAR','BEAT','BEEF','BEEN','BEER','BELT','BEND','BENT','BEST','BILE','BILL','BIND','BIRD','BITE','BLOW','BLUE','BLUR','BOAT','BODY','BOLD','BOLT','BOND','BONE','BOOK','BOOM','BOOT','BORE','BORN','BOWL','BREW','BRIM','BROW','BURN','CAGE','CAKE','CALL','CALM','CAME','CAMP','CANE','CAPE','CARD','CARE','CARP','CART','CASE','CASH','CAST','CATS','CAVE','CELL','CENT','CHEF','CHIP','CLAN','CLAP','CLAY','CLIP','COIL','COLD','COME','CONE','COOK','COOL','COPE','CORD','CORE','CORN','COST','COVE','CREW','CROP','CROW','CURE','CURL','CUTS','DAME','DAMP','DARE','DARK','DART','DATE','DAWN','DEAD','DEAL','DEAR','DECK','DEEP','DEER','DEFT','DENT','DESK','DIME','DINE','DIRE','DIRT','DISK','DOCK','DOGS','DOME','DONE','DOOR','DOSE','DOVE','DOWN','DRAG','DRAW','DRIP','DROP','DRUG','DRUM','DUCK','DUEL','DULL','DUNE','DUSK','DUST','EACH','EARL','EARN','EASE','EAST','EASY','EDGE','EMIT','EPIC','EVEN','EVIL','FACE','FADE','FAIL','FAIR','FAKE','FALL','FAME','FARE','FARM','FAST','FATE','FEAR','FEAT','FEED','FEEL','FEET','FELL','FELT','FEND','FERN','FILE','FILL','FILM','FIND','FINE','FIRE','FIRM','FISH','FIST','FLAG','FLAP','FLAT','FLEW','FLIP','FLOW','FOAM','FOLD','FOND','FONT','FOOL','FOOT','FORD','FORK','FORM','FORT','FOUL','FOUR','FREE','FUEL','FULL','FUND','FUSE','FUSS','GALE','GAME','GATE','GAVE','GAZE','GEAR','GILD','GIVE','GLAD','GLEE','GLOW','GLUE','GOAT','GOES','GOLD','GOLF','GOOD','GORE','GOWN','GRAB','GREW','GRIP','GRIT','GROW','GULF','GUST','HAIL','HAIR','HALF','HALL','HALO','HALT','HAND','HARD','HARE','HARM','HAZE','HAZY','HEAL','HEAP','HEAT','HEEL','HELD','HELM','HELP','HERD','HERO','HIGH','HILL','HINT','HIRE','HIVE','HOLE','HOME','HOOK','HOPE','HORN','HOST','HOUR','HOWL','HULL','HUNG','HUNT','HURL','HURT','IDLE','IRIS','ISLE','ITEM','JACK','JADE','JAIL','JAZZ','JERK','JEST','JOLT','JUST','KEEN','KEEP','KEYS','KICK','KILL','KIND','KING','KNEW','KNOW','LACE','LACK','LAID','LAKE','LAMB','LAMP','LAND','LANE','LARD','LARK','LASH','LAST','LATE','LAVA','LAWN','LAZY','LEAD','LEAF','LEAN','LEAP','LEND','LENS','LICE','LICK','LIFT','LIKE','LIME','LIMP','LINE','LINK','LINT','LION','LIST','LIVE','LOAD','LOAF','LOAN','LOCK','LOFT','LONG','LOOK','LOOM','LOOP','LOST','LOUD','LOVE','LUCK','LULL','LUMP','LUNG','LURE','LURK','MACE','MADE','MAIL','MAIN','MAKE','MALE','MALL','MALT','MANE','MANY','MARE','MARK','MARS','MASH','MASK','MAST','MATE','MAZE','MEAL','MEAN','MEAT','MEET','MELT','MERE','MESH','MILD','MILE','MILL','MIND','MINE','MIST','MOAN','MOAT','MOLD','MOLE','MOOD','MOON','MOOR','MORE','MOST','MOTH','MOVE','MUCH','MULE','MUSE','MUSK','MUST','NAIL','NAPE','NAVY','NEAR','NEAT','NEED','NEST','NEWS','NICE','NODE','NONE','NOON','NORM','NOSE','NOTE','NUMB','OBEY','ODDS','OMEN','ONCE','ONLY','OPEN','OVEN','OVER','PACE','PAGE','PAIL','PAIN','PAIR','PALE','PALM','PARK','PART','PASS','PAST','PATH','PAVE','PAWN','PEAK','PEAR','PEEL','PEER','PELT','PERK','PEST','PICK','PILE','PILL','PINE','PIPE','PLAY','PLEA','PLOD','PLOT','PLOW','PLUM','PLUS','POEM','POET','POLE','POLL','POND','POOL','POOR','PORE','POSE','POST','POUR','PREY','PROP','PULL','PUMP','PUNK','PURE','PUSH','RACE','RACK','RAGE','RAID','RAIL','RAIN','RAKE','RAMP','RANG','RANK','RANT','RASH','RATE','READ','REAL','REAP','REIN','RELY','REND','RENT','REST','RICE','RICH','RIDE','RIFE','RIFT','RING','RIOT','RISE','RISK','ROAD','ROAM','ROAR','ROBE','ROCK','RODE','ROLE','ROLL','ROOF','ROOK','ROPE','ROSE','RUIN','RULE','RUNE','RUNG','RUSH','RUST','SAFE','SAGE','SAID','SAIL','SAKE','SALE','SALT','SAME','SAND','SANE','SANG','SANK','SAVE','SEAL','SEAM','SEAR','SEEM','SEEP','SELF','SEND','SHED','SHIN','SHIP','SHOP','SHOT','SHOW','SHUT','SICK','SIGH','SILK','SILL','SING','SINK','SIZE','SKIN','SKIP','SLAB','SLAP','SLIM','SLIP','SLOT','SLOW','SNAP','SNOW','SOAK','SOAP','SOAR','SOCK','SOFT','SOIL','SOLE','SOME','SONG','SOOT','SORE','SORT','SOUL','SOUP','SOUR','SPAR','SPIN','SPOT','SPUR','STAR','STEM','STEP','STEW','STIR','STOP','STUB','STUD','STUN','SUIT','SUNG','SUNK','SWIM','TAIL','TALE','TALK','TALL','TAME','TANK','TAPE','TART','TASK','TEAL','TEAM','TEAR','TELL','TEND','TENT','TEST','TICK','TIDE','TIER','TILE','TILL','TILT','TIME','TINT','TINY','TIRE','TOIL','TOLL','TOMB','TOME','TONE','TOOL','TORE','TORN','TOSS','TOTE','TRAP','TRIM','TRIO','TRIP','TRUE','TUBE','TUCK','TUNA','URGE','USED','VALE','VANE','VEIL','VEIN','VERB','VERY','VEST','VETO','VILE','VINE','VOID','VOLT','WADE','WAIL','WAIT','WAKE','WALK','WALL','WAND','WANE','WARD','WARM','WARP','WARY','WEED','WEEP','WELD','WELL','WELT','WENT','WEST','WIDE','WILL','WILT','WIND','WINE','WING','WINK','WIRE','WISE','WISH','WOLF','WORD','WORE','WORK','WORM','WREN','YARD','YAWN','YEAR','YELL','YELP','YOKE','ZEAL','ZERO','ZEST','ZINC','ZONE','ZOOM','CORD','BAGS','BEAD','BOLD','BALE','HAVE','HIVE','HIKE','BIKE','PINK','RINK','RING','LIME','LAME','CAME','CONE','BONE','BANE','LANE','CANE','PALE','HALE','TALL','CALL','FALL','GALL','WALL','VINE','VANE','TONE','LONE','LORE','GORE','WAKE','BAKE','CAKE','FAKE','LAKE','RAKE','SAKE','TAKE','WADE','FADE','JADE'])},hard:{wordLen:5,puzzles:[{s:'BLACK',e:'WHITE',par:6},{s:'BREAD',e:'TOAST',par:6},{s:'LIGHT',e:'SHADE',par:6},{s:'SHARP',e:'BLUNT',par:6},{s:'BRAVE',e:'TIMID',par:7},{s:'FLAME',e:'FROST',par:7},{s:'STEEL',e:'STONE',par:5},{s:'RIVER',e:'OCEAN',par:6},{s:'PLANT',e:'BLOOM',par:5},{s:'STORM',e:'CLEAR',par:7}],words:new Set(['ABOUT','ABOVE','ABUSE','ACTOR','ACUTE','ADMIT','ADOPT','ADULT','AFTER','AGAIN','AGILE','AGREE','AHEAD','ALARM','ALBUM','ALERT','ALIKE','ALIVE','ALLEY','ALLOW','ALONE','ALONG','ALTER','ANGEL','ANGER','ANGLE','ANGRY','ANNEX','APPLY','ARISE','ARSON','ASIDE','ATLAS','ATTIC','AVOID','AWAKE','AWARD','AWARE','AWFUL','BADLY','BAKER','BASIC','BASIN','BATCH','BEACH','BEARD','BEAST','BEGIN','BEING','BELOW','BENCH','BIBLE','BIRTH','BLADE','BLACK','BLAND','BLANK','BLAST','BLAZE','BLEED','BLEND','BLESS','BLIND','BLOCK','BLOOD','BLOWN','BLOOM','BOARD','BONUS','BOOST','BOOTH','BORED','BOUND','BRAIN','BRAND','BRAVE','BREAD','BREAK','BREED','BRICK','BRIDE','BRIEF','BRING','BROAD','BROKE','BROWN','BRUSH','BUILD','BUILT','BURST','BUYER','CABIN','CAMEL','CARRY','CATCH','CAUSE','CHAIN','CHAIR','CHALK','CHAOS','CHARM','CHART','CHASE','CHEAP','CHECK','CHESS','CHEST','CHIEF','CHILD','CHILL','CHOSE','CIVIC','CIVIL','CLAIM','CLAMP','CLASS','CLEAN','CLEAR','CLERK','CLICK','CLIFF','CLIMB','CLING','CLOCK','CLONE','CLOSE','CLOUD','CLOWN','COACH','COMET','COMIC','CORAL','COULD','COUNT','COVER','CRAFT','CRASH','CRAZY','CREAM','CREEK','CRIME','CRISP','CROSS','CROWD','CROWN','CRUEL','CRUSH','CURVE','DAILY','DANCE','DEATH','DEBUT','DECAY','DEPTH','DIGIT','DIRTY','DIZZY','DODGE','DOUBT','DOUGH','DRAFT','DRAIN','DRAMA','DREAM','DRIFT','DRILL','DRINK','DRIVE','DROVE','DROWN','DYING','EAGER','EAGLE','EARLY','EARTH','EIGHT','ELITE','EMPTY','ENEMY','ENJOY','ENTER','EQUAL','ERROR','ESSAY','EVENT','EVERY','EXACT','EXIST','EXTRA','FAINT','FAIRY','FAITH','FALSE','FANCY','FATAL','FAULT','FEAST','FENCE','FETCH','FEVER','FIBER','FIELD','FIFTH','FIFTY','FIGHT','FINAL','FIRST','FIXED','FLAME','FLARE','FLASH','FLEET','FLESH','FLINT','FLOCK','FLOOD','FLOOR','FLOUR','FOCUS','FORCE','FORGE','FORTH','FORUM','FOUND','FRAME','FRANK','FRAUD','FRESH','FRONT','FROST','FROZE','FRUIT','FULLY','FUNNY','GLARE','GLASS','GLAZE','GLOBE','GLOOM','GLORY','GLOSS','GLOVE','GOING','GRACE','GRADE','GRAIN','GRAND','GRANT','GRASP','GRASS','GRAVE','GREAT','GREEN','GRIEF','GROAN','GROOM','GROUP','GROVE','GROWN','GUARD','GUESS','GUEST','GUIDE','GUILD','HABIT','HARSH','HASTE','HATCH','HAVEN','HEART','HEAVY','HEDGE','HENCE','HERBS','HONOR','HORSE','HOTEL','HOUSE','HUMAN','HUMOR','IDEAL','IMAGE','IMPLY','INDEX','INNER','INPUT','ISSUE','IVORY','JEWEL','JUDGE','KNIFE','KNOCK','KNOWN','LABEL','LANCE','LARGE','LASER','LATCH','LATER','LAUGH','LAYER','LEARN','LEASE','LEAST','LEAVE','LEGAL','LEVEL','LIGHT','LIMIT','LIVER','LOCAL','LODGE','LOGIC','LOOSE','LOWER','LOYAL','LUCKY','MAGIC','MAJOR','MAKER','MANOR','MAPLE','MARCH','MATCH','MAYOR','MEDAL','MERCY','MERGE','MIGHT','MINOR','MINUS','MODEL','MONEY','MONTH','MORAL','MOUNT','MOUTH','MOVIE','MUSIC','NAIVE','NASTY','NERVE','NIGHT','NOBLE','NOISE','NORTH','NOVEL','NURSE','OCCUR','OCEAN','OFFER','OFTEN','ORDER','OUGHT','OUTER','OWNED','OWNER','OZONE','PAINT','PANEL','PANIC','PAPER','PARTY','PEACE','PEARL','PENNY','PERCH','PHASE','PHONE','PHOTO','PIANO','PIECE','PILOT','PITCH','PLACE','PLAIN','PLANE','PLANK','PLANT','PLATE','PLAZA','PLEAD','PLUCK','PLUMB','PLUME','POINT','POLAR','POUCH','POWER','PRESS','PRICE','PRIDE','PRIME','PRINT','PRIOR','PRIZE','PROBE','PRONE','PROOF','PROSE','PROUD','PROVE','PROXY','PULSE','PUNCH','PUPIL','QUEEN','QUERY','QUICK','QUIET','QUITE','QUOTA','QUOTE','RADAR','RAISE','RALLY','RANGE','RAPID','RATIO','REACH','REALM','REBEL','REIGN','RELAX','REPAY','RIDER','RIDGE','RIGHT','RIGID','RISKY','RIVAL','RIVER','ROBOT','ROCKY','ROUND','ROUTE','ROYAL','RULER','RURAL','SAINT','SANDY','SAUCE','SCALE','SCARE','SCENE','SCOPE','SCORE','SCOUT','SEIZE','SENSE','SERVE','SEVEN','SHAFT','SHALL','SHAME','SHAPE','SHARE','SHARK','SHARP','SHIFT','SHINE','SHIRT','SHOCK','SHORT','SHOUT','SIGHT','SILLY','SINCE','SIXTH','SIXTY','SKILL','SKULL','SLASH','SLATE','SLAVE','SLEEP','SLEET','SLICE','SLICK','SLIDE','SLOPE','SMART','SMELL','SMILE','SMOKE','SOLID','SOLVE','SORRY','SOUTH','SPACE','SPARK','SPAWN','SPEAK','SPELL','SPEND','SPICE','SPINE','SPITE','SPLIT','SPOKE','SPOON','SPORT','SPRAY','SQUAD','STACK','STAFF','STAGE','STAIN','STAKE','STALE','STALL','STAMP','STAND','START','STATE','STEAM','STEEL','STEEP','STEER','STICK','STIFF','STILL','STOCK','STONE','STOOD','STORE','STORM','STORY','STOVE','STRAP','STRAW','STRAY','STRIP','STUCK','STUDY','STYLE','SUGAR','SUITE','SUNNY','SUPER','SURGE','SWAMP','SWEAR','SWEAT','SWEEP','SWELL','SWEPT','SWIFT','SWIRL','SWORD','TABLE','TASTE','TEACH','THANK','THEME','THERE','THESE','THICK','THING','THINK','THORN','THOSE','THREE','THREW','THROW','TIGER','TIMID','TIRED','TITLE','TODAY','TOKEN','TORCH','TOTAL','TOUCH','TOUGH','TOWEL','TOWER','TRACE','TRACK','TRAIL','TRAIN','TRASH','TREAD','TREAT','TREND','TRIAL','TRIBE','TRICK','TRIED','TROOP','TRUCK','TRULY','TRUNK','TRUST','TRUTH','TULIP','TWICE','TWIST','ULTRA','UNDER','UNION','UNTIL','UPPER','UPSET','URBAN','USAGE','USUAL','UTTER','VALID','VALUE','VALVE','VAPOR','VAULT','VERSE','VIDEO','VIGOR','VIRAL','VIRUS','VISIT','VISTA','VITAL','VIVID','VOICE','VOTER','WALTZ','WASTE','WATCH','WATER','WEARY','WEAVE','WEIGH','WEIRD','WHALE','WHEAT','WHEEL','WHERE','WHICH','WHILE','WHIRL','WHISK','WHOLE','WHOSE','WIDTH','WITCH','WORLD','WORRY','WORSE','WORST','WORTH','WOULD','WOUND','WRATH','WRIST','WRITE','WROTE','YACHT','YIELD','YOUNG','YOUTH','ZEBRA','SHADE','SHALE','SHAVE','BLUNT','SLANT','PLANE','BLANK','STAND','STARE','SHARE','SHORE','BLADE','BLAZE','GLAZE','GRACE','GRAZE','CRAVE','BRACE','TRACE','PLATE','SLATE','FLAME','BLAME','STEAD','STEED','SLEEP','FLEET','FLESH','PRESS','DRESS','CROSS','GROSS','GROAN','GRAIN','BRAIN','BLEED','BREED','GREED','SHEET','SWEET','SWEAT','WHEAT','CHEAT','CLEAT','TOAST'])}};
let level='easy',puzzleIdx={easy:0,medium:0,hard:0},chain=[],bestScores={easy:{},medium:{},hard:{}};
function diff(a,b){if(a.length!==b.length)return -1;let d=0;for(let i=0;i<a.length;i++)if(a[i]!==b[i])d++;return d;}
function isValid(w){const pz=LEVELS[level].puzzles[puzzleIdx[level]];return w===pz.s||w===pz.e||LEVELS[level].words.has(w);}
function render(){const pz=LEVELS[level].puzzles[puzzleIdx[level]];const el=document.getElementById('cdisp');el.innerHTML='';chain.forEach((w,i)=>{const d=document.createElement('div');d.className='cw'+(i===0?' start':'')+(w===pz.e?' done':'');d.innerHTML='<span class="w">'+w+'</span>'+(i<chain.length-1?'<span class="arr">→</span>':'');el.appendChild(d);});document.getElementById('steps-v').textContent=chain.length>1?chain.length-1:0;}
function submit(){const inp=document.getElementById('wi');const v=inp.value.trim().toUpperCase();inp.value='';const L=LEVELS[level];const pz=L.puzzles[puzzleIdx[level]];const hbar=document.getElementById('hbar');if(!v)return;if(v.length!==L.wordLen){hbar.textContent='Must be exactly '+L.wordLen+' letters';hbar.className='hbar err';return;}const cur=chain[chain.length-1];const d=diff(v,cur);if(d!==1){hbar.textContent=d===0?'Same word — change one letter':'Change exactly one letter';hbar.className='hbar err';return;}if(chain.includes(v)){hbar.textContent='Already used that word';hbar.className='hbar err';return;}if(!isValid(v)){hbar.textContent='"'+v+'" isn\'t in our word list';hbar.className='hbar err';return;}chain.push(v);render();if(v===pz.e){const steps=chain.length-1;const key=pz.s+'-'+pz.e;if(!bestScores[level][key]||steps<bestScores[level][key])bestScores[level][key]=steps;updateBest();showWin(steps,pz.par);return;}document.getElementById('cur-hint').textContent=v;hbar.textContent='Good! From '+v+' — change one letter';hbar.className='hbar ok';inp.focus();}
function undo(){if(chain.length<=1)return;chain.pop();render();const cur=chain[chain.length-1];document.getElementById('cur-hint').textContent=cur;document.getElementById('hbar').textContent='Back to '+cur+' — change one letter';document.getElementById('hbar').className='hbar';}
function showWin(steps,par){document.getElementById('game-input').style.display='none';document.getElementById('win-area').style.display='block';let msg='';if(steps<=par-2)msg='Brilliant! '+steps+' steps — way under par ('+par+')';else if(steps<=par)msg='Solved in '+steps+' steps — right on par!';else msg='Solved in '+steps+' steps (par was '+par+') — try again to beat it!';document.getElementById('wt').textContent='Chain complete!';document.getElementById('ws').textContent=msg;document.getElementById('win-chain').textContent=chain.join(' → ');document.getElementById('share-copied').textContent='';}
function shareResult(){const pz=LEVELS[level].puzzles[puzzleIdx[level]];const steps=chain.length-1;const par=pz.par;let rating='';if(steps<=par-2)rating='Brilliant!';else if(steps<=par)rating='Right on par!';else rating='Keep practising!';const lvl=level.charAt(0).toUpperCase()+level.slice(1);const text='Word Chain\n'+lvl+': '+pz.s+' → '+pz.e+'\n'+'Solved in '+steps+' steps (par '+par+')\n'+chain.join(' → ')+'\n'+rating+'\n'+'Play at wordchain-puzzle.com';navigator.clipboard.writeText(text).then(()=>{document.getElementById('share-copied').textContent='Copied! Ready to paste and share.';}).catch(()=>{document.getElementById('share-copied').textContent='Copy failed — try selecting and copying the chain above manually.';});}
function nextPuzzle(){const L=LEVELS[level];puzzleIdx[level]=(puzzleIdx[level]+1)%L.puzzles.length;loadPuzzle();}
function updateBest(){const pz=LEVELS[level].puzzles[puzzleIdx[level]];const key=pz.s+'-'+pz.e;const b=bestScores[level][key];document.getElementById('best-v').textContent=b||'—';}
function setLevel(l){level=l;['easy','medium','hard'].forEach(x=>{document.getElementById('tab-'+x).className='ltab'+(x===l?' active-'+l:'');});loadPuzzle();}
function loadPuzzle(){const L=LEVELS[level];const pz=L.puzzles[puzzleIdx[level]];chain=[pz.s];document.getElementById('start-badge').textContent=pz.s;document.getElementById('end-badge').textContent=pz.e;document.getElementById('par-v').textContent=pz.par;document.getElementById('steps-v').textContent=0;document.getElementById('wi').maxLength=L.wordLen;document.getElementById('wi').placeholder=L.wordLen+'-letter word...';document.getElementById('rule-txt').textContent='All '+L.wordLen+'-letter words · one letter changes per step';document.getElementById('hbar').textContent='Change one letter from '+pz.s;document.getElementById('hbar').className='hbar';document.getElementById('cur-hint').textContent=pz.s;document.getElementById('game-input').style.display='block';document.getElementById('win-area').style.display='none';updateBest();render();document.getElementById('wi').focus();}
function openHelp(){document.getElementById('overlay').classList.add('open');}
function closeHelpBtn(){document.getElementById('overlay').classList.remove('open');}
function closeHelp(e){if(e.target===document.getElementById('overlay'))closeHelpBtn();}
window.addEventListener('DOMContentLoaded',function(){document.getElementById('wi').addEventListener('keydown',e=>{if(e.key==='Enter')submit();});setLevel('easy');});
</script>
</body>
</html>
