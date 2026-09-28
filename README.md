# npm, PyPI & crates.io staleness/abandonment signals — a cheatsheet

![npm, PyPI & crates.io staleness/abandonment signals — a cheatsheet](assets/banner-1.png)

"Is this dependency still maintained?" gets a different answer shape from each registry: npm has a real opt-in flag most stale packages never set, PyPI has a status field that's just frozen text from whenever a maintainer last touched it, and crates.io has no opinion field at all — only the most precisely-dated timestamp of the three. Checked live against real packages.

## Where each registry puts it

| Registry | Field | Endpoint | Opt-in flag? | Reliable timestamp? |
|---|---|---|---|---|
| npm | `versions[v].deprecated` (string) | `GET https://registry.npmjs.org/{name}` | **Yes** — maintainer must set it explicitly | Yes — `time[v]` always present |
| PyPI | `info.classifiers[]` (`Development Status :: N - ...`) | `GET https://pypi.org/pypi/{name}/json` | Sort of — free text, never auto-revisited | Yes — `releases[v][*].upload_time_iso_8601` |
| crates.io | *(none)* | `GET https://index.crates.io/{path}` (sparse index) | **No** — no status field exists | Yes — `pubtime`, backfilled with no gaps |

## npm: `deprecated` is real, but silence doesn't mean healthy

```bash
curl -s https://registry.npmjs.org/istanbul | python3 -c "
import json, sys
d = json.load(sys.stdin)
v = d['dist-tags']['latest']
print(v, '|', d['time'][v], '|', d['versions'][v].get('deprecated'))
"
# 0.4.5 | 2016-08-21T20:02:09.468Z | "This module is no longer maintained, try this instead:\nnpm i nyc\n..."
```

Verified live 2026-09-28:

| Package | Latest version | Last published | `deprecated` set? |
|---|---|---|---|
| `istanbul` | 0.4.5 | 2016-08-21 | Yes — points to `nyc` |
| `jade` | 1.11.0 | 2015-06-12 | Yes — renamed to `pug` |
| `gulp-util` | 3.0.8 | 2016-12-26 | Yes — migration blog post |
| `request` | 2.88.2 | 2020-02-11 | Yes — links a GitHub issue |
| `bower` | 1.8.14 | 2022-03-14 | **No** — absent, despite the project's own README saying it's unmaintained |
| `moment`, `underscore` | (active) | 2026-09 / 2026-02 | No — correctly absent, both still shipping |

The flag only fires when a maintainer bothers to pull the trigger. `bower` shows the gap: genuinely dead, no flag, nothing at install time to warn you.

`colors` (the Jan 2022 sabotage incident) is a special case: the compromised versions (`1.4.1`, `1.4.2`) are visible in the registry's own `time` object, but `dist-tags.latest` was rolled back to `1.4.0` (2019) and never advanced — a plain `npm install colors` resolves to the pre-incident version; the bad ones are only reachable by pinning the exact version.

## PyPI: the status field exists, it just never changes

```bash
curl -s https://pypi.org/pypi/nose/json | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(d['info']['version'], [c for c in d['info']['classifiers'] if 'Development Status' in c])
"
# 1.3.7 ['Development Status :: 5 - Production/Stable']
```

Verified live 2026-09-28: `nose` — last release June 2015, superseded by pytest for over a decade — reports `Development Status :: 5 - Production/Stable`. `requests` — a release shipped this year — reports the **exact same string**. The classifier is whatever a maintainer typed at whatever release they picked; PyPI has no process that revisits it, so an eleven-year-dead package and an actively-shipping one read identically on this field.

## crates.io: no status field, but the cleanest timestamp of the three

```bash
curl -s https://index.crates.io/ru/st/rustc-serialize | tail -1 | python3 -m json.tool
```

Verified live 2026-09-28: every version line carries exactly `name`, `vers`, `deps`, `cksum`, `features`, `yanked`, `pubtime` — no `deprecated`, no classifier equivalent, not even a `homepage` to check by hand. `pubtime` is real and precise: `rustc-serialize` 0.3.25 (a crate long superseded by `serde`) shows `pubtime: 2023-12-01`, which is not a backfill artifact — crates.io backfills the field with no gaps but always preserves the *original* publish date, and 0.3.25 really was a genuine bugfix release shipped that day, eight years after the crate's main activity. crates.io gives you the most trustworthy raw timestamp and zero help deciding what to do with it.

![npm, PyPI and crates.io staleness signals, side by side](assets/banner-2.png)

## Practical takeaway

- **npm** — check `versions[v].deprecated`, but treat its absence as "nobody set it," not "actively maintained." Cross-check `time[v]` against today's date.
- **PyPI** — `Development Status` classifiers are frozen text, not a live signal; use `upload_time_iso_8601` for actual recency, ignore the classifier for staleness.
- **crates.io** — no flag exists; `pubtime` (sparse index, backfilled, gap-free) is your only signal, and it's reliable enough to build a staleness check on by itself.
- For a version-history incident (like `colors`), check `dist-tags.latest` — a compromised version can exist in the registry without ever being what a plain install resolves to.

This is one input into what my [Package Registry Scraper](https://apify.com/ponderable_hydrometer/package-registry-scraper) actor normalizes into one row per dependency across all three registries (staleness isn't a pulled field yet — on the list). The story behind why I went looking for this is on dev.to: [None of the Big Package Registries Can Tell You If a Dependency Is Abandoned](https://dev.to/ronin13/none-of-the-big-package-registries-can-tell-you-if-a-dependency-is-abandoned-1c56) (best-guess slug — dev.to appends a hash suffix on publish and sometimes truncates; confirm/fix at publish time).
