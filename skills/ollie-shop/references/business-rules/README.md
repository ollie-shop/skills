# Business Rules

This folder holds **merchant-specific business rules** documented in markdown. Each file describes one merchant's checkout customizations — things like shipping thresholds, payment restrictions, loyalty integrations, and UI modifications that go beyond the default template.

The skill reads every markdown file here during **Step 2: Identify Business Rules** of the component design flow (see [`component-design-flow.md`](../component-design-flow.md)) to produce the gap analysis and component backlog.

## The `rule.json` artifact (chupacabra crawl output)

Alongside the human-readable `rule.md`, a merchant may have a machine artifact **`rule.json`** — the raw output of the **chupacabra** crawler (`apps/chupacabra`). It carries:

```jsonc
{
  "platform": "vtex",
  "platformStoreId": "acme",
  "logo": "https://.../logo.svg",
  "theme": { /* themeClassification — brand colors, typography, button/input style */ },
  "businessRules": [ /* same rules as rule.md, with category/confidence/isThirdParty */ ]
}
```

`theme` here is the **classification** shape (`primaryColor`, `secondaryColor`, `accentColor`, `backgroundColor`, `typography.{headingFont,bodyFont,…Url}`, `buttonStyle.{borderRadius,textTransform}`, `inputStyle.{borderRadius,borderColor,backgroundColor}`) — NOT the final design tokens.

### Deriving `ollie.json` from `rule.json`

The store's `ollie.json` (project root) is the source of truth that feeds Supabase `versions.theme` (a `Record<string,string>` of CSS custom properties) and `versions.props`. Convert the classification into the daisyUI/Tailwind tokens (defined in `packages/config/tailwind.css`):

| `rule.json` → `theme.*` | `ollie.json` → `theme["--token"]` |
|---|---|
| `primaryColor` | `--color-primary` (+ `--color-primary-content` for contrast) |
| `secondaryColor` | `--color-secondary` (+ `-content`) |
| `accentColor` | `--color-accent` (+ `-content`) |
| `backgroundColor` | `--color-base-200` (subtle/page surface; keep `--color-base-100` white for cards) |
| `inputStyle.borderColor` | `--color-base-300` (borders/dividers) |
| `buttonStyle.borderRadius` | `--radius-field` |
| `inputStyle.borderRadius` | `--radius-selector` |
| (cards/modals) | `--radius-box` |

**Colors are tokens; fonts are props.** Typography does NOT go in `theme` — there is no font design token. Put it under `ollie.json` → `props.font`:

```jsonc
"props": {
  "font": {
    "fontFamily": "Sana Sans Alt",
    "href": "https://.../font.woff2"   // headingFontUrl/bodyFontUrl from rule.json
  }
}
```

`buttonStyle.textTransform` (e.g. `uppercase`) also has no token — surface it as a prop or in the component CSS, not in `theme`.

Result shape (`ollie.json`): `{ storeId, theme: { "--color-*": "...", "--radius-*": "..." }, props: { font: { fontFamily, href } } }`. `ollie.json` is `passthrough`, so these extra fields are valid. See a merchant's `ollie.json` for a worked example.

## Convention

- One file per merchant (e.g. `acme-store.md`, `my-store.md`).
- Frontmatter fields:
  - `title` — human-readable name
  - `platform` — e.g. `VTEX`, `Shopify`
  - `account` — the platform account identifier
  - `analyzedAt` — ISO date of the analysis
- Group rules by category: Interface, Payment, Catalog, Shipping, User Experience, Profile, Integration, Analytics.
- For each rule, include: name, short behavioral description, and confidence score when auto-detected.

## Related

- [`component-design-flow.md`](../component-design-flow.md) — how these rules feed into the component backlog
- Oliver agent pipeline (`oliver-rules-comparator` → `oliver-slot-resolver` → `oliver-software-architect`) — consumes these files to produce CR-*/SR-*/COMP-* specs
