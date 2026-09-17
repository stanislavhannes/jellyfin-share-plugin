# Jellyfin Share Plugin

A Jellyfin plugin that adds a "Share" button to movie, episode, season and series detail pages, allowing you to create temporary, shareable links for your media content.

This is a fork of [monxas/jellyfin-share-plugin](https://github.com/monxas/jellyfin-share-plugin)
with Jellyfin 12 support, permission checks, and a number of fixes — see
[What this fork changes](#what-this-fork-changes).

## Requirements

- Jellyfin 12.0 or later (built against the 12.1 ABI / .NET 10)
- For Jellyfin 10.11 use the 1.x line instead — a 2.x build will not load on it
- [Jellyfin Share Backend](https://github.com/stanislavhannes/jellyfin-share-backend) running and configured

## Installation

The catalogue serves the right build automatically: Jellyfin only offers versions
whose `targetAbi` it satisfies, so a 10.11 server sees the 1.x line and a 12.x
server sees 2.x.

### From Repository (Recommended)

1. Go to **Dashboard → Plugins → Repositories**
2. Add repository: `https://raw.githubusercontent.com/stanislavhannes/jellyfin-share-plugin/main/manifest.json`
3. Go to **Catalog** and install "Jellyfin Share"
4. Restart Jellyfin

### Manual Installation

1. Download the latest release from [Releases](https://github.com/stanislavhannes/jellyfin-share-plugin/releases)
2. Extract `Jellyfin.Plugin.Share.dll` to your Jellyfin plugins directory:
   - Linux: `/var/lib/jellyfin/plugins/JellyfinShare/`
   - Docker (official `jellyfin/jellyfin`): `/config/plugins/Jellyfin Share_<version>/`
   - Docker (linuxserver.io): `/config/data/plugins/Jellyfin Share_<version>/`
   - Windows: `C:\ProgramData\Jellyfin\Server\plugins\JellyfinShare\`

   The version in the folder name must be four parts (`1.2.3.0`). Jellyfin sorts
   plugin folders by it, so a locally built DLL needs a higher version than any
   copy installed from the catalogue or the catalogue's will win.
3. Restart Jellyfin

## Configuration

1. Go to **Dashboard → Plugins → Jellyfin Share**
2. Enter your backend URL (e.g., `http://localhost:8097` or `https://share.yourdomain.com`).
   This is how *this server* reaches the backend, which behind Docker or a reverse
   proxy is not the address a viewer opens - the backend reports that one itself.
3. Enter your backend API key
4. Configure the defaults the share dialog starts with: expiry in days, whether
   shares never expire by default, max plays and max concurrent viewers
5. Click **Save**
6. Click **Test Connection** to verify

## Enabling the Share Button

The plugin needs to inject JavaScript into the Jellyfin web interface. Since v1.2.0 it does
this itself: on every startup `IndexPatcher` appends `<script src="/plugins/share/loader.js">`
to `web/index.html` if it isn't there already. Because it re-runs on each boot, the button
survives Jellyfin upgrades — which wipe `web/` and undo the patch — **as long as the web
directory is writable by the user Jellyfin runs as.**

If the button disappears after an upgrade, that write permission is what to check first:

```
[WRN] Jellyfin Share: No write permission for "/usr/share/jellyfin/web/index.html"
```

`web/` ships as root-owned, while most container images run Jellyfin as a non-root user, so a
fresh image resets the ownership. On linuxserver.io images, restore it on every boot with a
`/custom-cont-init.d` hook (runs as root, before Jellyfin starts):

```yaml
volumes:
  - /path/to/jellyfin-init:/custom-cont-init.d:ro
```

```bash
# jellyfin-init/10-share-plugin-webperms.sh  (must be chmod +x)
#!/bin/bash
chown "${PUID:-911}:${PGID:-911}" /usr/share/jellyfin/web /usr/share/jellyfin/web/index.html
```

Confirm it worked — this line should appear on startup:

```
[INF] Jellyfin Share: Successfully patched index.html at "/usr/share/jellyfin/web/index.html"
```

Do **not** also inject `client.js` via the JavaScript Injector plugin or the Custom CSS
`</style><script>` trick — those are legacy workarounds, they broke on Jellyfin 10.11, and if
one starts working again the script loads twice.

## Usage

1. Navigate to a movie, episode, season or series detail page
2. Click the **Share** button in the action buttons row
3. Configure share options:
   - **Share type** (series and seasons only): share the item itself, or create one
     link per season / per episode in a single step
   - **Expires in (days)**, or **Never expires** to create a link without a deadline
   - **Quality**: Original, 1080p, 720p or 480p. A cap only ever lowers quality;
     it never raises it above the source
   - **Password**: Optional password protection
   - **Max plays**: Limit total number of plays (0 = unlimited)
   - **Max concurrent viewers**: Limit simultaneous viewers (0 = unlimited)
4. Click **Create Share Link**
5. Copy and share the generated URL

## API Endpoints

The plugin exposes these endpoints (authenticated):

- `GET /plugins/share/config` - Get plugin configuration and defaults
- `POST /plugins/share/create` - Create a share for one item
- `POST /plugins/share/batch` - Create one share per child of a season or series
- `GET /plugins/share/list` - List the calling user's own shares
- `POST /plugins/share/revoke/{shareId}` - Revoke one of the caller's shares
- `GET /plugins/share/analytics/{shareId}` - Statistics for one of the caller's shares

A user can only share items their own account is allowed to see, and can only
revoke or inspect shares they created themselves.

## Building from Source

```bash
# Clone the repository
git clone https://github.com/stanislavhannes/jellyfin-share-plugin.git
cd jellyfin-share-plugin

# Build
dotnet build -c Release

# The DLL will be in bin/Release/net10.0/
```

## Troubleshooting

### Share button doesn't appear

1. Verify the plugin is installed and enabled in Dashboard → Plugins
2. Check the startup log for the `IndexPatcher` line quoted under
   [Enabling the Share Button](#enabling-the-share-button) - a missing write
   permission on `web/` is the usual cause
3. Check that the client script loads (Browser DevTools → Network → "client.js").
   It is cached for a day and keyed to the plugin version, so reload with a hard
   refresh after an update
4. Do not add the script tag to branding settings - that is the legacy workaround
   this plugin replaced, and it loads the script twice

### "Plugin not configured" error

1. Go to Dashboard → Plugins → Jellyfin Share
2. Verify the backend URL and API key are set
3. Click "Test Connection" to verify connectivity

### Connection test fails

1. Verify the backend server is running
2. Check if the URL is accessible from Jellyfin server
3. Verify the API key matches what's configured in the backend

## What this fork changes

Relative to the upstream project. Every item was reproduced on a live server
before being fixed.

**Security**

- A user could share any item on the server, including content their own account
  is not allowed to open — items were resolved through `ILibraryManager`, which
  has no notion of who is asking. Now resolved against the caller with
  `IsVisibleStandalone`.
- A user could revoke or read the statistics of another user's share by its id.
  Both now refuse with 403.

**Fixes**

- The share button appeared only on the first detail page visited. Jellyfin keeps
  visited views hidden in the DOM, and the button row was looked up across the
  whole document, so season and episode pages found the stale one.
- The item type was guessed from English UI text, so on a non-English Jellyfin it
  silently fell back to "Movie" and hid the season and series batch options.
- Creating links for each season and then sharing the series itself threw
  `can't access property "value"` and produced no link.
- "My Shares" listed unreachable links: the URL was composed from the plugin's
  own backend address, which behind Docker or a proxy is not what a browser can
  open.
- The configured default expiry was never read by the dialog.

**Features**

- Share button on season and series pages, with batch creation per season or
  episode.
- Expiry in days, plus shares that never expire.
- Quality choice per share: Original, 1080p, 720p or 480p.
- Jellyfin 12 support (net10.0, `targetAbi 12.1.0.0`). The `1.x` branch stays on
  net9.0 for Jellyfin 10.11.

## License

MIT License - see [LICENSE](LICENSE) for details.

## Related

- [Jellyfin Share Backend](https://github.com/stanislavhannes/jellyfin-share-backend) - The backend server that handles share links
