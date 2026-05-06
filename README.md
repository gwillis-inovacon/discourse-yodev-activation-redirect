# yoDEV Activation Redirect

A Discourse theme component that redirects users from the standard Discourse account-activation success page back to the yoDEV homepage with `?activated=1`, so the homepage can complete OIDC sign-in and route the user onto `/plan-select`.

## Why

The yoDEV homepage signup flow uses `/api/signup` to create accounts via Discourse's `/users.json` API directly, which means the user's browser never visits Discourse via the SSO authorize endpoint before clicking the activation link. Without that visit, no `return_sso_url` / `destination_url` cookie is set on Discourse's domain — so when activation completes, Discourse defaults to its own homepage instead of the yoDEV homepage. This component closes that gap from the Discourse side.

The homepage already detects `?activated=1` on `/` and auto-starts OIDC, so the user lands on `/plan-select` without further interaction.

## Installation

1. Push this repo to a GitHub remote (e.g., `gwillis-inovacon/discourse-yodev-activation-redirect`).
2. In Discourse admin → **Customize → Components → Install** → from a git repository → paste the repo URL.
3. Add the component to whichever themes are active on the instance.
4. Open the component's settings (the gear icon) and set:
   - **homepage_url** — full URL of the yoDEV homepage on this Discourse instance's environment. Examples:
     - Dev Discourse (`development.yodev.dev`) → `https://development.yodev.dev` (if the homepage shares the host) or whatever the dev homepage URL is
     - Prod Discourse (`yodev.dev`) → `https://www.yodev.dev`
5. Save.

## Settings

| Setting | Default | Description |
|---|---|---|
| `homepage_url` | _(empty)_ | Full URL the activation success page should redirect to. The component appends `?activated=1`. |
| `redirect_delay_ms` | `1500` | Milliseconds to wait after the success page renders before redirecting. Long enough to see the success message, short enough not to feel stalled. |
| `watch_timeout_ms` | `10000` | Maximum time to wait for the activation success indicator to appear before giving up (so the script doesn't run forever on failure pages). |

If `homepage_url` is empty, the component does nothing — handy for staging environments where you might want to disable the redirect without uninstalling.

## How it works

The script registers an `onPageChange` callback (Discourse plugin API). When the user navigates to `/u/activate-account/<token>`:

1. Polls the DOM every 200ms looking for the standard Discourse Finish button (`#welcome-button` or the activate-account view's primary button).
2. When found, overrides the button's `href` to the configured homepage URL (so manual clicks also work).
3. Schedules a `setTimeout` to redirect to the homepage after `redirect_delay_ms`.
4. If the button doesn't appear within `watch_timeout_ms` (e.g., on a failed-activation error page), the polling gives up and no redirect happens.

## Compatibility

Targets Discourse plugin API version `1.10.0` (released circa 2023). Should work on any reasonably modern Discourse install.

## Removal

Disable or uninstall the component in Discourse admin → Customize → Components. No persistent state to clean up.
