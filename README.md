# ExitZero Widgets

The widget inventory for the [ExitZero](https://github.com/meetkool/ExitZero) app.

Each widget is a single JSON file. The app ships a renderer; this repo ships
the descriptions. **Publishing a widget is a git push — users do not reinstall
the APK.**

## How it works

```
push widget.json  ─▶  raw.githubusercontent.com  ─▶  app fetches registry.json
                                                      │
                          Profile ▸ Settings ▸ Widgets ▸ Add
                                                      │
                                                      ▼
                                          card appears on the dashboard
```

The app reads `registry.json`, shows every entry in the in-app store, and
downloads a widget's manifest when the user taps **Add**. Manifests and their
data are cached on device, so installed widgets render instantly and survive
being offline.

## Publishing a widget

1. Add `widgets/<your-id>.json` (see the format below).
2. Add a matching row to `registry.json`.
3. Commit and push to `main`.

That's it. Users see it next time they open the widget store, or immediately
if they pull to refresh.

## Why JSON and not code

Flutter compiles ahead of time on Android and iOS, so the app physically
cannot load new Dart at runtime — and app stores forbid dynamic code delivery
anyway. A widget therefore describes *what to draw*, and the app decides *how*.
That is also what makes installing one safe: a manifest is a layout, it cannot
execute anything.

The practical limit: a widget can only use the component types listed below.
Anything new needs an app release.

## Manifest format

```jsonc
{
  "schemaVersion": 1,            // bump only on breaking changes
  "id": "github-profile",        // must match the registry row and filename
  "name": "GitHub Profile",
  "description": "Shown in the store.",
  "author": "meetkool",
  "version": "1.0.0",
  "icon": "code",                // from the icon list below
  "accent": "#F77F00",           // default colour for components

  "layout": {
    "span": 2,                   // 1 = half width, 2 = full width
    "minSpan": 1, "maxSpan": 2,
    "height": 130,
    "minHeight": 110, "maxHeight": 220
  },

  // Optional. Prompted once, when the user installs.
  "config": [
    { "key": "user", "label": "GitHub username", "type": "string",
      "default": "meetkool", "hint": "e.g. torvalds", "required": true }
  ],

  // Optional. Leave it out for a widget that needs no network.
  "data": {
    "url": "https://api.github.com/users/{{config.user}}",
    "method": "GET",
    "headers": {},
    "refreshSeconds": 1800,
    "root": "data.items"         // optional: unwrap before binding
  },

  "body": [ /* components */ ]
}
```

### Rules the app enforces

- `data.url` **must be https**. Anything else is refused at runtime.
- An unknown component type is skipped, not crashed on.
- A manifest with a `schemaVersion` newer than the installed app shows
  "Update required" instead of rendering partially.
- A failed refresh keeps showing the last good data rather than an error.

## Expressions

Any string field can contain `{{ ... }}`.

| Root | Is |
|---|---|
| `data` | the parsed response body (after `root`) |
| `config` | what the user typed at install |
| `item` | the current entry inside a `list` |
| `index` | the current index inside a `list` |

Paths walk maps by key and lists by number: `{{data.items.0.title}}`.
Supply a fallback with `??`:

```
{{data.followers ?? 0}}
{{data.bio ?? No bio set}}
```

## Components

| Type | Purpose | Main fields |
|---|---|---|
| `label` | small spaced caption | `text`, `color`, `size` |
| `text` | body text | `text`, `size`, `weight`, `color`, `maxLines`, `format` |
| `metric` | the big number | `value`, `unit`, `caption`, `format`, `size`, `color` |
| `iconBadge` | icon in a tinted circle | `icon`, `color`, `size`, `bgOpacity` |
| `badge` | small pill | `text` **or** `value` + `format` + `prefix`/`suffix`, `color` |
| `progressBar` | horizontal bar | `value`, `max`, `color`, `thickness` |
| `progressRing` | ring | `value`, `max`, `color`, `size`, `centerText`, `centerIcon` |
| `row` | horizontal group | `children`, `justify`, `align`, `gap` |
| `column` | vertical group | `children`, `justify`, `align`, `gap`, `flex`, `expand` |
| `list` | repeat over an array | `source`, `item`, `limit`, `gap`, `emptyText` |
| `divider` | hairline | `height` |
| `spacer` | gap or flexible space | `size` (omit to flex) |

Every component also accepts `showIf`: when its expression resolves to empty,
`0` or `false`, the component is left out.

`justify`: `start` `center` `end` `between` `around`
`align`: `start` `center` `end` `stretch`
`format`: `compact` `percent` `integer` `oneDecimal`

### Colours

Brand tokens `orange` `burnt` `teal` `tealLight` `cream` `dark` `deep`,
basics `white` `black` `green` `red` `yellow` `blue` `purple` `grey`,
or a literal `#RRGGBB` / `#AARRGGBB`.

### Icons

Only names the app knows are allowed — a manifest cannot point at an arbitrary
glyph, which keeps icon tree-shaking working:

`widgets` `star` `favorite` `bolt` `local_fire_department` `code` `terminal`
`bug_report` `commit` `trending_up` `trending_down` `timeline` `insights`
`check_circle` `cancel` `schedule` `alarm` `calendar_today` `event`
`notifications` `mail` `send` `chat` `person` `group` `work` `school` `book`
`water_drop` `restaurant` `fitness_center` `directions_run` `bedtime`
`wb_sunny` `cloud` `thermostat` `attach_money` `savings` `shopping_cart`
`music_note` `movie` `sports_esports` `flag` `emoji_events` `lightbulb`
`format_quote` `link` `cloud_off` `error_outline`

## A complete example

`widgets/water-intake.json` needs no network at all:

```json
{
  "schemaVersion": 1,
  "id": "water-intake",
  "name": "Water Intake",
  "icon": "water_drop",
  "accent": "#3A86FF",
  "layout": { "span": 1, "height": 140 },
  "config": [
    { "key": "glasses", "label": "Glasses so far today", "type": "number", "default": "3" },
    { "key": "target",  "label": "Daily target",         "type": "number", "default": "8" }
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

## Pointing the app somewhere else

The app reads this repo because of `lib/services/widget_repo_config.dart` in
ExitZero. Change `owner`, `repo` or `branch` there to run your own inventory.
