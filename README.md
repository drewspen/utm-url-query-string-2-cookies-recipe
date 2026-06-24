# UTM & URL Query String to Cookies — GTM Recipe

> Persist any URL query string parameter — UTM values, ad platform click IDs, or custom campaign codes — into session or long-lived persistent browser cookies using only GTM sandboxed Custom Templates. No Custom HTML tags. No new CSP hashes. First-touch UTM attribution that survives across sessions and days.

---

## The Problem

UTM parameters exist only in the landing page URL. Navigate to any other page and they are gone from the address bar. GA4 attributes them to the session, but it does not make the raw values available to your form scripts, CRM integration, or marketing automation platform.

**The real-world gap:** A B2B prospect clicks a LinkedIn ad on Monday and lands on your product page. On Thursday they return directly, read the pricing page, and submit a contact form. GA4 attributes the Thursday session to *direct*. Your CRM receives no campaign data. You cannot connect the closed deal back to the LinkedIn campaign spend.

This recipe bridges that gap. On the first tagged landing it writes the UTM values to browser cookies — once as a session cookie for current-visit attribution, once as a persistent cookie for first-touch attribution that survives for up to 730 days. Every subsequent page view and GA4 event carries both sets of values automatically via the Shared Event Settings variable.

---

## Why No Custom HTML — and Why That Matters for CSP

The traditional GTM approach to writing cookies is a Custom HTML tag containing an inline `<script>` block. Every such script must be covered by its SHA-256 hash in your `Content-Security-Policy` `script-src` header. Change one character and the hash changes; the CSP must be redeployed and reviewed.

This recipe uses only GTM **sandboxed Custom Templates**. Template code executes within GTM's own sandbox using internal APIs (`setCookie`, `getCookieValues`) that operate under the permission already granted to the GTM container — no new `<script>` tags are injected into the DOM, and your CSP never needs to change regardless of how many times the template configuration is updated.

---

## Download

| File | Link |
|---|---|
| GTM Container JSON | [Download from Google Drive](https://drive.google.com/file/d/1YOIJseTGI_XH5O-HPkHOEcbHK01bQCpm/view?usp=drivesdk) |

Import using **Admin → Import Container → Merge** in GTM to preserve existing tags and triggers.

---

## What's in the Container

### Folders

| Folder | Contents |
|---|---|
| `Analytics` | GA4 configuration tag, shared event settings (including UTM cookie reads), environment/stream routing, client ID and timestamp infrastructure |
| `Process URL query strings 2 cookies` | Two Write tags (session + persistent) and ten Cookie Variable variables |
| `Extend Cookie Life` | Cookie Rewriter tag, sentinel guard variables, and DOM Ready trigger |

### Tags

| Tag | Template | Trigger | Purpose |
|---|---|---|---|
| `Google Analytics Configuration` | Google Tag (`googtag`) | All Pages (once per load) | GA4 config tag; Shared Event Settings automatically include UTM cookie values on every hit |
| `Write certain URL query strings 2 cookies session` | Write URL query strings 2 cookies | All Pages | Reads enabled URL query parameters and writes session cookies (expire on browser close) |
| `Write certain URL query strings 2 cookies user` | Write URL query strings 2 cookies | All Pages | Reads the same parameters and writes persistent cookies (`_1` suffix, 730-day expiry); preserves existing values to maintain first-touch attribution |
| `Extend Cookie Life Tag` | Cookie Rewriter | DOM Ready (with sentinel guard) | Rewrites all ten persistent UTM cookies to reset expiry to 720 days from now on each visit |

### Triggers

| Trigger | Type | Condition | Purpose |
|---|---|---|---|
| All Pages (built-in) | Page View | — | Fires the two Write tags on every page so incoming UTMs are always captured |
| `Extend Cookie Life Trigger` | DOM Ready | `_arrewr` cookie is absent or empty | Fires the Cookie Rewriter exactly once per page load, guarded by sentinel cookie |

---

## Two Tracks: Session and Persistent

### Session Cookies — Current Visit Attribution

Expire when the browser closes. Hold UTM values for the current visit only. Available throughout the same browsing session regardless of how many pages the user navigates.

| Cookie Name | Reads From |
|---|---|
| `utm_source` | `utm_source` URL query parameter |
| `utm_medium` | `utm_medium` URL query parameter |
| `utm_campaign` | `utm_campaign` URL query parameter |
| `utm_content` | `utm_content` URL query parameter |
| `utm_term` | `utm_term` URL query parameter |

### Persistent Cookies — First-Touch Attribution

Survive browser restarts. Expire 730 days from initial write (rolling — extended on each visit by the Cookie Rewriter). The `_1` suffix convention marks these as first-touch values. Once written, the tag does not overwrite an existing persistent cookie, preserving the original campaign data.

| Cookie Name | Expiry | Reads From |
|---|---|---|
| `utm_source_1` | 730 days (rolling) | `utm_source` on first UTM-tagged landing |
| `utm_medium_1` | 730 days (rolling) | `utm_medium` on first UTM-tagged landing |
| `utm_campaign_1` | 730 days (rolling) | `utm_campaign` on first UTM-tagged landing |
| `utm_content_1` | 730 days (rolling) | `utm_content` on first UTM-tagged landing |
| `utm_term_1` | 730 days (rolling) | `utm_term` on first UTM-tagged landing |

> **`_u` variants:** The Extend Cookie Life Tag also maintains `utm_*_u` cookies. These are reserved for a "last UTM-tagged visit" persistent track — configure the user write tag with a `_u` suffix to enable last-touch persistent attribution alongside the first-touch `_1` values.

---

## Rolling Cookie Expiry

The `Extend Cookie Life Tag` runs the Cookie Rewriter on every page visit to reset the expiry of all ten persistent cookies to 720 days from the current time. This means an active user's attribution window never expires as long as they visit at least once every 720 days.

### Sentinel Guard (`_arrewr`)

To ensure the Cookie Rewriter fires exactly once per page load (not once per DOM Ready event if multiple fire), the `Extend Cookie Life Trigger` checks the `_arrewr` cookie against the regex `^(undefined|null|0|false|NaN|)$`. The Cookie Rewriter sets `_arrewr` on its first run, suppressing any further firing on the same page.

---

## Cookie Variable Variables

Ten built-in First-Party Cookie variables read the values back for use in GTM tags, triggers, and shared event settings:

| Variable Name | Cookie Read | Track |
|---|---|---|
| `utm_source cookie session` | `utm_source` | Current session |
| `utm_medium cookie session` | `utm_medium` | Current session |
| `utm_campaign cookie session` | `utm_campaign` | Current session |
| `utm_content cookie session` | `utm_content` | Current session |
| `utm_term cookie session` | `utm_term` | Current session |
| `utm_source cookie user` | `utm_source_1` | First touch (persistent) |
| `utm_medium cookie user` | `utm_medium_1` | First touch (persistent) |
| `utm_campaign cookie user` | `utm_campaign_1` | First touch (persistent) |
| `utm_content cookie user` | `utm_content_1` | First touch (persistent) |
| `utm_term cookie user` | `utm_term_1` | First touch (persistent) |

---

## Shared Event Settings — UTM Parameters on Every GA4 Hit

Six UTM parameters are pre-wired into the Google Tag Shared Event Settings variable, so they appear automatically on every GA4 event:

| GA4 Parameter | GTM Variable | Attribution |
|---|---|---|
| `e_cur_utm_source` | `{{utm_source cookie session}}` | Current session source |
| `e_cur_utm_medium` | `{{utm_medium cookie session}}` | Current session medium |
| `e_cur_utm_campaign` | `{{utm_campaign cookie session}}` | Current session campaign |
| `e_1st_utm_source` | `{{utm_source cookie user}}` | First-touch source |
| `e_1st_utm_medium` | `{{utm_medium cookie user}}` | First-touch medium |
| `e_1st_utm_campaign` | `{{utm_campaign cookie user}}` | First-touch campaign |

Add `e_cur_utm_content`, `e_cur_utm_term`, `e_1st_utm_content`, and `e_1st_utm_term` to the Shared Event Settings table if you need those dimensions in GA4 reports.

---

## Supported Query String Parameters

The `Write URL query strings 2 cookies` template supports the following parameters. The container ships with UTM parameters enabled and all click IDs disabled.

| Parameter | Default | Platform / Purpose |
|---|---|---|
| `utm_source` | ✅ Enabled | Standard UTM — traffic source |
| `utm_medium` | ✅ Enabled | Standard UTM — traffic medium |
| `utm_campaign` | ✅ Enabled | Standard UTM — campaign name |
| `utm_content` | ✅ Enabled | Standard UTM — ad content / variant |
| `utm_term` | ✅ Enabled | Standard UTM — keyword |
| `gclid` | ☐ Disabled | Google Ads — auto-tagged click ID |
| `gad_source` | ☐ Disabled | Google Ads — ad source |
| `gad_campaignid` | ☐ Disabled | Google Ads — campaign ID |
| `gbraid` | ☐ Disabled | Google Ads — iOS app conversion |
| `wbraid` | ☐ Disabled | Google Ads — web-to-app conversion |
| `gclsrc` | ☐ Disabled | Google — DoubleClick import source |
| `dclid` | ☐ Disabled | Google Display & Video 360 click ID |
| `msclkid` | ☐ Disabled | Microsoft Advertising click ID |
| `fbclid` | ☐ Disabled | Meta (Facebook/Instagram) click ID |
| `igshid` | ☐ Disabled | Instagram share ID |
| `ttclid` | ☐ Disabled | TikTok click ID |
| `ttcid` | ☐ Disabled | TikTok campaign ID |
| `twclid` | ☐ Disabled | X (Twitter) click ID |
| `li_fat_id` | ☐ Disabled | LinkedIn first-party ad tracking ID |
| `ScCID` / `ScCid` | ☐ Disabled | Snapchat click ID |
| `rdt_cid` | ☐ Disabled | Reddit click ID |
| `srsltid` | ☐ Disabled | Google Shopping — surface result listing ID |
| `utm_id` | ☐ Disabled | UTM campaign ID (GA4 extended) |
| `utm_source_platform` | ☐ Disabled | UTM source platform (GA4 extended) |
| `utm_marketing_tactic` | ☐ Disabled | UTM marketing tactic (GA4 extended) |
| `utm_creative_format` | ☐ Disabled | UTM creative format (GA4 extended) |
| `epic` | ☐ Disabled | Custom campaign / partner click ID |
| `cid` | ☐ Disabled | Custom contact or campaign ID |

---

## Custom Templates Included

| Template | Author | Purpose |
|---|---|---|
| `Write URL query strings 2 cookies` | drewspen | Reads URL query parameters and writes session or persistent cookies via GTM sandbox APIs. No CSP implications. |
| `Cookie Rewriter` | drewspen | Rewrites a list of cookies with a refreshed expiry date for rolling window behaviour. |
| `If Else If — Advanced Lookup Table` | Sublimetrix | Conditional value selection used by routing and coalesce variables. |
| `Timestamp` | Sublimetrix | Reads Unix timestamps from GA4 cookies for shared event parameters. |
| `Get Root Domain` | — | Extracts the registrable root domain for cross-subdomain cookie scoping. |

---

## Environment-Aware Stream Routing

The container ships with a two-tier routing chain identical to other recipes in this series:

**Tier 1 — URL hostname pattern** (`Measurement Stream RegEx Lookup`): routes non-production hostnames (`dev`, `uat`, `qa`, `orig`, `stg`, `stage`, `staging`, `int`, `local`, `admin`, `aem`) and the GTM Preview appspot.com domain to the development stream.

**Tier 2 — GTM Environment Name** (`Environment Stream ID`): routes live environments (`^liv.*`) to production and pre-production environments (`^(pre|de|st|ua|qa|te).*`) to development. Default is production.

Update `Measurement Stream ID Production` and `Measurement Stream ID Development` Constant variables with your actual GA4 Measurement IDs.

---

## How to Import

1. Download `utm-and-url-query-string-2-cookies-recipe.json` from the [Google Drive link](https://drive.google.com/file/d/1YOIJseTGI_XH5O-HPkHOEcbHK01bQCpm/view?usp=drivesdk).
2. In GTM, go to **Admin → Import Container**.
3. Select the downloaded JSON file.
4. Choose **Merge** (not Overwrite).
5. Update `Measurement Stream ID Production` and `Measurement Stream ID Development` to your GA4 Measurement IDs.
6. Enable any additional query string parameters (click IDs, custom fields) in both Write tags.
7. Verify `{{Root Domain}}` resolves correctly in GTM Preview mode on your site.
8. Register the six Shared Event Settings UTM parameters as custom event-scoped dimensions in GA4 under **Admin → Custom definitions → Custom dimensions**.
9. Test with a URL containing UTM parameters: confirm cookies are written, verify they persist across page navigations, and confirm the values appear in GA4 event parameters in Tag Assistant.

> **Consent note:** Persistent marketing attribution cookies may require user consent under GDPR, CCPA, or equivalent laws. Gate both Write tags behind a consent-aware trigger if your site uses a Consent Management Platform.

---

## What You Can Do With It

- Attach first-touch UTM values to CRM lead records via form hidden fields — connect campaign spend to closed revenue without server-side infrastructure
- Compare `e_cur_utm_campaign` (session UTM) against `e_1st_utm_campaign` (first-touch) in GA4 Explorations to identify multi-campaign conversion journeys
- Enable `gclid`, `fbclid`, or `msclkid` to persist ad platform click IDs for enhanced conversions and offline conversion uploads
- Build BigQuery attribution models joining on `e_1st_utm_source` without rebuilding the tracking implementation
- Use the `_1` / `_u` cookie suffix convention to maintain first-touch and last-touch attribution in parallel within a single container

---

## Related Recipes

- [Universal Click Tracking Recipe](../universal-click-tracking-recipe/) — DOM traversal coalesce variables for deep component click identification, without custom JavaScript or new CSP hashes.
- [Browser Viewport Measurement Recipe](../browser-viewport-measurement-recipe/) — Event-level viewport dimensions on every GA4 hit via JavaScript Variable type and Shared Event Settings.

---

## Contributing

Pull requests and issues welcome. If you enable additional query string parameters, extend the cookie naming convention, or add new attribution tracks, please open a PR with an updated parameter table and cookie inventory in the README.

## License

MIT
