# Immich Server Google Maps Patch

## 日本語

このリポジトリは、Immich に対する小さな custom patch を含む `immich-server` image をビルドします。

現在含まれる主な修正:

- アセット詳細パネルの外部マップリンクを `OpenStreetMap` から `Google Maps` に変更
  表示は `place/{DMS}/@{lat},{lon},450m/data=...` 形式で、共有リンクに近い見え方を優先
- Intel `QSV` のレート制御を調整し、`Auto` と `maxBitrate` の組み合わせで bitrate mode に切り替わるように修正
- 回転メタデータ付きの縦動画が `608x1080` のように潰れないよう、transcode 時の scaling 判定を修正
- 画像設定のプレビュー解像度に `Original` を追加し、画像・動画プレビューを元解像度で生成できるように修正
- Web非対応画像の `fullsize` 生成で、`IMG_3593.HEIC` のような大文字拡張子も通常ジョブで拾えるように修正
- WebUI の写真ビューアで元画像ロード後に低解像度レイヤーが残る問題は、v3 upstream 側の新しい `AdaptiveImage` 実装で解消済みのため、旧reraster patchは外しています

`v3.0.1` 公式ソース確認時点では、上記のうち旧reraster patch以外は upstream に取り込まれていないため、このカスタムビルドで維持します。

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

- `ghcr.io/vision0141/immich-server-googlemaps:v3.0.1-googlemaps`
- `ghcr.io/vision0141/immich-server-googlemaps:v3-googlemaps`

Watchtower 用の推奨:

- 自動追従したい場合は `v3-googlemaps`
- 手動で更新管理したい場合は `vX.Y.Z-googlemaps`

### patch 内容

変更対象 upstream file:

- `web/src/lib/components/asset-viewer/DetailPanel.svelte`
- `web/src/routes/admin/system-settings/ImageSettings.svelte`
- `server/src/dtos/system-config.dto.ts`
- `server/src/services/media.service.ts`
- `server/src/repositories/asset-job.repository.ts`
- `server/src/utils/media.ts`

変更内容:

- OpenStreetMap URL -> Google Maps shared-style URL (`place/{DMS}/@{lat},{lon},450m/data=!3m1!1e3!...`)
- リンク表示名 -> `Google Maps`
- プレビュー解像度の `Original` 選択肢を `0` として保存し、画像では resize 無効、動画では既存の `original` target resolution として扱う
- `fullsize` 未生成のWeb非対応画像を抽出するとき、`originalFileName` の拡張子判定をcase-insensitiveにする
- QSV の `maxBitrate` 指定時は `Auto` で bitrate mode を使う
- `ICQ` / `CQP` を明示した場合は quality mode を固定する
- 回転 metadata を保持したまま transcode する縦動画で、stored orientation に合わせて scaling する
- 旧写真ビューアreraster patchはv3では不要なため適用しない

### Intel QSV 補足

この image には `AV1 QSV` 向けのレート制御 patch が含まれます。

- `cqMode = auto` かつ `maxBitrate` なし: `ICQ`
- `cqMode = auto` かつ `maxBitrate` あり: `VBR`
- `cqMode = icq` / `cqp`: 指定 mode を優先

実機検証では、`AV1 QSV` の `QVBR` はこの環境では実用にならず、`Auto + maxBitrate` は `VBR` を使う方針にしています。

また、`QSV` の hardware decode は環境によって不安定な場合があります。rate control patch は encode side の挙動修正であり、hardware decode の安定化 patch ではありません。

### GitHub Actions

workflow の実行方法:

- `workflow_dispatch` による手動実行
- `schedule` による毎時実行

動作内容:

- 定期実行では GitHub API の `releases/latest` から最新のstable release tagを取得
- RC / prerelease は定期実行では拾わず、手動実行時に `immich_ref` へ `v3.0.0-rc.4` のようなtagを指定した場合だけビルド
- その version の image が GHCR に既にあれば build を skip
- 新しい upstream version が出ていれば exact tag と rolling tag を build/push

### Compose 設定例

Immich の compose では `immich-server` の image だけ差し替えます。

```yaml
services:
  immich-server:
    image: ghcr.io/vision0141/immich-server-googlemaps:v3-googlemaps
```

`immich-machine-learning` や database 関連の container はそのままで構いません。

### 補足

- 最初の GHCR push 後、package visibility が private で作られる場合があります。
- Watchtower から認証なしで pull したい場合は、初回 push 後に GHCR package を public に変更してください。

## English

This repository builds a custom `immich-server` image with a small set of patches for Immich.

Current patches include:

- changing the asset detail panel's external map link from OpenStreetMap to Google Maps
  using a shared-style `place/{DMS}/@{lat},{lon},450m/data=!3m1!1e3!...` URL for a more stable map view
- adjusting Intel `QSV` rate-control behavior so `Auto + maxBitrate` switches to bitrate mode
- fixing portrait videos with rotation metadata so transcoding no longer produces squashed `608x1080`-style output
- adding an `Original` preview-resolution option for image settings so image and video previews can be generated at source resolution
- making fullsize thumbnail jobs pick up web-unsupported images with uppercase filename extensions such as `IMG_3593.HEIC`
- the former WebUI photo-viewer reraster workaround is no longer applied because the v3 upstream `AdaptiveImage` implementation already addresses the stale low-resolution layer behavior

As of the `v3.0.1` upstream source check, only the old reraster workaround is covered upstream. The other items are still kept in this custom build.

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

- `ghcr.io/vision0141/immich-server-googlemaps:v3.0.1-googlemaps`
- `ghcr.io/vision0141/immich-server-googlemaps:v3-googlemaps`

Recommended for Watchtower:

- use `v3-googlemaps` for automatic major-version tracking
- pin to `vX.Y.Z-googlemaps` if you want manual upgrades

### What is patched

Upstream file:

- `web/src/lib/components/asset-viewer/DetailPanel.svelte`
- `web/src/routes/admin/system-settings/ImageSettings.svelte`
- `server/src/dtos/system-config.dto.ts`
- `server/src/services/media.service.ts`
- `server/src/repositories/asset-job.repository.ts`
- `server/src/utils/media.ts`

Change:

- OpenStreetMap URL -> Google Maps shared-style URL (`place/{DMS}/@{lat},{lon},450m/data=!3m1!1e3!...`)
- link label -> `Google Maps`
- store the `Original` preview-resolution selection as `0`, disable image resizing for it, and pass video preview generation through the existing `original` target resolution
- make the missing-`fullsize` web-unsupported image filter case-insensitive for `originalFileName` extension matching
- use bitrate mode for QSV when `maxBitrate` is set and `cqMode` is `auto`
- keep explicit `ICQ` / `CQP` selections in quality mode
- scale rotated portrait videos using stored frame orientation during transcode
- the old photo-viewer reraster patch is not applied on v3 because it is no longer needed

### Intel QSV Notes

This image also includes a rate-control patch for `AV1 QSV`.

- `cqMode = auto` with no `maxBitrate`: `ICQ`
- `cqMode = auto` with `maxBitrate`: `VBR`
- `cqMode = icq` / `cqp`: keep the selected quality mode

In local testing, `AV1 QSV` `QVBR` was not usable on this stack, so `Auto + maxBitrate` intentionally uses `VBR`.

Hardware decode for `QSV` may still be unstable depending on the environment. This patch improves encode-side rate control behavior; it does not claim to fix hardware decode stability.

### GitHub Actions

The workflow runs:

- manually via `workflow_dispatch`
- hourly via `schedule`

Behavior:

- scheduled runs resolve the latest stable upstream release tag via GitHub's `releases/latest`
- RC / prerelease tags are not picked up automatically; build them only by passing a tag such as `v3.0.0-rc.4` to `immich_ref` in `workflow_dispatch`
- skip the build if that exact version tag already exists in GHCR
- build and push exact + rolling tags when a new upstream version appears

### Compose example

Replace only the server image in your Immich compose file:

```yaml
services:
  immich-server:
    image: ghcr.io/vision0141/immich-server-googlemaps:v3-googlemaps
```

Leave `immich-machine-learning` and the database containers unchanged.

### Notes

- The first GHCR package push may create the package as private depending on GitHub package settings.
- If you want Watchtower to pull without registry credentials, set the GHCR package visibility to public after the first push.
