# Immich Server Google Maps Patch

This repository builds a custom `immich-server` image that changes the asset detail panel's external map link from OpenStreetMap to Google Maps.

It is designed to stay close to upstream:

- clone the latest upstream Immich release tag
- apply a tiny patch
- build the official `server/Dockerfile`
- push the image to GHCR

## Published Images

The workflow publishes these tags:

- `ghcr.io/vision0141/immich-server-googlemaps:vX.Y.Z-googlemaps`
- `ghcr.io/vision0141/immich-server-googlemaps:vX-googlemaps`

Examples:

- `ghcr.io/vision0141/immich-server-googlemaps:v2.6.3-googlemaps`
- `ghcr.io/vision0141/immich-server-googlemaps:v2-googlemaps`

Recommended for Watchtower:

- use `v2-googlemaps` for automatic major-version tracking
- pin to `vX.Y.Z-googlemaps` if you want manual upgrades

## What Is Patched

Upstream file:

- `web/src/lib/components/asset-viewer/detail-panel.svelte`

Change:

- OpenStreetMap URL -> Google Maps URL
- link label -> `Google Maps`

## GitHub Actions

The workflow runs:

- manually via `workflow_dispatch`
- daily via `schedule`

Behavior:

- resolve the latest upstream Immich release tag
- skip the build if that exact version tag already exists in GHCR
- build and push exact + rolling tags when a new upstream version appears

## Compose Example

Replace only the server image in your Immich compose file:

```yaml
services:
  immich-server:
    image: ghcr.io/vision0141/immich-server-googlemaps:v2-googlemaps
```

Leave `immich-machine-learning` and the database containers unchanged.

## Notes

- The first GHCR package push may create the package as private depending on GitHub package settings.
- If you want Watchtower to pull without registry credentials, set the GHCR package visibility to public after the first push.
