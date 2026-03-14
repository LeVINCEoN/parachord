# Local Playlist Sync to Streaming Services

**Date:** 2026-03-14
**Status:** Design

## Problem

Locally created playlists in Parachord exist only on the device. With the upcoming Android app, we need a way to sync locally created playlists between desktop and mobile — using Spotify/Apple Music as the intermediary.

## Design Decisions

- **Auto-sync with opt-out:** Local playlists automatically create on connected services during background sync. Users can mark individual playlists as "local only" to opt out.
- **All connected services:** Playlists sync to every connected sync provider (Spotify + Apple Music if both enabled). Each provider is independent.
- **Resolve at push time:** Track URIs (Spotify URIs, Apple Music IDs) are resolved via search when creating/updating the remote playlist, not pre-cached.
- **Auto-push ongoing changes:** After initial creation, edits (add/remove tracks, rename) auto-push on the next background sync cycle. No manual push required.

## Data Model Changes

### New playlist fields

```javascript
{
  // Existing fields...
  localOnly: false,          // Opt-out flag. When true, skip during auto-sync.
  syncedTo: {                // Tracks remote playlists created FROM this local playlist.
    spotify: {               // (Inverse of syncedFrom, which tracks playlists pulled from a service)
      externalId: '4abc...',
      snapshotId: 'xyz...',
      syncedAt: 1710000000000,
      unresolvedTracks: [],  // Tracks that couldn't be found on this service
      pendingAction: null    // null | 'remote-deleted'
    },
    applemusic: { ... }
  }
}
```

### Relationship to existing fields

- **`locallyModified`** — already exists. Set to `true` on any track add/remove/rename. Reset to `false` only after successfully pushing to ALL providers in `syncedTo`.
- **`syncedFrom`** — unchanged. Tracks playlists pulled from a service. A playlist can have both `syncedFrom` (pulled from Spotify) and `syncedTo` (pushed to Apple Music).
- **`localOnly`** — new. Setting to `true` does NOT delete already-created remote playlists, just stops future syncing.

## Provider Changes

### New method: `createPlaylist(name, description, token)`

Each sync provider needs a `createPlaylist` method:

**Spotify:**
- `POST /users/{user_id}/playlists` with `{ name, description, public: false }`
- Returns `{ externalId, snapshotId }`

**Apple Music:**
- `POST /me/library/playlists` with `{ attributes: { name, description } }`
- Returns `{ externalId, snapshotId }`
- Also requires new `updatePlaylistTracks()` and `updatePlaylistDetails()` methods (Apple Music provider is currently read-only)

### New IPC handler: `sync:create-playlist`

```javascript
ipcMain.handle('sync:create-playlist', async (event, providerId, name, description, tracks) => {
  // 1. Resolve track URIs via provider search API
  // 2. Create empty playlist on service
  // 3. Add resolved tracks via updatePlaylistTracks()
  // 4. Return { externalId, snapshotId, unresolvedTracks }
});
```

## Background Sync Integration

In `runBackgroundSync()`, after the existing collection sync completes for each provider:

1. Load playlists from store
2. **Initial creation:** Find playlists where `!localOnly && !syncedTo?.[providerId]` — create on the service
3. **Push updates:** Find playlists where `locallyModified && syncedTo?.[providerId] && !pendingAction` — push changes
4. Update `syncedTo` metadata, save playlists back to store
5. Reset `locallyModified = false` only after all providers are successfully synced

## Error Handling

### Track resolution failures
- Push the playlist with whatever tracks resolve successfully
- Store unresolved tracks in `syncedTo[provider].unresolvedTracks`
- Retry unresolved tracks on subsequent syncs (catalogs change over time)
- A partial playlist is better than no playlist

### Creation/push failures
- If API call fails (rate limit, auth expired, network), leave `syncedTo` unset for that provider
- Next background sync cycle retries automatically
- Each provider is independent — failure on one doesn't block others

### Remote playlist deleted
When a push fails with 404 or we detect a `syncedTo` playlist no longer exists remotely:

1. Set `syncedTo[provider].pendingAction = 'remote-deleted'`
2. Skip this playlist during background sync until user responds
3. Show notification/prompt: "'{playlist name}' was deleted on {service}. What would you like to do?"
4. Options:
   - **Delete locally too** — remove the local playlist entirely
   - **Keep local, stop syncing** — set `localOnly = true`, clear `syncedTo[provider]`
   - **Re-create on service** — clear `syncedTo[provider]`, next sync cycle re-creates it

### Auto-push `locallyModified` reset
Only reset `locallyModified = false` after successfully pushing to ALL providers in `syncedTo`. If one provider fails, keep the flag so it retries next cycle.

## UI Changes

### Playlist detail view
- **Sync status indicators** — small Spotify/Apple Music icons showing which services this playlist is synced to (next to playlist title or in metadata area)
- **"Local only" toggle** — in playlist settings/menu. Stops auto-sync without deleting remote playlists.
- **Pending action banner** — shown when `pendingAction === 'remote-deleted'`, with the three action buttons described above.

### Sync banner updates
The existing push/pull sync banner (for `syncedFrom` playlists) remains unchanged. The new `syncedTo` sync happens automatically in the background — no manual push/pull UI needed.

## Implementation Order

1. **Spotify provider:** Add `createPlaylist()` method
2. **Apple Music provider:** Add `createPlaylist()`, `updatePlaylistTracks()`, `updatePlaylistDetails()` methods
3. **IPC handler:** Add `sync:create-playlist` with track resolution
4. **Background sync:** Extend `runBackgroundSync()` with playlist creation and auto-push logic
5. **Data model:** Add `localOnly`, `syncedTo` fields to playlist objects
6. **UI:** Add sync status indicators, local-only toggle, and remote deletion prompt
7. **Remote deletion detection:** Add 404 handling in push flow and sync cycle
