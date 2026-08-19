# yoDEV Activation Redirect

A Discourse theme component that redirects **homepage-originated signups** from the post-activation landing page back to the yoDEV homepage with `?activated=1`, so the homepage can complete OIDC sign-in and route the user onto `/plan-select`.

People who signed up on the community directly are left alone.

## Why

The yoDEV homepage signup flow uses `/api/signup` to create accounts via Discourse's `/users.json` API directly, which means the user's browser never visits Discourse via the SSO authorize endpoint before clicking the activation link. Without that visit, no `return_sso_url` cookie is set on Discourse's domain — so when activation completes, Discourse defaults to its own homepage instead of the yoDEV homepage. This component closes that gap from the Discourse side.

The homepage already detects `?activated=1` on `/` and auto-starts OIDC, so the user lands on `/plan-select` without further interaction.

### Why it only fires for some signups (YOD-497)

The community is a signup door in its own right. Someone who creates an account on the forum wants the forum — bouncing them to a plan chooser offering "free" or $15/month, with no sight of the homepage, the pricing page or any promotion, and no obvious way back, is a bad first five minutes.

The original design discriminated with a per-signup User Field holding the destination URL, falling back to a global setting when it was empty. On production that field was never populated, so **every** activation hit the fallback and every new account was redirected regardless of origin. That fallback is gone.

Origin is now carried by **group membership**: the homepage adds each account it creates to a Discourse group, and this component redirects only members. Chosen over the user field for two reasons — user fields render on Discourse's own signup form and can be self-asserted by whoever fills them in (YOD-487), so a field lets a direct signup forge its origin; and group membership works when the user signs up on a desktop and opens the activation email on their phone, which the localStorage handoff below does not.

## How it works

On the Discourse instance this was built against (v2026.5), clicking the "Activate Account" button navigates the user **directly to `/`** in one step — there's no separate `/u/activate-account/<token>` welcome page that we can hook into. So the component uses a localStorage-flag handoff:

1. **On the pre-activation page** (`/u/activate-account/<token>`): the script hooks the click on the "Activate Account" button (or its containing form). When clicked, it writes a timestamped flag to `localStorage`.
2. **On every subsequent page load** (script runs on every Discourse page that uses the theme): it checks `localStorage` for the flag. If present and less than 5 minutes old, it consumes the flag.
3. **It then checks origin**: `/session/current.json` is read for the user's groups. Only members of `homepage_signup_group` are redirected to the configured homepage URL with `?activated=1` appended. Everyone else stays where they are.
4. The 5-minute expiry guards against a forgotten flag bouncing the user on a later, unrelated visit.

The origin check **fails closed**: any error reading the session, or any doubt about membership, leaves the user on the community. Redirecting on a failed check would reinstate YOD-497 for everyone the moment that call breaks.

Settings are passed from theme settings to the script via SCSS-generated CSS custom properties (`<script type="text/discourse-plugin">` with `settings.X` was found not to execute on this Discourse setup, so the script reads values from `getComputedStyle` of `:root`).

## Installation

1. Push this repo to a GitHub remote (or use the existing one: `gwillis-inovacon/discourse-yodev-activation-redirect`).
2. In Discourse admin → **Customize → Components → Install** → from a git repository → paste the repo URL.
3. Add the component to whichever themes are active on the instance (Discourse admin → Customize → Themes → click the active theme → Components → Add component).
4. Open the component's settings (the gear icon) and set:
   - **homepage_url** — full URL of the **yoDEV homepage app** (NOT the Discourse host). For dev: whatever Railway service hosts the homepage (e.g., `https://yodev-homepage-new-production.up.railway.app`). For prod: `https://yodev.dev`.
   - **homepage_signup_group** — leave at `homepage_signup` unless you named the group something else.
5. Save.
6. Create the group — see **One-time setup: the origin group** below. Until it exists and the homepage is deployed with the tagging change, nothing is redirected.

## Settings

| Setting | Default | Description |
|---|---|---|
| `homepage_signup_group` | `homepage_signup` | **The origin gate.** Name of the Discourse group the homepage app adds its own signups to. Only members are redirected. Set blank to redirect every activation regardless of origin — the pre-YOD-497 behaviour, which is the bug; only useful for a Discourse instance where the homepage genuinely is the only signup door. |
| `homepage_url` | _(empty)_ | Destination. Full URL of the yoDEV homepage app (NOT the Discourse host); the component appends `/?activated=1`. Blank disables the redirect entirely. |
| `redirect_delay_ms` | `0` | Milliseconds to wait after detecting the activation flag before redirecting. Default `0` = immediate. Discourse's default post-activation behavior may navigate the user away within ~1s of landing on `/`, so any non-zero delay risks losing the race. |
| `watch_timeout_ms` | `60000` | Maximum time the script polls for the Activate Account button to appear on the pre-activation page before giving up. Defaults to 60s to allow for slow Ember route rendering. |

## One-time setup: the origin group

Without this the component redirects nobody, because no account is ever a member.

1. **Create the group** at Admin → Groups → New. Name it `homepage_signup` (or whatever you set `homepage_signup_group` to).
2. ⚠️ **Set its visibility to "Logged-on users" (`visibility_level` 2).** Discourse omits groups the requester cannot see from `current_user.groups`, so a more private group reads back as "nobody is a member" and silently disables the redirect for everyone — with no error anywhere. This bit the tier groups before; it is not specific to this component.
3. **Owner/membership settings**: nobody needs to request or join it — the homepage adds members via the admin API. Leave "Allow users to join freely" off.
4. The homepage does the rest: `tagHomepageSignup()` in `server-shared.ts` adds each account it creates, best-effort, right after `/api/signup` succeeds.

Verify with a real signup on each door and watch the browser console for `[yoDEV-activation-redirect] groups: [...]`.

### Multi-environment note

The retired user-field design could send different signups to different homepage URLs from one Discourse instance. Group membership carries origin only, not destination, so every member goes to the single `homepage_url`. If one Discourse ever needs to serve two homepages again, use one group per environment and map group → URL — do not bring back a user-field carrying the URL, which is a self-assertable redirect target (YOD-489).

## Console output

The script logs all decisions to the browser console with the prefix `[yoDEV-activation-redirect]`. Useful for diagnosing flow issues — search the console for that prefix.

## Compatibility

Built and tested against Discourse `v2026.5.0-latest` with Ember `v6.10.1`. Older Discourse versions that DO render a separate `/u/activate-account/<token>` welcome page (with a `#welcome-button`) would need a different approach (poll for `#welcome-button` instead of using the localStorage handoff).

## Removal

Disable or uninstall the component in Discourse admin → Customize → Components. The localStorage flag is short-lived (5 min expiry) and will clean itself up; no persistent state to manually remove.
