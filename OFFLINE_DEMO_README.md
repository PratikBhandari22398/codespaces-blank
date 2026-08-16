# Foxiz Offline Demo Build

This repository now keeps only direct-upload ZIP artifacts at the top level:

| ZIP | Purpose | SHA-256 |
|---|---|---|
| `foxiz.zip` | Parent theme ZIP to upload through **WordPress Admin > Appearance > Themes > Add New > Upload Theme** | `5effc2a66d0d5d0f5a3c6785ec89ee4fe0b61ec7360dda9c561f8a58520249c6` |
| `foxiz-child.zip` | Optional child theme ZIP to upload after the parent theme | `7981ebbd3360100907b29ca80d4b4f3995dbf5eba1bf5d1a41d8a5f2d8b640d9` |
| `foxiz-core.zip` | Bundled companion plugin ZIP, also embedded at `foxiz/plugins/foxiz-core.zip` inside `foxiz.zip` | `9188654be882fe255c981e178d7a2394d3a0aee8fbe98ce60addfd98cdf0b82b` |

The original untrusted package ZIPs were removed from the repository so you do not accidentally download or install them.

## What changed for offline/local demo use

The direct-upload `foxiz.zip` is rebuilt from the sanitized theme and includes an updated `foxiz-core.zip` plugin with these offline-demo safety changes:

- license registration and deregistration AJAX handlers are not registered;
- token renewal hooks and scheduled token refresh are not registered;
- vendor theme/plugin update filters are not registered;
- the remote demo importer is not auto-loaded from the admin bootstrap;
- outgoing requests to `https://api.themeruby.com` through the admin API helper return a local error instead of making a network call;
- the no-license dashboard screen was replaced with an **Offline Demo Mode** explanation instead of a purchase-code form.

These changes do **not** fake activation, write a license, unlock paid/license-gated features, or bypass ThemeRuby licensing. They only remove broken/nagging license UI and disable remote vendor/demo/update paths for a local/disposable demo site.

## Suggested install order

1. Upload and activate `foxiz.zip` in WordPress.
2. If WordPress prompts for the companion plugin, install the bundled plugin. You can also upload `foxiz-core.zip` manually from **Plugins > Add New > Upload Plugin**.
3. Optionally upload and activate `foxiz-child.zip`.
4. Build demo content manually or import your own trusted local content. Remote prebuilt template/demo import is intentionally disabled.

## Remaining risk

This is still derived from an untrusted third-party package. Use it only on a disposable local/demo WordPress install with no production credentials, no customer data, and no valuable API keys.
