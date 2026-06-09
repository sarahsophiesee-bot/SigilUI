# Sigil UI

A key‑verification gate for Roblox script hubs, shaped like a **sealed access pass**. Same engine as [Onyx](./README_OnyxUI.md), a completely different idea of what a key screen looks like.

A key is treated as admission: a **dark card‑stock pass** on a dimmed backdrop, split by a **perforation seam** with two punched notches, an eyebrow‑labelled serial field, and a **wax‑seal medallion** in the stub that *stamps shut* — ring and check — the moment your key is accepted. One restrained accent (wax burgundy); everything else is warm ink on dark stock.

---

## Features

- **Drop‑in key gate** — set an `OnVerify` callback, call `Launch()`, run your script in `OnSuccess`.
- **Junkie / [jnkie.com](https://jnkie.com) support** — `LaunchJunkie{...}` loads the SDK, wires `check_key`, pulls the key link, and auto‑detects keyless.
- **Keyless mode** — force it, disable it, or let Junkie decide.
- **Saved keys** — remembered and re‑validated automatically on the next run (executor filesystem permitting).
- **Abuse handling** — escalating cooldown after repeated failures, HWID‑ban auto‑kick, transient/network errors retried without penalty.
- **Embeddable shop / promo strip** — optional bottom strip with a Buy button that copies your link.
- **PC + mobile** — responsive sizing, touch‑friendly hit areas, draggable card.

### The design

The signature is the **seal**. As you type, the field border and the medallion ring shift to wax; on a valid key the seal fills burgundy, a check stamps in with a small rotation, and the status reads *"Verified — access sealed."* Errors turn the seal red with an ✕. The perforation seam and notches are drawn from plain frames, while the small glyphs (close, the seal's check/✕, the link and cart icons) come from the Lucide icon sheet. Type is a heavy geometric wordmark, a clean body, and a monospace serial — warm cream on dark stock.

---

## Quick start

```lua
local Sigil = loadstring(game:HttpGet("YOUR_RAW_URL"))()

Sigil.Appearance.Title = "My Script"
Sigil.Links.GetKey  = "https://your-getkey-link"
Sigil.Links.Discord = "https://discord.gg/your-invite"

Sigil.Callbacks.OnVerify  = function(key)
    return key == "demo"            -- return true/false, or a result table (see below)
end
Sigil.Callbacks.OnSuccess = function()
    loadstring(game:HttpGet("YOUR_SCRIPT_URL"))()
end

Sigil:Launch()
```

### Junkie mode

```lua
local Sigil = loadstring(game:HttpGet("YOUR_RAW_URL"))()

Sigil.Callbacks.OnSuccess = function()
    loadstring(game:HttpGet("YOUR_SCRIPT_URL"))()
end

Sigil:LaunchJunkie({
    Service    = "your-service",
    Identifier = "your-identifier",
    Provider   = "jnkie",
})
```

In Junkie mode, `OnVerify` is ignored — verification goes through the SDK's `check_key`. If `Links.GetKey` is empty it's filled from `Junkie.get_key_link()`.

### Keyless mode

```lua
Sigil.Options.Keyless = true       -- force the keyless screen (seal is pre-stamped)
-- Sigil.Options.KeylessUI = false -- skip the screen entirely and continue
Sigil:Launch()
```

In `LaunchJunkie`, leave `Options.Keyless = nil` to auto‑detect keyless via `check_key("KEYLESS")`.

### Optional shop

```lua
Sigil.Shop.Enabled    = true
Sigil.Shop.Title      = "Get premium access"
Sigil.Shop.Subtitle   = "Instant delivery"
Sigil.Shop.ButtonText = "Buy"
Sigil.Shop.Link       = "https://your-shop-link"   -- copied to clipboard on click
Sigil.Shop.Icon       = ""                          -- optional rbxassetid://...
```

---

## The verify function

`OnVerify(key)` may return either a boolean **or** a result table. The table form lets you drive precise messages and behaviour:

```lua
Sigil.Callbacks.OnVerify = function(key)
    return { valid = false, error = "KEY_EXPIRED" }
    -- or: return { valid = true }
    -- or: return { valid = false, message = "Custom message" }
end
```

Recognised error codes and how Sigil reacts:

| Code                | Shown as              | Behaviour                              |
| ------------------- | --------------------- | -------------------------------------- |
| `KEY_INVALID`       | Key not found         | counts toward cooldown                 |
| `KEY_EXPIRED`       | Key has expired       | counts toward cooldown                 |
| `KEY_INVALIDATED`   | Key was revoked       | counts toward cooldown                 |
| `REVOKED`           | Key was revoked       | counts toward cooldown                 |
| `HWID_MISMATCH`     | HWID limit reached    | counts toward cooldown                 |
| `ALREADY_USED`      | Key already redeemed  | counts toward cooldown                 |
| `SERVICE_NOT_FOUND` | Service not found     | counts toward cooldown                 |
| `SERVICE_MISMATCH`  | Wrong service         | counts toward cooldown                 |
| `PREMIUM_REQUIRED`  | Premium required      | counts toward cooldown                 |
| `HWID_BANNED`       | Hardware banned       | kicks the player after ~2s             |
| `NETWORK` / `ERROR` | Network error         | **transient** — retried, no cooldown   |

A boolean `false` (or any unrecognised code) is treated as a generic invalid key. After **3** failed attempts the button locks for **5s**; after **5**, for **15s**. Transient/network failures never count toward this.

On success the key is saved (if `Storage.Remember`), `getgenv().SCRIPT_KEY` is set, and `OnSuccess` fires.

---

## Configuration

```lua
Sigil.Appearance = {
    Title           = "Sigil",
    Subtitle        = "Enter your key to continue",
    KeylessTitle    = "Sigil",
    KeylessSubtitle = "No key required for this build — you're verified.",
    Icon            = "",   -- optional rbxassetid:// shown left of the wordmark
}

Sigil.Links = {
    GetKey  = "",   -- "Get a key" button copies this
    Discord = "",   -- "Discord" button copies this (hidden when empty)
}

Sigil.Storage = {
    FileName = "SigilKey",  -- saved as SigilUI/SigilKey.txt
    Remember = true,        -- write the key to disk on success
    AutoLoad = true,        -- try a saved key automatically on launch
}

Sigil.Options = {
    Keyless   = nil,   -- nil: Junkie auto-detect · true: force · false: always require a key
    KeylessUI = true,  -- false: skip the keyless screen and continue
    Draggable = true,
}
```

The palette (paper, ink, wax) lives in a local theme table near the top of the file (`local T = { ... }`). Edit those values to recolour the pass.

---

## API

| Method                                       | Description                                                        |
| -------------------------------------------- | ------------------------------------------------------------------ |
| `Sigil:Launch()`                             | Start the key gate using `Callbacks.OnVerify`.                     |
| `Sigil:LaunchJunkie(config)`                 | Start in Junkie mode (`config = { Service, Identifier, Provider }`). |
| `Sigil:Notify(title, message, duration, kind)` | Show a toast. `kind`: `info`/`success`/`error`/`warning`/`copy`. |
| `Sigil:GetSavedKey()`                        | Return the saved key, or `nil`.                                    |
| `Sigil:ClearSavedKey()`                      | Delete the saved key.                                              |
| `Sigil:Destroy()`                            | Close everything and tear down.                                    |

`getgenv().SCRIPT_KEY` holds the accepted key (or `"KEYLESS"`). On a re‑run, a still‑valid `SCRIPT_KEY` skips the UI entirely.

---

## Compatibility with Onyx

Sigil and Onyx share the exact same engine, public API, and `getgenv().SCRIPT_KEY` contract. To switch between them you only change the `loadstring` URL and the variable name — your `OnVerify` / `OnSuccess` / `LaunchJunkie` code stays identical.

---

## Requirements

- An executor with `loadstring` and `game:HttpGet`.
- **Icons** are fetched once at runtime from the Lucide sprite sheet (`raw.githubusercontent.com/sarahsophiesee-bot/SakuraUI/.../lucide-roblox.luau`). If that request fails, the close button falls back to a drawn ✕ and the link/cart glyphs are simply omitted — everything still works.
- **Remembering keys** needs filesystem functions (`writefile`, `readfile`, `isfile`, `delfile`, `makefolder`, `isfolder`). Without them, verification still works — keys just aren't saved between runs.
- The "Get a key" / "Discord" / "Buy" buttons use `setclipboard`.

---

## License

MIT.
