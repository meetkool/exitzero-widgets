# ExitZero Widgets

The widget inventory for the [ExitZero](https://github.com/meetkool/ExitZero) app.

A widget is **one JSON file**. The app ships the renderer; this repo ships the
descriptions. **Publishing a widget is a git push — nobody reinstalls the APK.**

---

## Publish a widget in three steps

1. Create `widgets/<your-id>.json`
2. Add a matching row to `registry.json`
3. Commit and push to `main`

Then on the phone: **Profile ▸ ⚙️ ▸ Widgets ▸ pull down to refresh**.

Pull-to-refresh also re-fetches the manifests of widgets you already have, so
**editing a published widget reaches phones that already installed it**. You do
not need to remove and re-add.

Two caches sit in the way, so allow a moment:

| Cache | Length | Skip it by |
|---|---|---|
| GitHub raw CDN | 5 minutes | waiting |
| The app's own | until refresh | pulling to refresh |

---

## Why JSON and not code

Flutter compiles ahead of time, so the app physically cannot load new Dart at
runtime, and app stores forbid shipping code this way. A widget therefore
describes **what to draw**, and the app decides **how**. That is also what makes
installing one safe: a manifest is a layout, it cannot execute anything.

**The practical boundary:**

| Needs a new APK | Just a push |
|---|---|
| A new **component type** | New widgets using existing components |
| An **icon** outside the 49 below | Any `#RRGGBB` colour |
| A new **config field type** | New data sources, URLs, refresh rates |
| Raising `schemaVersion` past 1 | Editing, re-versioning, removing widgets |

---

## Manifest format

```jsonc
{
  "schemaVersion": 1,            // this app supports 1
  "id": "my-widget",             // must match the filename and registry row
  "name": "My Widget",
  "description": "Shown in the store.",
  "author": "meetkool",
  "version": "1.0.0",
  "icon": "star",                // from the icon list
  "accent": "#F77F00",           // default colour for components
  "tags": ["fun"],

  "layout": {
    "span": 2,                   // 1 = half width, 2 = full width
    "minSpan": 1, "maxSpan": 2,
    "height": 140,
    "minHeight": 100, "maxHeight": 260
  },

  // Optional. Asked once, when the user installs.
  "config": [
    { "key": "user", "label": "GitHub username", "type": "string",
      "default": "meetkool", "hint": "e.g. torvalds", "required": true }
  ],

  // Optional. Leave it out for a widget that needs no network.
  "data": {
    "url": "https://api.example.com/thing/{{config.user}}",
    "method": "GET",             // GET or POST
    "headers": {},
    "refreshSeconds": 900,
    "root": "data.items"         // optional: unwrap before binding
  },

  "body": [ /* components */ ]
}
```

### Sizing rule — read this one

A card is 32px of padding plus a ~24px heading row plus your components. A
component with a **fixed size** (`avatar3d`, `globe3d`, `cameraFeed`,
`dualCameraFeed`, `cameraCapability`) will overflow if the card can be made
smaller than it needs. Keep:

```
minHeight  >=  component size + 70
```

Check it at **minHeight**, not at `height` — the user can shrink the card.

### What the app enforces

- `data.url` **must be https**. Anything else is refused at runtime.
- An **unknown component** renders "Update the app to see this" naming the type,
  so an out-of-date app is obvious rather than a blank card.
- A `schemaVersion` newer than the app shows "Update required" instead of
  rendering half a widget.
- A **failed refresh keeps the last good data** rather than flipping to an error.

---

## Expressions

Any string field may contain `{{ ... }}`.

| Root | Is |
|---|---|
| `data` | the parsed response body, after `root` |
| `config` | what the user typed at install |
| `item` | the current entry inside a `list` |
| `index` | the current index inside a `list` |

Paths walk maps by key and lists by number: `{{data.items.0.title}}`.
Supply a fallback with `??`:

```
{{data.followers ?? 0}}
{{data.bio ?? No bio set}}
```

**Config values always arrive as strings.** `{{config.count ?? 5}}` yields
`"5"`, not `5` — numeric fields parse it, but keep it in mind.

---

## Components

Every component also accepts **`showIf`**: when its expression resolves to
empty, `0` or `false`, the component is left out silently.

### Text and numbers

| Type | Fields |
|---|---|
| `label` | `text`, `color`, `size` — small spaced caption, upper-cased for you |
| `text` | `text`, `size`, `weight`, `color`, `maxLines`, `format` |
| `metric` | `value`, `unit`, `unitSize`, `caption`, `format`, `size`, `color` |
| `badge` | `text` **or** `value`+`format`+`prefix`+`suffix`, `color`, `size` |

`badge` has two modes: plain `text`, or a `value` you want **formatted** —
formatting needs the number alone, which a template with words around it cannot
give:

```json
{ "type": "badge", "value": "{{data.change ?? 0}}",
  "format": "oneDecimal", "suffix": "% 24h", "color": "green" }
```

### Indicators

| Type | Fields |
|---|---|
| `iconBadge` | `icon`, `color`, `size`, `bgOpacity` |
| `progressBar` | `value`, `max`, `color`, `thickness` |
| `progressRing` | `value`, `max`, `color`, `size`, `thickness`, `centerText`, `centerIcon` |

`value` accepts a 0–1 fraction, a 0–100 number, or a `value`/`max` pair.

### Layout

| Type | Fields |
|---|---|
| `row` | `children`, `justify`, `align`, `gap` |
| `column` | `children`, `justify`, `align`, `gap`, `flex`, `expand` |
| `divider` | `height` |
| `spacer` | `size` (omit to flex) |

`justify`: `start` `center` `end` `between` `around`
`align`: `start` `center` `end` `stretch`
`weight`: `normal` `medium` `semibold` `bold`
`format`: `compact` `percent` `integer` `oneDecimal`

> **Long text belongs in a `column`, not a `row`.** Row children are not wrapped
> in `Flexible`, so a long string inside a row overflows its card. A column
> gives its children the full width, so text ellipsises properly.

### Repeating

```json
{ "type": "list",
  "source": "{{data}}",          // or "{{data.items}}"
  "limit": 3,
  "gap": 8,
  "emptyText": "Nothing yet.",
  "item": { "type": "column", "children": [
    { "type": "text", "text": "{{item.name}}", "weight": "semibold" },
    { "type": "text", "text": "{{item.detail}}", "size": 10 }
  ]}
}
```

### Scenes and hardware

| Type | Fields |
|---|---|
| `avatar3d` | `size`, `color`, `secondsPerLap`, `showTrack` |
| `globe3d` | `size`, `color`, `secondsPerSpin`, `showAtmosphere`, `showSatellite`, `markers` |
| `cameraFeed` | `size`, `color`, `facing` (`front`/`back`) |
| `dualCameraFeed` | `size`, `color`, `backRotation`, `frontRotation`, `mirrorFront`, `debug` |
| `cameraCapability` | `size`, `color` |
| `spotdlDownloader` | `size`, `color`, `baseUrl`, `query` |

`globe3d` markers are literal degrees:

```json
"markers": [ { "lat": 19.07, "lon": 72.87 }, { "lat": 51.51, "lon": -0.13 } ]
```

`dualCameraFeed` rotations are **clockwise degrees** (`0`, `90`, `180`, `270`)
and live here rather than in the app, so a device needing different numbers is a
push. `debug: true` prints each lens's reported sensor orientation on its badge,
which is how you find the right value on a new phone.

`spotdlDownloader` is a client for a [spotDL](https://github.com/spotDL/spotify-downloader)
web server **you run yourself** — spotDL is Python, so none of it runs on the
phone. Start it on your machine with `spotdl web --host 0.0.0.0`, then point
`baseUrl` at that machine's LAN address:

```json
{
  "type": "spotdlDownloader",
  "size": 300,
  "color": "#1DB954",
  "baseUrl": "{{config.server}}",
  "query": "{{config.query}}"
}
```

Typing a search term lists tracks; pasting a Spotify playlist, album or artist
link lists everything behind it. The download button asks the server for that
track and copies the finished mp3 into `Music/ExitZero`, so the `musicPlayer`
widget finds it straight away. `baseUrl` is the one field that belongs in
`config` rather than the manifest — it is different for every person.

---

## Colours

Brand tokens: `orange` `burnt` `teal` `tealLight` `cream` `dark` `deep`
Basics: `white` `black` `green` `red` `yellow` `blue` `purple` `grey`
Or a literal `#RRGGBB` / `#AARRGGBB`.

## Icons

Only these 49. A manifest cannot point at an arbitrary glyph — that keeps
Flutter's icon tree-shaking working and the app small:

```
alarm            attach_money     bedtime          bolt             book
bug_report       calendar_today   cancel           chat             check_circle
cloud            cloud_off        code             commit           directions_run
emoji_events     error_outline    event            favorite         fitness_center
flag             format_quote     group            insights         lightbulb
link             local_fire_department              mail             movie
music_note       notifications    person           restaurant       savings
schedule         school           send             shopping_cart    sports_esports
star             terminal         thermostat       timeline         trending_down
trending_up      water_drop       wb_sunny         widgets          work
```

---

## A complete example, no network

`widgets/water-intake.json`:

```json
{
  "schemaVersion": 1,
  "id": "water-intake",
  "name": "Water Intake",
  "icon": "water_drop",
  "accent": "#3A86FF",
  "layout": { "span": 1, "height": 140, "minHeight": 120, "maxHeight": 220 },
  "config": [
    { "key": "glasses", "label": "Glasses so far", "type": "number", "default": "3" },
    { "key": "target",  "label": "Daily target",   "type": "number", "default": "8" }
  ],
  "body": [
    { "type": "label", "text": "Hydration" },
    { "type": "row", "justify": "between", "align": "center", "children": [
      { "type": "metric", "value": "{{config.glasses ?? 0}}",
        "unit": "/ {{config.target ?? 8}}", "caption": "glasses" },
      { "type": "progressRing", "value": "{{config.glasses ?? 0}}",
        "max": "{{config.target ?? 8}}", "color": "#3A86FF",
        "size": 52, "centerIcon": "water_drop" }
    ]}
  ]
}
```

And its `registry.json` row:

```json
{
  "id": "water-intake",
  "name": "Water Intake",
  "description": "Daily hydration ring.",
  "version": "1.0.0",
  "icon": "water_drop",
  "accent": "#3A86FF",
  "tags": ["health"],
  "path": "widgets/water-intake.json"
}
```

---

## Checklist before pushing

- [ ] `id` matches the filename **and** the registry row
- [ ] Every `{{config.x}}` has a matching `config` entry
- [ ] Every `icon` is in the list above
- [ ] Every non-hex `color` is a token above
- [ ] `data.url` starts with `https://`
- [ ] `minHeight >= fixed component size + 70`
- [ ] Long text is inside a `column`, not a `row`
- [ ] The JSON parses (`python3 -m json.tool widgets/yours.json`)

Bump `version` in **both** the manifest and the registry row when you change a
published widget, so it is obvious in the store which copy is out there.

## Running your own inventory

The app reads this repo because of `lib/services/widget_repo_config.dart` in
ExitZero. Change `owner`, `repo` or `branch` there to point it elsewhere.
