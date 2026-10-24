# Immich Server Google Maps Patch

## 日本語

このリポジトリは、Immich に対する小さな custom patch を含む `immich-server` image をビルドします。

現在含まれる主な修正:

- アセット詳細パネルの外部マップリンクを `OpenStreetMap` から `Google Maps` に変更
  表示は `place/{DMS}/@{lat},{lon},450m/data=...` 形式で、共有リンクに近い見え方を優先。DMSは0.1秒単位に丸めてから度、分、秒へ分け、秒が `60.0` にならないようにする
- 地図の背景をGoogleの地図(Maps JavaScript API)にする。Googleが使えない時は予備のstyle.json(Googleの画像)、それも無ければ管理画面の地図スタイル(MapTiler)
  地図を1枚作る前に同じoriginの `/prism-maps/google-load` を呼び、`{ ok, key }` を受け取る(自宅環境ではCaddyがImmichのログインを確かめてPrism Chatへ中継し、Prism Chatの地図と合わせて月9,900回で断る)。断られた時、この入口が無い時、キーの拒否(`gm_authFailure`)、読込みの失敗では、同じoriginの予備 `/prism-maps/style.json`(自宅環境ではGoogleの画像の地図、Map Tiles API。その上限の後はMapTilerへ転送)を読み、それも読めなければ管理画面の地図スタイル(MapTiler)にする。作った地図はページの中で使い回し、課金の回数を増やさない
- Intel `QSV` のレート制御を調整し、`Auto` と `maxBitrate` の組み合わせで bitrate mode に切り替わるように修正
- 画像設定のプレビュー解像度に `Original` を追加し、画像・動画プレビューを元解像度で生成できるように修正
- Web非対応画像の `fullsize` 生成で、`IMG_3593.HEIC` のような大文字拡張子も通常ジョブで拾えるように修正
- OCRは、まずクラウドOCR窓口(`IMMICH_CLOUD_OCR_URL`、`IMMICH_CLOUD_OCR_TOKEN`)へassetIdを送り、窓口がImmichと同じ形の結果を返したらそれを保存する(MLのOCRは呼ばない)。窓口の答えが `keep`(読む価値のある文字が無い、月の無料枠の上限など)なら保存済みのOCRを変えずにジョブを終え、`local` の時だけMLのOCRで縮小版を読む。失敗した、時間切れ(`IMMICH_CLOUD_OCR_TIMEOUT_MS`、既定180秒)、形の違う答えの時はMLへ退避せずジョブを失敗にする(OCRの済みの印 `ocrAt` が付かないので、窓口が後でOCRキューへ戻す)。環境変数が無ければ従来どおりMLで読む。窓口は自宅環境の `DockerData/cloud-ocr-gateway`(Google Document AI、Cloud Vision、AzureのF0の無料枠の中で読む)
- アプリ(モバイル)へのOCRの同期(`AssetOcrV1`)に、自分へ共有しているパートナーの写真のOCRも含める。upstreamは自分の写真だけを送るため、パートナーの写真はWebでは枠が出てもアプリでは出ない。範囲はパートナーの資産とexifの同期と同じ(`inTimeline` を問わない)で、アプリの手元のDBの外部キーを満たす。共有アルバムだけで見える写真は、upstreamのアルバムの資産の同期が送る範囲を細かく絞っており、合わせないとアプリの同期が外部キー違反で止まるため含めない。パートナーになる前のOCRは送らないため、既存の端末はOCRの同期の記録(`session_sync_checkpoint` の `AssetOcrV1`)を消して送り直させる(`immich-ocr-partner-sync.patch`)
- WebUI の写真ビューアで元画像ロード後に低解像度レイヤーが残る問題は、v3 upstream 側の新しい `AdaptiveImage` 実装で解消済みのため、旧reraster patchは外しています
- 回転メタデータ付きの縦動画が `608x1080` のように潰れる問題は、v3.3.0 upstream の `isFrameVertical` 修正で解消済みのため、旧rotated portrait scaling patchは外しています

`v3.3.3` 公式ソース確認時点では、上記のうち外したpatch以外は upstream に取り込まれていないため、このカスタムビルドで維持します。

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
- `web/src/lib/components/shared-components/map/Map.svelte`
- `web/src/lib/utils/google-basemap.ts`(新規)
- `web/src/routes/admin/system-settings/ImageSettings.svelte`
- `server/src/dtos/config.dto.ts`
- `server/src/services/media.service.ts`
- `server/src/repositories/asset-job.repository.ts`
- `server/src/utils/media.ts`
- `server/src/services/ocr.service.ts`、`server/src/services/ocr.service.spec.ts`(`immich-cloud-ocr.patch`)
- `server/src/repositories/sync.repository.ts`、`server/src/queries/sync.repository.sql`、`server/test/medium/specs/sync/sync-asset-ocr.spec.ts`(`immich-ocr-partner-sync.patch`)

変更内容:

- Googleの地図を使う間は、MapLibreを空のスタイルにして写真の印だけを描き、下に敷いたGoogleの地図(`hybrid`)を位置と拡大(MapLibre+1段)に合わせる。回転、傾き、地球儀は止め、Googleのロゴと著作の表示は隠さない(MapLibreの縮尺は上へずらし、下端のリンクは押された位置で代わりに押す)

- OpenStreetMap URL -> Google Maps shared-style URL (`place/{DMS}/@{lat},{lon},450m/data=!3m1!1e3!...`)
- リンク表示名 -> `Google Maps`
- プレビュー解像度の `Original` 選択肢を `0` として保存し、画像では resize 無効、動画では既存の `original` target resolution として扱う
- `fullsize` 未生成のWeb非対応画像を抽出するとき、`originalFileName` の拡張子判定をcase-insensitiveにする
- QSV の `maxBitrate` 指定時は `Auto` で bitrate mode を使う
- `ICQ` / `CQP` を明示した場合は quality mode を固定する
- 旧写真ビューアreraster patchはv3では不要なため適用しない
- 旧rotated portrait scaling patchはv3.3.0以降のupstream修正で不要なため適用しない

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
- using Google Maps (Maps JavaScript API) as the map basemap when same-origin `POST /prism-maps/google-load` returns `{ ok: true, key }`; otherwise (limit reached, endpoint missing, `gm_authFailure`, load error) the configured MapLibre style (MapTiler) is used. Created Google maps are reused within the page to keep billable map loads low
- adjusting Intel `QSV` rate-control behavior so `Auto + maxBitrate` switches to bitrate mode
- adding an `Original` preview-resolution option for image settings so image and video previews can be generated at source resolution
- making fullsize thumbnail jobs pick up web-unsupported images with uppercase filename extensions such as `IMG_3593.HEIC`
- including OCR of assets shared by partners in the mobile OCR sync (`AssetOcrV1`), using the same scope as partner asset and exif sync; album-only assets are not included
- the former WebUI photo-viewer reraster workaround is no longer applied because the v3 upstream `AdaptiveImage` implementation already addresses the stale low-resolution layer behavior
- the former rotated portrait scaling patch is no longer applied because upstream v3.3.0 fixed it with `isFrameVertical`

As of the `v3.3.3` upstream source check, only the removed patches are covered upstream. The other items are still kept in this custom build.

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
- `web/src/lib/components/shared-components/map/Map.svelte`
- `web/src/lib/utils/google-basemap.ts` (new)
- `web/src/routes/admin/system-settings/ImageSettings.svelte`
- `server/src/dtos/config.dto.ts`
- `server/src/services/media.service.ts`
- `server/src/repositories/asset-job.repository.ts`
- `server/src/utils/media.ts`
- `server/src/repositories/sync.repository.ts`, `server/src/queries/sync.repository.sql`, `server/test/medium/specs/sync/sync-asset-ocr.spec.ts`

Change:

- while Google Maps is active, MapLibre uses an empty style and only draws the photo markers; the Google map (`hybrid`) underneath follows MapLibre's center and zoom (+1). Rotation, pitch and globe are disabled, and the Google logo and attribution stay visible

- OpenStreetMap URL -> Google Maps shared-style URL (`place/{DMS}/@{lat},{lon},450m/data=!3m1!1e3!...`)
- link label -> `Google Maps`
- store the `Original` preview-resolution selection as `0`, disable image resizing for it, and pass video preview generation through the existing `original` target resolution
- make the missing-`fullsize` web-unsupported image filter case-insensitive for `originalFileName` extension matching
- use bitrate mode for QSV when `maxBitrate` is set and `cqMode` is `auto`
- keep explicit `ICQ` / `CQP` selections in quality mode
- the old photo-viewer reraster patch is not applied on v3 because it is no longer needed
- the old rotated portrait scaling patch is not applied on v3.3.0 or later because upstream fixed it

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

### v3.2.0 互換性

v3.2.0 で設定 DTO が `config.dto.ts` の `AdminConfig` schema に統合されたため、Original preview の入力検証パッチを移植しました。preview のみ 0 を許可し、thumbnail の正数制約は維持します。設定 schema の回帰テストもパッチに含めています。
