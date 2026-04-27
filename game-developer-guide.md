# Developer guide

Everything you need to ship a game that fits the platform. If you're instructing an AI assistant to build a game, point it at the URLs below — together they cover what to build, how to wire up platform features, and what to watch out for:

- `play.nitzan.games/build` — this page (file layout, display target, SDK reference)
- `github.com/nitzangames/play-nitzan-games-docs/blob/main/game-developer-guide.md` — same content as plain markdown for AI tooling that can't fetch the live site
- `play.nitzan.games/play-sdk.js` — the canonical SDK source
- `play.nitzan.games/game-dev-notes.md` — real-world lessons: pointer events in sandboxed iframes, canvas setup gotchas, GC discipline, mobile perf, thumbnail rules

## 1. File layout

Your zip must contain at root:

```
my-game.zip
  ├── index.html        ← entry point (required, exact filename)
  ├── meta.json         ← game metadata (required, exact filename)
  ├── thumbnail.png     ← 1024×1024 game icon (required, name set in meta.json)
  ├── game.js           ← your game code
  └── (any other files)
```

Limits: zip up to 50 MB compressed and 50 MB uncompressed. No path traversal (`..`) or absolute paths in entry names.

## 2. meta.json

```json
{
  "slug": "my-game",
  "title": "My Game",
  "description": "A short description of your game.",
  "tags": ["arcade", "puzzle"],
  "author": "Your Name",
  "thumbnail": "thumbnail.png"
}
```

- **slug** — URL-safe identifier (lowercase letters, digits, hyphens only). Becomes the URL: `play.nitzan.games/play/my-game`. Once a slug is yours, only you can redeploy it.
- **title** — display name shown on the platform.
- **author** — the platform shows your account display name (set in the user menu) when present, falling back to this field.
- **thumbnail** — must be a 1024×1024 PNG with the game title visible in the image. Filename must match this field.

## 3. Display target

Games render in a **1080×1920 portrait** iframe scaled to fit the player's viewport. Author your canvas / layout to that aspect ratio. The iframe sandbox is `allow-scripts allow-pointer-lock allow-same-origin` — no top-frame navigation or popups.

Use `window.innerWidth` / `window.innerHeight` to size your canvas dynamically (the parent scales the whole iframe, so the inner viewport is always your authored size). Test on both desktop (mouse) and mobile (touch).

## 4. Using the PlaySDK

The PlaySDK is **automatically injected** into your `index.html` at deploy time as an inline `<script>`. After deploy, `window.PlaySDK` is available before your other scripts run. You don't need to import, download, or copy it for production.

For **local development** or **AI tooling**, the canonical SDK source is served at:

```
https://play.nitzan.games/play-sdk.js
```

Add this to your `index.html` while developing locally:

```
<script src="https://play.nitzan.games/play-sdk.js"></script>
```

The deploy injector detects this reference and skips its own injection — safe in production. AI assistants can fetch the URL above to learn the exact API surface.

## 5. PlaySDK reference

### Save & load (cloud-synced)

```
await PlaySDK.save("highscore", "1000");
const score = await PlaySDK.load("highscore");
// Saves locally + syncs to cloud when the user is signed in.
```

### Leaderboards

```
// Submit a score
await PlaySDK.submitScore("level-1", 1000, "desc", {});
// "desc" = higher is better, "asc" = lower is better (e.g. fastest time).
// Last arg is metadata (any JSON-serializable object).

// Read top 10
const board = await PlaySDK.getLeaderboard("level-1", 10);
// board.entries = [{ rank, value, metadata, isMe }, ...]
```

### Real-time multiplayer

```
// Built-in lobby (create / join rooms)
PlaySDK.multiplayer.showLobby({
  maxPlayers: 4,
  onStart: () => startMultiplayerGame(),
  onCancel: () => showMainMenu(),
});

// Or use the raw API
const room = await PlaySDK.multiplayer.createRoom({ maxPlayers: 2 });
room.send({ x: 100, y: 200 });
PlaySDK.multiplayer.on("game", (from, data) => { /* ... */ });
```

### Turn-based multiplayer

```
const match = await PlaySDK.turnBased.createMatch({
  maxPlayers: 2,
  initialState: { board: Array(9).fill(null) },
});
// Share match.inviteCode with a friend.

await PlaySDK.turnBased.submitMove(match.id, {
  state: { board: ["X", null /* ... */] },
});
```

### NBucks (virtual currency)

Spend NBucks from inside your game. The user is prompted to confirm the spend (and to top up if their balance is low). On success the call resolves with the new balance; on cancel or insufficient funds it rejects.

```
// Spend 5 NBucks for an in-game item.
try {
  const { balance } = await PlaySDK.spendNbucks({
    amount: 5,
    itemDescription: "Extra life",
    itemId: "extra-life",   // optional, your own ID for analytics
  });
  grantExtraLife();
} catch (err) {
  // User cancelled or had insufficient funds and didn't top up.
}
```

### Rewarded video ads

```
// Check if ads are available (true on mobile app, false on web).
if (PlaySDK.adsAvailable) {
  showWatchAdButton();
}

// Show a rewarded video ad.
const result = await PlaySDK.showRewardedAd();
if (result.rewarded) {
  // User watched the full ad — grant the reward.
  grantExtraLife();
}
// On web: rewards are granted for free (no ad shown).
// On mobile: uses AdMob rewarded video.
```

### Pause / resume (battery)

```
PlaySDK.onPause(() => {
  cancelAnimationFrame(rafId);
  audioCtx?.suspend();
});
PlaySDK.onResume(() => {
  rafId = requestAnimationFrame(gameLoop);
  audioCtx?.resume();
});
```

### Haptic feedback

```
// Impact feedback
PlaySDK.haptic("light");   // gentle tap
PlaySDK.haptic("medium");  // default
PlaySDK.haptic("heavy");   // strong thud

// Notification feedback
PlaySDK.haptic("success"); // e.g. level complete
PlaySDK.haptic("warning"); // e.g. low health
PlaySDK.haptic("error");   // e.g. game over
// Native haptics on iOS, vibration on Android, no-op on desktop.
```

### Display name

```
const name = await PlaySDK.getDisplayName();
// Returns the user's chosen name, or null if not signed in / not set.
```

### Screenshot mode

```
// Auto-start gameplay for App Store screenshots.
if (window.PlaySDK && PlaySDK.screenshotMode) {
  startGame(); // skip menus, go straight to gameplay
}
```

## 6. Deployment

**Option A — Game Creator:** open [/create](/create), build with AI, click Deploy.

**Option B — Upload page:** zip your folder, drop it at [/upload](/upload) after signing in with Google or Apple. Goes live at `play.nitzan.games/play/your-slug` and appears on the homepage once an admin approves it.

**Option C — CLI deploy with deploy key** (for trusted developers, auto-approved):

```
cd my-game
zip -r /tmp/game.zip .
curl -X POST https://play.nitzan.games/api/deploy \
  -H "Authorization: Bearer YOUR_DEPLOY_KEY" \
  -F "file=@/tmp/game.zip"
```

## 7. Guidelines

- Author for 1080×1920 portrait; size your canvas dynamically.
- Target both desktop (keyboard / mouse) and mobile (touch).
- Use a canvas-based approach for best performance.
- Thumbnail must be 1024×1024 PNG with the game title visible.
- Use `PlaySDK.onPause` / `onResume` to save battery on mobile.
- No external ads or tracking — use `PlaySDK.showRewardedAd()` for optional rewarded videos.
- Follow the [Content Policy](/content-policy): no gambling, sexual content, hate speech, or copyrighted material you don't own.

## Questions?

Contact [contact@nitzan.games](mailto:contact@nitzan.games) for a deploy key or any other support.
