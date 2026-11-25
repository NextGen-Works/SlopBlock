# SLOP Block

A privacy-first browser extension that finds, warns about, hides and reports spammy
or suspicious page content. Everything runs locally: no backend, no accounts, no
telemetry, and no remote code.

SLOP Block is a small, inspectable rule-and-score classifier. It does not use machine
learning and does not send page content anywhere unless you explicitly turn on an
optional remote endpoint and confirm it.

## Features

- **Visibility-first scanning.** Only content that is actually rendered is scanned.
  Scanning is driven by `IntersectionObserver`, a debounced `MutationObserver` and
  idle-frame batches, and stops when the page is hidden.
- **Explainable scores.** Every detection gets a confidence from `0.0` to `1.0`, a
  `Low` / `Medium` / `High` / `Very High` label, a category and a pattern ID.
- **Conservative removal.** Automatic hiding happens only at **Very High** confidence
  by default. You can lower the threshold to High or Medium, but the default stays
  cautious.
- **Sensitive-site protection.** Banking, government, medical, password-manager,
  authentication, payment and security pages are detected from hostnames, URL paths
  and visible form fields. On those pages the extension warns instead of hiding, and
  forms, inputs, video/audio players and dialogs are never removed.
- **Allow and block controls.** Allow once, for the session, for 5 or 15 minutes, for
  an hour, until the browser closes, or permanently. Trusted domains always win over
  blocks and automatic decisions.
- **Custom regex rules.** Add block or unblock rules with validation, a sample tester,
  unsafe-regex warnings and explicit confirmation for expensive patterns. No remote
  JavaScript is ever evaluated.
- **Local stats and reports.** A dashboard shows local counters, confidence and
  category breakdowns. Reports can be saved locally, exported as JSON, or used to
  prefill a GitHub issue page (no token or API call).
- **Optional debug logs.** Off by default. When enabled, metadata-only logs expire
  automatically after 7 days. Excerpts require a second opt-in.
- **Simple and advanced modes.** Advanced sections are gated, and an optional
  parent/guardian PIN protects them with a local PBKDF2-salted hash and failed-attempt
  delays.
- **Local-only sync.** Settings are written to `storage.local` with a mirror key and a
  `storage.onChanged` bridge. `storage.sync` is never used.
- **Dark, light and system themes**, English and Spanish UI, keyboard-friendly controls,
  and no external assets or trackers.

## Install for development

Requirements: Node.js 20 or newer.

```bash
npm install
npm run build:icons   # only needed if src/icons/*.png are missing
npm run build
```

The build writes two unpacked extensions:

- `dist/chrome` — Manifest V3 with a minimal service worker
- `dist/firefox` — Manifest V3 with Firefox-specific metadata

To load it:

- **Chrome / Edge:** open `chrome://extensions`, enable Developer mode, choose
  *Load unpacked*, and select `dist/chrome`.
- **Firefox:** open `about:debugging#/runtime/this-firefox`, choose *Load Temporary
  Add-on*, and select `dist/firefox/manifest.json`.

Optional packaging:

```bash
npm run zip      # writes artifacts/slop-block-<browser>-<version>.zip
```

## Quality checks

```bash
npm run lint
npm test
npm run build
npm run check    # all three
```

Tests use Vitest with jsdom and an in-memory WebExtension API. They cover domain
normalization, sensitive-site detection, rule validation, regex safety, classifier
scoring, allow/block precedence, storage, settings export/import, passcode lockout,
debug-log expiry, GitHub report URLs, remote fetch failure modes and the DOM scanner.

## How detection works

1. The content script collects visible candidate containers (`article`, `li`,
   `blockquote`, post/comment/test-id containers, and standalone paragraphs).
2. Obvious non-content regions are skipped: navigation, headers, footers, forms,
   inputs, password-manager fields, video/audio players and dialogs.
3. The local classifier combines:
   - built-in rules (`src/common/default-rules.js`),
   - your custom rules,
   - optional validated remote rules,
   - and bounded heuristics (link farms, shorteners, IP-literal links, punycode,
     promo wording, urgency, all-caps, emoji floods, hashtag/mention spam, duplicate
     text, money and crypto pitches, engagement bait).
4. Link and unblock rules are checked first. An unblock rule short-circuits detection.
5. `src/common/allow.js` resolves the final action using this precedence:
   explicit allow → trusted domain → custom unblock → manual allow → manual block →
   automatic classification.
6. At Very High confidence the container is hidden inline with a small card and a
   *Show* action. At High or Medium (or on sensitive pages) you get a warning card
   instead.

Scores are heuristic, not a verdict. False positives and false negatives are possible;
the report flow exists so rules can be improved.

## Allow scopes

| Scope | Survives reload | Ends |
| --- | --- | --- |
| Once | No | After the current element is revealed |
| Session | Yes, this tab session | When the browser session ends |
| 5 / 15 / 60 minutes | Yes | At the chosen time |
| Until browser close | Yes | When the browser closes |
| Permanent | Yes | When you remove it |

## Custom rules

Rules are JSON objects. See [`docs/rules.md`](docs/rules.md) and
[`rules/rules.schema.json`](rules/rules.schema.json).

```json
{
  "version": 1,
  "rules": [
    {
      "id": "my-promo-rule",
      "type": "block",
      "category": "spam_promo",
      "textRegex": "\\b(?:buy now|limited offer)\\b",
      "confidenceBoost": 0.3,
      "why": "Pushy sales wording"
    }
  ]
}
```

Unsafe patterns (nested quantifiers, huge repetition counts, stacked wildcards) are
flagged before saving and are rejected outright in remote rule sets.

A complete, validated sample payload lives at
[`examples/remote-rules.example.json`](examples/remote-rules.example.json). Upload it
anywhere static (HTTPS) to try the optional remote flow.

## Optional remote rules and classifier

Remote support is **off by default** and requires all of the following:

- a URL you enter yourself (HTTPS only, `http://localhost` allowed for development),
- a separate confirmation checkbox,
- the host permission requested from the options page,
- a validated payload with `"version": 1`.

One invalid rule rejects the entire remote payload (fail closed). Responses are capped
at 1 MB with a 12-second timeout, rate limits and exponential backoff. No secrets,
tokens or account data are stored or sent. The remote classifier receives aggregate
features only; sharing an excerpt requires a second opt-in.

## Project layout

```
src/
  common/        shared UMD modules (also loadable from Node tests)
  content/       scanner, inline cards, bootstrap
  background/    MV3 service worker (storage, badges, remote fetch, reports)
  popup/         toolbar popup
  options/       full settings UI
  dashboard/     local statistics dashboard
  i18n/          runtime dictionaries (en, es)
  _locales/      manifest metadata strings
  icons/         generated PNG icons
scripts/         icon generator and build/zip tool
rules/           generated default rules + JSON schema
test/            Vitest suites
docs/            architecture, rules and privacy notes
```

Shared modules use a classic-script UMD pattern (`root.SLOP.*` plus
`module.exports`) so the same files work in content scripts, extension pages, the
service worker and Node tests without a bundler.

## Permissions

- `storage` — settings, statistics, reports, allow lists, optional debug logs.
- `activeTab` — current-tab actions from the popup.
- Optional host permissions — only requested if you configure a remote rules or
  classifier URL.

There is no `<all_urls>` host permission for network access, no analytics and no
external fonts or scripts.

## Privacy

See [PRIVACY.md](PRIVACY.md) for the full data inventory and retention rules.

## Known limitations

- Rules are heuristics and can misfire on satire, marketing, translated text and
  unusual layouts.
- The scanner intentionally ignores cross-origin iframes and closed shadow roots.
- Very large or virtualized feeds are processed incrementally; newly rendered content
  is picked up by the mutation observer.
- The Firefox build targets MV3; older Firefox versions may need
  `browser_specific_settings.gecko.strict_min_version` raised.

## License

MIT. See [LICENSE](LICENSE).
