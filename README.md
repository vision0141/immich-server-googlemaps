# Immich Server Google Maps Patch

## 日本語

このリポジトリは、Immich のアセット詳細パネルにある外部マップリンクを `OpenStreetMap` から `Google Maps` に変更した custom `immich-server` image をビルドします。

upstream への追従をしやすくするため、構成は最小差分にしています。

- 最新の upstream Immich release tag を取得
- 小さな patch を適用
- 公式 `server/Dockerfile` をそのまま build
- GHCR へ push

### 公開される image tag

workflow は次の tag を push します。

- `ghcr.io/vision0141/immich-server-googlemaps:vX.Y.Z-googlemaps`
- `ghcr.io/vision0141/immich-server-googlemaps:vX-googlemaps`

例:

- `ghcr.io/vision0141/immich-server-googlemaps:v2.6.3-googlemaps`
- `ghcr.io/vision0141/immich-server-googlemaps:v2-googlemaps`

Watchtower 用の推奨:

- 自動追従したい場合は `v2-googlemaps`
- 手動で更新管理したい場合は `vX.Y.Z-googlemaps`

### patch 内容

変更対象 upstream file:

- `web/src/lib/components/asset-viewer/detail-panel.svelte`

変更内容:

- OpenStreetMap URL -> Google Maps URL
- リンク表示名 -> `Google Maps`

### GitHub Actions

workflow の実行方法:

- `workflow_dispatch` による手動実行
- `schedule` による毎日実行

動作内容:

- 最新の upstream Immich release tag を取得
- その version の image が GHCR に既にあれば build を skip
- 新しい upstream version が出ていれば exact tag と rolling tag を build/push

### Compose 設定例

Immich の compose では `immich-server` の image だけ差し替えます。

```yaml
services:
  immich-server:
    image: ghcr.io/vision0141/immich-server-googlemaps:v2-googlemaps
```

`immich-machine-learning` や database 関連の container はそのままで構いません。

### 補足

- 最初の GHCR push 後、package visibility が private で作られる場合があります。
- Watchtower から認証なしで pull したい場合は、初回 push 後に GHCR package を public に変更してください。

## English

This repository builds a custom `immich-server` image that changes the asset detail panel's external map link from OpenStreetMap to Google Maps.

It is designed to stay close to upstream:

- clone the latest upstream Immich release tag
- apply a tiny patch
- build the official `server/Dockerfile`
- push the image to GHCR

### Published images

The workflow publishes these tags:

- `ghcr.io/vision0141/immich-server-googlemaps:vX.Y.Z-googlemaps`
- `ghcr.io/vision0141/immich-server-googlemaps:vX-googlemaps`

Examples:

- `ghcr.io/vision0141/immich-server-googlemaps:v2.6.3-googlemaps`
- `ghcr.io/vision0141/immich-server-googlemaps:v2-googlemaps`

Recommended for Watchtower:

- use `v2-googlemaps` for automatic major-version tracking
- pin to `vX.Y.Z-googlemaps` if you want manual upgrades

### What is patched

Upstream file:

- `web/src/lib/components/asset-viewer/detail-panel.svelte`

Change:

- OpenStreetMap URL -> Google Maps URL
- link label -> `Google Maps`

### GitHub Actions

The workflow runs:

- manually via `workflow_dispatch`
- daily via `schedule`

Behavior:

- resolve the latest upstream Immich release tag
- skip the build if that exact version tag already exists in GHCR
- build and push exact + rolling tags when a new upstream version appears

### Compose example

Replace only the server image in your Immich compose file:

```yaml
services:
  immich-server:
    image: ghcr.io/vision0141/immich-server-googlemaps:v2-googlemaps
```

Leave `immich-machine-learning` and the database containers unchanged.

### Notes

- The first GHCR package push may create the package as private depending on GitHub package settings.
- If you want Watchtower to pull without registry credentials, set the GHCR package visibility to public after the first push.
