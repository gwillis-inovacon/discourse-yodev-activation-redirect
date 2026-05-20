# yoDEV Activation Redirect

A Discourse theme component that redirects users from the post-activation landing page back to the yoDEV homepage with `?activated=1`, so the homepage can complete OIDC sign-in and route the user onto `/plan-select`.

## Why

The yoDEV homepage signup flow uses `/api/signup` to create accounts via Discourse's `/users.json` API directly, which means the user's browser never visits Discourse via the SSO authorize endpoint before clicking the activation link. Without that visit, no `return_sso_url` cookie is set on Discourse's domain — so when activation completes, Discourse defaults to its own homepage instead of the yoDEV homepage. This component closes that gap from the Discourse side.

The homepage already detects `?activated=1` on `/` and auto-starts OIDC, so the user lands on `/plan-select` without further interaction.

## How it works

On the Discourse instance this was built against (v2026.5), clicking the "Activate Account" button navigates the user **directly to `/`** in one step — there's no separate `/u/activate-account/<token>` welcome page that we can hook into. So the component uses a localStorage-flag handoff:

1. **On the pre-activation page** (`/u/activate-account/<token>`): the script hooks the click on the "Activate Account" button (or its containing form). When clicked, it writes a timestamped flag to `localStorage`.
2. **On every subsequent page load** (script runs on every Discourse page that uses the theme): it checks `localStorage` for the flag. If present and less than 5 minutes old, it consumes the flag and redirects to the configured homepage URL with `?activated=1` appended.
3. The 5-minute expiry guards against a forgotten flag bouncing the user on a later, unrelated visit.

Settings are passed from theme settings to the script via SCSS-generated CSS custom properties (`<script type="text/discourse-plugin">` with `settings.X` was found not to execute on this Discourse setup, so the script reads values from `getComputedStyle` of `:root`).

## Installation

1. Push this repo to a GitHub remote (or use the existing one: `gwillis-inovacon/discourse-yodev-activation-redirect`).
2. In Discourse admin → **Customize → Components → Install** → from a git repository → paste the repo URL.
3. Add the component to whichever themes are active on the instance (Discourse admin → Customize → Themes → click the active theme → Components → Add component).
4. Open the component's settings (the gear icon) and set:
   - **homepage_url** — full URL of the **yoDEV homepage app** (NOT the Discourse host). For dev: whatever Railway service hosts the homepage (e.g., `https://yodev-homepage-new-production.up.railway.app`). For prod: the production homepage URL when one exists.
5. Save.

## Settings

| Setting | Default | Description |
|---|---|---|
| `homepage_redirect_field_id` | `0` | **PRIMARY source.** Numeric ID of a Discourse User Field that stores the per-signup homepage URL. The component reads `/session/current.json` after activation and looks up `current_user.custom_fields.user_field_<id>`. Requires that field's name (`user_field_<id>`) to be in Discourse's `public_user_custom_fields` site setting so the API exposes it. If `0` or the user has no value for the field, falls back to `homepage_url`. |
| `homepage_url` | _(empty)_ | **FALLBACK.** Full URL of the yoDEV homepage app to redirect to when the per-signup field isn't set. The component appends `/?activated=1`. If both this and the per-signup field are empty, the component logs a warning and does nothing. |
| `redirect_delay_ms` | `0` | Milliseconds to wait after detecting the activation flag before redirecting. Default `0` = immediate. Discourse's default post-activation behavior may navigate the user away within ~1s of landing on `/`, so any non-zero delay risks losing the race. |
| `watch_timeout_ms` | `60000` | Maximum time the script polls for the Activate Account button to appear on the pre-activation page before giving up. Defaults to 60s to allow for slow Ember route rendering. |

## Multi-environment setup (one Discourse instance serving multiple homepage URLs)

The per-signup user field design lets a single Discourse instance redirect different signups to different homepage URLs — required when the same prod Discourse handles auth for both staging and prod (or any multi-env setup).

**One-time Discourse admin setup:**

1. **Create the User Field** at Admin → Customize → User Fields → Add User Field. Type = `text`. Required = no. Hidden = yes (admin-only — users don't need to see it). Note the numeric ID assigned to it (visible in the URL: `/admin/customize/user_fields/<id>`).
2. **Expose it via the API** at Admin → Settings → search `public user custom fields` → add `user_field_<id>` to the list. Without this, `/session/current.json` will NOT include the field even when the user has a value set.
3. **Configure the theme component** with `homepage_redirect_field_id` = `<id>`. Leave `homepage_url` blank (or set to a sensible global default for signups that don't go through your homepage app).
4. **In the signup-originating app** (e.g., yoDEV homepage), set the user field when creating the user via Discourse's `/users.json` API: `formData.append("user_fields[<id>]", "<this homepage's URL>")`. The value is what the theme reads back at activation time.

## Console output

The script logs all decisions to the browser console with the prefix `[yoDEV-activation-redirect]`. Useful for diagnosing flow issues — search the console for that prefix.

## Compatibility

Built and tested against Discourse `v2026.5.0-latest` with Ember `v6.10.1`. Older Discourse versions that DO render a separate `/u/activate-account/<token>` welcome page (with a `#welcome-button`) would need a different approach (poll for `#welcome-button` instead of using the localStorage handoff).

## Removal

Disable or uninstall the component in Discourse admin → Customize → Components. The localStorage flag is short-lived (5 min expiry) and will clean itself up; no persistent state to manually remove.
