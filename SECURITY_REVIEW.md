# Security Review: `foxiz_v2.7.7_package.zip`

## Scope and method

Reviewed the WordPress theme archive committed as `foxiz_v2.7.7_package.zip` without executing any PHP from the archive.

The package contains:

| Item | SHA-256 |
|---|---|
| `foxiz_v2.7.7_package.zip` | `a8476558e594a803f23031a59dbc76fcef45885950cf9240b0e5fe0506181458` |
| nested `foxiz.zip` | `512b83cea3e4b3db762bc357f66b5a597df28e87facedbf659e36bcb2e7ca515` |
| nested `foxiz-child.zip` | `7981ebbd3360100907b29ca80d4b4f3995dbf5eba1bf5d1a41d8a5f2d8b640d9` |
| nested `foxiz/plugins/foxiz-core.zip` | `6622432ff5bfd4e932e531ad05b4066a323382006d6bd3a3e0111b5794ba5dc4` |

After extracting nested archives, the reviewed theme/plugin tree contained 499 PHP files, 1 remaining nested ZIP, and no obvious native executable or shell-script payloads (`.exe`, `.dll`, `.so`, `.bin`, `.sh`). Static scans searched for common WordPress malware indicators including `eval(`, `base64_decode(`, `gzinflate(`, `gzuncompress(`, `str_rot13(`, `shell_exec(`, `exec(`, `passthru(`, `system(`, `popen(`, `proc_open(`, `assert(`, `create_function(`, `/e` regex evaluation, `curl_exec(`, `file_put_contents(`, `fwrite(`, `chmod(`, `move_uploaded_file(`, `unserialize(`, `hex2bin(`, and `pack(`.

`clamscan` and `yara` were not available in this container, so this is a manual/static source review rather than an antivirus-engine verdict.

## Verdict

**Do not install this ZIP on a production WordPress site.**

I did not find a classic encoded PHP web shell in the extracted theme/plugin code during static review: no `base64_decode`, `gzinflate`, `gzuncompress`, or `str_rot13` hits were present, and the suspicious PHP execution primitives searched above did not show an obvious obfuscated payload.

However, the theme is visibly modified/nulled and contains high-risk behavior in `foxiz/functions.php`:

1. It forcibly writes license/activation options into WordPress at load time.
2. It redirects ThemeRuby API demo/import requests away from the vendor domain to `wordpressnull.org` over plain HTTP.
3. It disables SSL verification for those remote HTTP requests.
4. It blocks other matching validation calls with a forged `403` response.

That combination is enough to classify the archive as untrusted. Even if no encoded backdoor is present in the currently committed bytes, installing it would give the package a remote, unauthenticated supply-chain path for demo/import content from a non-vendor domain.

## High-risk evidence

### Forced license/activation tampering

The theme's main `functions.php` writes activation-related WordPress options immediately when the file loads:

```php
update_option( 'foxiz_license_id', [
    'is_activated' => 1,
    'purchase_code' => '********-****-****-****-************'
] );
update_option( '_ruby_validated', '' );
update_option('_licfoxiz_license_id', ['licensed' => true] );
set_site_transient('_licfoxiz_license_id', true);
```

It also writes fake API key status and marks the theme active for another 30 days:

```php
update_option( 'ruby_api_keys', [
    'expiration' => '__foxiz_expiration',
    'activation' => '__foxiz_activation',
] );
update_option( '__foxiz_expiration', strtotime('+30 days') );
update_option( '__foxiz_activation', 'active' );
```

This is not normal behavior for a legitimate, vendor-distributed theme package.

### Vendor API interception and non-vendor remote content

The same file fetches demo metadata from `wordpressnull.org` over unencrypted HTTP with SSL verification disabled:

```php
$response = wp_remote_get(
    "http://wordpressnull.org/foxiz/demos.json",
    [ 'sslverify' => false, 'timeout' => 30 ]
);
```

It then installs an HTTP filter that intercepts requests intended for `https://api.themeruby.com/` and substitutes responses from `wordpressnull.org`:

```php
add_filter( 'pre_http_request', function( $pre, $post_args, $url ) {
    if ( strpos( $url, 'https://api.themeruby.com/' ) !== false ) {
```

For demo-list requests, it again returns `http://wordpressnull.org/foxiz/demos.json`:

```php
$response = wp_remote_get(
    "http://wordpressnull.org/foxiz/demos.json",
    [ 'sslverify' => false, 'timeout' => 60 ]
);
```

For import requests, it dynamically builds a URL to `wordpressnull.org` from request query parameters:

```php
$ext = in_array( $query_args['data'], ['content', 'pages'] ) ? '.xml' : '.json';
$response = wp_remote_get(
    "http://wordpressnull.org/foxiz/demos/{$query_args['demo']}/{$query_args['data']}{$ext}",
    [ 'sslverify' => false, 'timeout' => 300 ]
);
```

This allows the installed theme to pull import/demo payloads from a third-party nulled-theme domain rather than the official vendor endpoint.

## Other observations

- The extracted package includes a bundled `foxiz-core.zip` plugin under the theme's `plugins/` directory. That archive was also extracted and included in the static review.
- The scan found normal-looking WordPress/Redux/importer code that uses remote HTTP requests, filesystem APIs, `maybe_unserialize`, and `chmod`. Those can be legitimate in a WordPress import/admin framework, but they increase the risk if the package source is already untrusted.
- I found no direct evidence in the static scan of common encoded PHP payloads such as `base64_decode` + `eval`, `gzinflate`, `gzuncompress`, or `str_rot13`.
- Static review cannot prove absence of malware. A malicious payload could be fetched later from a remote server, hidden in media/import data, triggered conditionally, or introduced by an update/download flow.


## Offline demo artifacts created

The repository now provides direct-upload ZIP artifacts at the top level instead of the original all-in-one package ZIPs:

| ZIP | Purpose | SHA-256 |
|---|---|---|
| `foxiz.zip` | Parent theme ZIP to upload directly in WordPress | `5effc2a66d0d5d0f5a3c6785ec89ee4fe0b61ec7360dda9c561f8a58520249c6` |
| `foxiz-child.zip` | Optional child theme ZIP | `7981ebbd3360100907b29ca80d4b4f3995dbf5eba1bf5d1a41d8a5f2d8b640d9` |
| `foxiz-core.zip` | Companion plugin ZIP, also embedded in `foxiz.zip` at `foxiz/plugins/foxiz-core.zip` | `9188654be882fe255c981e178d7a2394d3a0aee8fbe98ce60addfd98cdf0b82b` |

The original untrusted `foxiz_v2.7.7_package.zip` and intermediate `foxiz_v2.7.7_package.sanitized.zip` package ZIPs were removed from the repository to avoid accidental installation of the unsafe package format.

The direct-upload `foxiz.zip` remains based on the sanitized theme where the injected top-of-file `foxiz/functions.php` block was removed. The rebuilt embedded `foxiz-core.zip` also includes offline-demo hardening: license registration/deregistration AJAX handlers are not registered, token-refresh hooks are not scheduled, vendor theme/plugin update filters are not registered, the remote demo importer is not auto-loaded from the admin bootstrap, vendor API helper calls to `https://api.themeruby.com` return a local error, and the no-license dashboard form was replaced by an Offline Demo Mode explanation.

These changes do not fake activation, write a license, unlock paid/license-gated features, or bypass ThemeRuby licensing. They only remove broken license UI/nags and disable remote vendor/demo/update paths for a local disposable demo site.

## Recommendation

For safest use, do not install the original package from the internet. If you still need to test this theme on a disposable demo WordPress site, upload the top-level `foxiz.zip` from this repository directly through WordPress and keep the site isolated. If you need the Foxiz theme for anything beyond throwaway testing, obtain a clean copy directly from the official vendor/marketplace account, verify its checksum if available, and install it on a staging site first. If the original ZIP was already installed anywhere, assume the site is compromised until checked: remove the theme/plugin, rotate WordPress admin and database credentials, audit admin users, inspect `wp-content/uploads` for PHP files, and review web server/PHP logs for unexpected requests.

## Commands run

```sh
find .. -name AGENTS.md -print
find . -maxdepth 3 -type f -printf '%p\n' | sed 's#^./##' | sort | head -200
python3 - <<'PY'
import zipfile
z='foxiz_v2.7.7_package.zip'
with zipfile.ZipFile(z) as f:
    infos=f.infolist()
    print('files',len(infos),'size',sum(i.file_size for i in infos),'compressed',sum(i.compress_size for i in infos))
    for i in infos[:80]: print(i.filename, i.file_size)
PY
mkdir -p /tmp/theme_audit && rm -rf /tmp/theme_audit/*
unzip -q foxiz_v2.7.7_package.zip -d /tmp/theme_audit/package
unzip -q /tmp/theme_audit/package/foxiz.zip -d /tmp/theme_audit/foxiz
unzip -q /tmp/theme_audit/package/foxiz-child.zip -d /tmp/theme_audit/child
unzip -q /tmp/theme_audit/foxiz/foxiz/plugins/foxiz-core.zip -d /tmp/theme_audit/core
rg -n --hidden -S "(eval\\s*\\(|base64_decode\\s*\\(|gzinflate\\s*\\(|str_rot13\\s*\\(|shell_exec\\s*\\(|exec\\s*\\(|passthru\\s*\\(|system\\s*\\(|popen\\s*\\(|proc_open\\s*\\(|assert\\s*\\(|create_function\\s*\\(|preg_replace\\s*\\(.*/e|wp_remote_get\\s*\\(|wp_remote_post\\s*\\(|curl_exec\\s*\\(|file_put_contents\\s*\\(|fwrite\\s*\\(|chmod\\s*\\(|move_uploaded_file\\s*\\(|unserialize\\s*\\(|gzuncompress\\s*\\(|hex2bin\\s*\\(|pack\\s*\\()" /tmp/theme_audit/core /tmp/theme_audit/foxiz/foxiz --glob '!*.pot' --glob '!*.css' --glob '!*.min.js' --glob '!*.map'
sha256sum foxiz_v2.7.7_package.zip /tmp/theme_audit/package/foxiz.zip /tmp/theme_audit/package/foxiz-child.zip /tmp/theme_audit/foxiz/foxiz/plugins/foxiz-core.zip
clamscan --version || true
yara --version || true
```
