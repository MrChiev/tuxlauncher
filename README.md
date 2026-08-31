# AstroTuxLauncher Pterodactyl Egg (CRLF-fixed)

Pterodactyl egg + Docker image source for running an Astroneer dedicated server
via [AstroTuxLauncher](https://github.com/JoeJoeTV/AstroTuxLauncher).

## The problem

The publicly published image `ghcr.io/imkringle/tuxlauncher:latest` currently
fails to start with:

```
[FATAL tini (7)] exec /entrypoint.sh failed: No such file or directory
```

This happens because `entrypoint.sh` inside that published image has
Windows-style CRLF line endings (`\r\n`). The shebang line becomes
`#!/bin/bash\r`, so the kernel looks for an interpreter literally named
`/bin/bash\r`, which doesn't exist — even though `/bin/bash` itself is present
and fine.

Confirmed by checking the file directly inside the published image:
```bash
docker run --rm --entrypoint sh ghcr.io/imkringle/tuxlauncher:latest \
  -c "head -c 20 /entrypoint.sh | od -c"
# -> #   !   /   b   i   n   /   b   a   s   h  \r  \n   <- the \r is the bug
```

The source files in `docker/` here (as provided by the maintainer) have clean
Unix (`\n`) line endings, so building directly from them avoids the bug.

## Build

```bash
cd docker
docker build -t ghcr.io/mrchiev/tuxlauncher:latest .
docker push ghcr.io/mrchiev/tuxlauncher:latest
```

Verify the fix:
```bash
docker run --rm --entrypoint sh ghcr.io/mrchiev/tuxlauncher:latest \
  -c "head -c 20 /entrypoint.sh | od -c"
```
Should show only `\n`, no `\r`.

## Use in Pterodactyl

Import `egg-astroneer-astrotuxlauncher.json` into your panel (Admin → Nests →
Import Egg). It's already pointed at `ghcr.io/mrchiev/tuxlauncher:latest`
— update that in the egg JSON if you push under a different tag.

## Requirements

| Storage | RAM |
|---|---|
| 5.0 GiB | 2.0 GiB |

**Important**: your Wings node's `tmpfs_size` in `/etc/pterodactyl/config.yml`
needs to be at least `512` (default is `256`), or the server will crash with
an Out of Storage error. Restart Wings after changing it:
```bash
sudo systemctl restart wings
```

## Known limitation

Linux/Proton clients cannot currently join encrypted servers — the WINE-side
fix for Astroneer's encryption support (WINE 10.6+ / GnuTLS 3.8.3+) hasn't
merged into Proton yet. Windows clients are unaffected.

## Credits

Original design and logic (WINE setup, AstroTuxLauncher integration, config
handling) by Kringle (imkringle@proton.me), based on
[JoeJoeTV/AstroTuxLauncher](https://github.com/JoeJoeTV/AstroTuxLauncher).

CRLF fix, rebuild, and maintenance of this image by Chiev
(taingtiengchiev@gmail.com). If the CRLF issue gets fixed upstream in the
original repo, feel free to use that image instead.
