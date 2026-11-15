# BUBBLE SHOOTER – 30 LEVEL HTML5 GAME (YOUTUBE PLAYABLES EDITION)

A production-quality, responsive HTML5 & Phaser 3 casual arcade **Bubble Shooter** game featuring 30 individually crafted levels, 3D glossy bubble graphics, staggered hexagonal grid physics, local synthesized audio effects, and full compliance with Google's official **YouTube Playables SDK** (`ytgame`) specification.

---

## 1. Project Structure

```text
bubble-shooter/
│
├── index.html                  # Root HTML5 entry point (<script src="https://www.youtube.com/game_api/v1"></script> first)
├── style.css                   # Responsive viewport & touch-action CSS
│
├── lib/
│   └── phaser.min.js           # Locally bundled Phaser 3.80.1 framework
│
├── js/
│   ├── main.js                 # DOM bootstrap, gesture lock & ytgame.health error telemetry hook
│   ├── config.js               # Global constants, color palettes, scoring & localization dictionary (I18N)
│   ├── game.js                 # Phaser.Scale.RESIZE orchestrator, orientation & global pause/resume handlers
│   ├── bubble.js               # Bubble entity & object pooling (`BubblePool`)
│   ├── bubble-grid.js          # Staggered hexagonal grid math, BFS match-3+ & floating cluster detection
│   ├── shooter.js              # Turret aiming, dotted trajectory, next bubble preview & recoil
│   ├── collision.js            # Sub-stepped continuous wall/bubble collision & trajectory raycasting
│   ├── level-manager.js        # 30-level progression & row string parser
│   ├── score-manager.js        # Cluster scoring, combo multipliers, shot bonuses & 1-3 star calculation
│   ├── save-manager.js         # Exclusive YouTube Cloud Save in Playables + localStorage fallback for local dev
│   ├── audio-manager.js        # YouTube master audio synchronization + subordinate in-game SFX toggle
│   ├── ui-manager.js           # Glossy arcade buttons, particle bursts, falling bubbles & score popups
│   └── playable-adapter.js     # Official YouTube Playables SDK (`ytgame`) integration module
│
├── scenes/
│   ├── boot-scene.js           # Calls firstFrameReady(), loads audio, fetches language & awaits cloud loadData()
│   ├── menu-scene.js           # Main Menu, localized labels, SFX toggle & calls gameReady()
│   ├── level-select-scene.js   # 30-Level selection grid with sequential unlocking & star badges
│   ├── game-scene.js           # Core gameplay screen, HUD, responsive resize & system Pause/Resume handling
│   └── result-scene.js         # Level Complete (1-3 stars, victory particles) & Game Over screens
│
├── data/
│   └── levels.js               # 30 individually designed levels across 6 progressive difficulty tiers
│
├── assets/
│   ├── images/
│   │   └── logo-badge.svg      # Vector decorative badge
│   ├── audio/
│   │   ├── shoot.wav           # Bubble launch whoosh
│   │   ├── bounce.wav          # Wall reflection bounce
│   │   ├── pop.wav             # Cluster pop sound
│   │   ├── multi_pop.wav       # Large cluster / floating cascade chord
│   │   ├── win.wav             # Level complete fanfare
│   │   ├── gameover.wav        # Game over progression
│   │   └── click.wav           # UI button press
│   └── fonts/
│       └── arcade-fonts.css    # Local typography stack
│
└── README.md                   # Setup, testing, bundling & YouTube Playables SDK integration documentation
```

---

## 2. Official YouTube Playables SDK Integration Details

### A. SDK Initialization & Lifecycle ([`index.html`](index.html), [`js/playable-adapter.js`](js/playable-adapter.js))
1. **SDK Script Loading Order:** `<script src="https://www.youtube.com/game_api/v1"></script>` is loaded at the very top of `<head>` in `index.html` before Phaser 3 and all game scripts.
2. **Global Object Integrity:** The codebase never overrides or reassigns `window.ytgame` / `globalThis.ytgame`.
3. **Environment Detection:** Checked strictly via `Boolean(window.ytgame && window.ytgame.IN_PLAYABLES_ENV === true)`.
4. **Readiness Lifecycle:**
   - `ytgame.game.firstFrameReady()` is invoked once in `BootScene.preload()` ([`scenes/boot-scene.js`](scenes/boot-scene.js)) immediately after the first visual frame is drawn.
   - `ytgame.game.gameReady()` is invoked once in `MenuScene.create()` ([`scenes/menu-scene.js`](scenes/menu-scene.js)) only after all textures, audio assets, language preference, and cloud save data have loaded and the UI is interactive.
   - `PlayableAdapter` enforces that `firstFrameReady()` always precedes `gameReady()` and prevents duplicate calls across scene restarts.

### B. Cloud Save – Mandatory Exclusivity ([`js/save-manager.js`](js/save-manager.js))
- **Exclusive Cloud Save in Playables:** When `ytgame.IN_PLAYABLES_ENV` is `true`, `SaveManager` uses `ytgame.game.loadData()` and `ytgame.game.saveData(dataString)` **exclusively**. It never reads or writes `localStorage`, cookies, or IndexedDB inside the Playables environment.
- **Load-Before-Save Guarantee:** `SaveManager.init()` awaits `ytgame.game.loadData()` during `BootScene` before any gameplay or save operation can occur, and `PlayableAdapter.saveCloudData()` also guards that `_cloudLoadCompleted` is `true` before calling `ytgame.game.saveData()`.
- **Milestones & Pause Persistence:** Progress is saved as a validated JSON string (`< 3 MiB`, ~1.2 KB actual size) upon level completion, new high score, SFX preference change, and automatically whenever `ytgame.system.onPause` is triggered.
- **Backward Compatibility & Corruption Recovery:** `_migrateAndMergeState()` upgrades older v0/v1 save schemas to v2, safely recovers from empty or malformed JSON strings, and strips any disallowed fields (such as `language`/`locale`).
- **Local Development Fallback:** When running outside YouTube Playables (`IN_PLAYABLES_ENV` is falsy), `localStorage` remains active for local testing.

### C. Audio Integration ([`js/audio-manager.js`](js/audio-manager.js))
- **Platform Master Audio Gate:** `AudioManager` initializes `ytAudioEnabled` from `ytgame.system.isAudioEnabled()` and subscribes to `ytgame.system.onAudioEnabledChange(callback)`.
- **Strict Silence When Disabled:** `canPlayAudio()` requires `ytAudioEnabled && userSfxEnabled && !systemPaused`. When YouTube audio is disabled, Phaser's sound manager is muted/paused and the Web Audio synthesizer context is suspended so zero audio can play.
- **Subordinate In-Game SFX Toggle:** The in-game `SFX: ON / OFF` button controls only the player's local sound-effect preference (`userSfxEnabled`) and never overrides YouTube's master mute control. When YouTube audio is muted by the host, the button displays `YOUTUBE AUDIO: MUTED`.

### D. Pause & Resume Lifecycle ([`js/playable-adapter.js`](js/playable-adapter.js), [`scenes/game-scene.js`](scenes/game-scene.js), [`js/game.js`](js/game.js))
- **Authoritative SDK Callbacks:** Subscribes to `ytgame.system.onPause(callback)` and `ytgame.system.onResume(callback)`. Phaser's `sound.pauseOnBlur` is set to `false` and the browser Page Visibility API is not used as a substitute.
- **On Pause:** Halts the gameplay update loop (`this.isPaused = true`), pauses all active Phaser tweens (`this.tweens.pauseAll()`) and timers (`this.time.paused = true`), stops all sound/Web Audio output, blocks player input, and saves current progress to YouTube Cloud Save.
- **On Resume:** Restores timers, animations, and input, re-queries `ytgame.system.isAudioEnabled()`, and restores audio state only if YouTube audio is enabled.

### E. Score Integration ([`js/save-manager.js`](js/save-manager.js), [`js/playable-adapter.js`](js/playable-adapter.js))
- Calls `ytgame.engagement.sendScore({ value: score })` with a guaranteed integer value.
- `_reconcileBestScore()` ensures the score submitted to YouTube always matches `this.state.bestScore` persisted in `ytgame.game.saveData()`.

### F. Recommended Integrations & Ads Compatibility
- **Localization (`ytgame.system.getLanguage()`):** Fetched asynchronously during boot in `PlayableAdapter.fetchLanguage()` and mapped to `Config.I18N` (`en`, `es`, `fr`, `de`, `pt`, `ja`). The language tag is never stored in cloud save data.
- **Error Telemetry (`ytgame.health.logError()` & `ytgame.health.logWarning()`):** Integrated with rate-limiting in `PlayableAdapter` and wired to global `error`/`unhandledrejection` handlers and SDK catch blocks.
- **Ads Compatibility (`ytgame.ads`):** `requestInterstitialAd()` and `requestRewardedAd(rewardId)` wrappers are available on `PlayableAdapter` without any automatic ad triggers.

---

## 3. How to Test Locally & Generate the Production ZIP

### Local HTTP Server
```bash
cd bubble-shooter
npx serve . -l 8080
```
Open `http://localhost:8080` in your browser.

### Generate Production ZIP Bundle (`index.html` at Root)
```powershell
$ProgressPreference = 'SilentlyContinue'
Set-Location "C:\Users\thenu\.gemini\antigravity\scratch\bubble-shooter"
Compress-Archive -Path index.html, style.css, lib, js, scenes, data, assets, README.md -DestinationPath bubble-shooter-playable.zip -Force
```
