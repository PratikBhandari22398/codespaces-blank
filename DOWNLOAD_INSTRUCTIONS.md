# Downloading the offline-demo artifacts

If this environment cannot push ZIP/binary files, use the generated export files under `export/`.

Generated files:

| File | Purpose |
|---|---|
| `export/foxiz-offline-demo-artifacts.tar.gz` | Downloadable archive containing `OFFLINE_DEMO_README.md`, `SECURITY_REVIEW.md`, `foxiz.zip`, `foxiz-child.zip`, and `foxiz-core.zip`. |
| `export/foxiz-offline-demo-artifacts.sha256` | SHA-256 checksum for the tarball. |
| `export/foxiz-offline-demo-changes.bundle` | Git bundle containing the committed changes from this branch. |
| `export/foxiz-offline-demo.patch` | Binary-capable `git format-patch` output for the committed changes. |

The tarball checksum generated in this environment is:

```text
6daa97a2263752ee21607cd79e34c9c5a65dacf40cccd82112f5d16435242433  /workspace/new/export/foxiz-offline-demo-artifacts.tar.gz
```

## Option A: download the ready-to-use artifact tarball

Download this file from the environment file browser if available:

```text
/workspace/new/export/foxiz-offline-demo-artifacts.tar.gz
```

After downloading, extract it locally:

```bash
tar -xzf foxiz-offline-demo-artifacts.tar.gz
```

Then upload `foxiz.zip` directly through WordPress.

## Option B: use the git bundle

Download this file:

```text
/workspace/new/export/foxiz-offline-demo-changes.bundle
```

Then apply it locally:

```bash
git clone foxiz-offline-demo-changes.bundle foxiz-offline-demo
cd foxiz-offline-demo
git log --oneline --max-count=5
```

## Option C: regenerate the export files

From the repo root in this environment:

```bash
mkdir -p export
tar -czf export/foxiz-offline-demo-artifacts.tar.gz OFFLINE_DEMO_README.md SECURITY_REVIEW.md foxiz.zip foxiz-child.zip foxiz-core.zip
sha256sum export/foxiz-offline-demo-artifacts.tar.gz > export/foxiz-offline-demo-artifacts.sha256
git bundle create export/foxiz-offline-demo-changes.bundle HEAD ^35625ec
git format-patch --stdout --binary 35625ec..HEAD > export/foxiz-offline-demo.patch
```

`export/` is git-ignored so these transfer artifacts do not make future pushes larger.
