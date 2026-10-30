# Mafia 2 Online — Pterodactyl egg

A Pterodactyl egg for the **Mafia 2 Online (M2O) V2** dedicated server — the multiplayer
modification for Mafia II, built on the [MafiaHub Framework](https://github.com/MafiaHub/Framework).

Import `egg-mafia-2-online.json` through **Admin → Nests → Import Egg**.

---

## Requirements

| | |
|---|---|
| glibc | **2.38 or newer** |
| Images | `ghcr.io/parkervcp/yolks:ubuntu` (24.04) or `ghcr.io/parkervcp/yolks:debian` (13) |
| Allocations | **two** — see Ports |
| Disk | ~1 GB for the server itself (`libnode.so.141` alone is 145 MB) |

On Ubuntu 22.04 or Debian 12 the binary does not start at all: the loader reports missing
`GLIBC_2.38` / `GLIBCXX_3.4.32` symbols. There is no workaround other than a newer base image.

## Ports

The server needs **two** ports, and they are not derived from one another.

| Port | Protocol | Default | What it is |
|---|---|---|---|
| Primary allocation | **UDP** | `27015` | All game traffic. The panel writes the server's allocated port here, so the default only applies to a server started with no config at all. |
| `API_PORT` | **TCP** | `27016` | HTTP endpoint serving server metadata as JSON, read by the in-game server browser and the MafiaHub masterlist. Set it to the server's second allocation. |

> **The guide's "port + 1" is wrong.** `apiport` is an independent setting that defaults to
> `27016` no matter what the game port is. Give the server a second allocation and set
> `API_PORT` to that port, otherwise the HTTP endpoint binds to a port the container never
> exposes and the server is invisible to the browser.

## Variables

| Variable | Default | Visible / Editable | What it does |
|---|---|---|---|
| `M2O_DOWNLOAD_URL` | [QGS mirror](https://source.qgshosting.com/downloads/m2o/latest.zip) | no / no | Direct link to the build archive (`M2O-<version>.zip`, or a tarball with the same contents). The installer downloads and unpacks it. Point it elsewhere to pin another version; left empty, the layout and `server.json` are still prepared and the binaries can be uploaded by hand. **Admin only** — it decides what code the server runs. |
| `API_PORT` | `27016` | yes / no | TCP port for the HTTP info endpoint. Must match the server's second allocation. Not editable by the client because it is tied to an allocation. |
| `MAX_PLAYERS` | `32` | yes / yes | Connection cap, also what the server reports to the browser. |
| `SEASON` | `summer` | yes / yes | World season — `summer` or `winter`. Validated at boot: a misspelled value stops the server with an error naming the key. |
| `MAP_NAME` | `Empire Bay` | yes / yes | Name shown in the server browser. Cosmetic. |
| `SERVER_PASSWORD` | *empty* | yes / yes | Password players must give to join. Empty lets anyone in. |
| `SERVER_TOKEN` | *empty* | yes / yes | MafiaHub masterlist push token. Without it the server runs fine as a direct-connect server, it is simply not announced and carries no public player count. |
| `LD_LIBRARY_PATH` | `.` | no / no | Lets the loader find `libnode.so.141` next to the executable. Do not change. |
| `TRACY_NO_INVARIANT_CHECK` | `1` | no / no | Without it the embedded profiler aborts the server on startup on CPUs with no invariant TSC — exit code 1 and no message. Do not change. |

## What the install does

1. Downloads `M2O_DOWNLOAD_URL` when set, into the server volume.
2. Extracts `server-linux/` from the archive **keeping its tree intact**, then moves its contents
   into the server root.
3. Makes `M2OServer` and `crashpad_handler` executable — a zip transfer does not preserve the bit.
4. Writes `server.json` if it is missing, already using the allocated ports.

> The official guide suggests `unzip -j '*server-linux/*'`. Do not: the archive also ships
> `resources/<name>/{client,server}/…`, and junking paths collapses those into the server root
> where the identical `main.js` and `package.json` files overwrite each other.

After a successful install the server root holds:

```
M2OServer            libnode.so.141     crashpad_handler
server.json          resources/         logs/
```

## server.json

Written by the installer and then kept in sync by the panel on every boot:

```json
{
    "host": "0.0.0.0",
    "port": 27015,
    "apihost": "0.0.0.0",
    "apiport": 27016,
    "map": "Empire Bay",
    "maxplayers": 32,
    "password": "",
    "server-token": "",
    "mod": { "season": "summer", "map_file": "" }
}
```

The panel manages `host`, `port`, `apihost`, `apiport`, `map`, `maxplayers`, `password`,
`server-token` and `mod.season`. Anything else you add by hand is left alone — the server keeps
unknown keys under `mod` with a warning rather than rejecting them.

Start the server once with no config at all and it writes a complete default `server.json`
documenting every key that build understands.

## Startup and shutdown

| | |
|---|---|
| Startup command | `./M2OServer` |
| Done marker | `Server successfully started` |
| Stop command | `quit` |

The done marker deliberately leaves out the mod name, because the server logs
`<ModName> Server successfully started`.

> **Known issue — shutdown hangs.** `quit` sets the stop flag and logs `Stopping server...`, but
> the process does not exit; the panel sits in *stopping* until Wings force-kills it on its stop
> timeout. Sending SIGINT (`^C`) behaves identically — in the framework both routes set the same
> `_stopRequested` flag, so swapping the stop command does not help. This is upstream.

## Player counts

M2O speaks MafiaNet, not the Source query protocol, so nothing outside the process can query it,
and the server's own HTTP endpoint reports `max_players` but never the live count.

The count does leave the server: every tick it pings the MafiaHub masterlist with its current
connection count, and that list is public:

```
GET https://masterlist.mafiahub.dev/servers
```

Each entry carries `gamemode`, `ip`, `port`, `current_players`, `max_players`, `name`, `version`,
`passworded` and `last_ping`. **A server only appears there when it runs with a valid
`SERVER_TOKEN`** — without one it is perfectly playable but has no public player count.

## Resources

Drop each resource into `resources/` next to the binaries; every one found there loads at startup,
ordered by its manifest `priority`. The manifest format the shipped examples actually use is:

```json
{
  "name": "my-gamemode",
  "version": "1.0.0",
  "mafiahub": {
    "serverScripts": ["server/main.js"],
    "clientScripts": ["client/main.js"],
    "files": ["client/**/*.js"],
    "priority": 20
  }
}
```

The server runtime is a full Node.js engine: `require()` works, including core modules such as
`fs` and `http`, alongside the framework globals (`Events`, `World`, `Player`, `Vehicle`, `Chat`,
`Voice`, `Entity`, `Blip`, `Marker`, `Prop`, `Npc`, `Shop`, `Door`, `Team`, `Weapon`,
`TextLabel`, …).

## Links

- [Scripting documentation](https://docs.mafia2online.com/)
- [MafiaHub Framework](https://github.com/MafiaHub/Framework)
- [mafia2online.com](https://mafia2online.com/)

The build archive is distributed through the project's Discord rather than a public URL. This egg
defaults to a QGS mirror of 1.6.1:

```
https://source.qgshosting.com/downloads/m2o/M2O-1.6.1.zip
```

Mirror a newer build yourself and change `M2O_DOWNLOAD_URL` to roll the egg forward.
