---
layout: page
title: Quake 3 Server Guide
hidden: true
robots: noindex
---

Welcome to the dedicated server homepage for `quake3.jkrl.me`.

## Installing

For this guide we're going to assume you already have a `baseq3/` folder with the original `pak0.pk3` game file and the ioquake3 `pak1.pk3` through `pak8.pk3` patch files. In this case all you need to do is:

1. Create an `ioquake3/` folder wherever your applications live
2. Download the [ioquake3 game engine](https://ioquake3.org/get-it/), then extract the application and put it in that folder
3. Put your `baseq3/` folder in the `ioquake3/` folder as well

Here's an example folder structure on Mac:

```
ioquake3
├── baseq3
│   ├── pak0.pk3
│   ├── pak1.pk3
│   ├── pak2.pk3
│   ├── pak3.pk3
│   ├── pak4.pk3
│   ├── pak5.pk3
│   ├── pak6.pk3
│   ├── pak7.pk3
│   └── pak8.pk3
└── ioquake3.app
```

You can now run ioquake3.

## Using the Quake 3 Console

Once in game, you can open the console with `~`. Console commands are how you'll connect to the server, change maps, select game types, etc. All console commands start with `/`.

## Connecting

To connect to the server, set the password and then connect using the domain and port. The commands are:

```
/password <password>
/connect quake3.jkrl.me:27960
```

If you haven't run the game before, also run the following command to enable automatic downloads of custom maps from the server:

```
/cl_allowdownload 1
```

## RCON

RCON is Quake 3's remote console. You can authenticate to it with `/rconpassword <password>`. After you've authenticated you can run `/rcon <command>` using one of the commands below to change the server's configuration.

Commands only require `/rcon` if they change the configuration of the server itself. For example, to simply change teams `/team` is fine, but to restart the game on the current map you're going to need `/rcon map_restart`.

## Map Rotations

I have set up a couple preset map rotations for Team Deathmatch, FFA, and CTF. The map rotations are stored in `.cfg` files on the server. You can run the different game modes and their map rotations with:

```
/rcon exec td.cfg       # for team deathmatch
/rcon exec ctf.cfg      # for ctf
/rcon exec ffa.cfg      # for ffa
```

Each have a differing number of maps in their rotation labelled `<gametype><number>` (i.e. `td1`, `ctf4`, `ffa2`). You can select Team Deathmatch map 5 from the rotation with `/rcon vstr td5`, and when that game ends the server will carry on to `td6`, etc.

If you ever manually select a map with `/rcon map <mapname>`, the map rotation will be broken and the server will keep replaying the same map. You can get back on the map rotation with one of the `exec` or `vstr` commands just mentioned.

## Commands

Here are some examples of useful in-game commands. Most will require `/rcon`:

```
/team red           # switch to red team
/g_gametype 3       # set the game type: 2->FFA 3->TD 4->CTF
/g_spskill 3        # bot skill from 1-5
/map q3dm17         # select the map
/bot_minplayers 4   # fill server with bots to satisfy the minimum
                    # set to 2 less than your desired number
/map_restart        # restart the current map
/fraglimit 25       # TD & FFA frag limit
/capturelimit 5     # CTF capture limit
/timelimit 10       # game time limit
/g_friendlyfire 0   # turn off friendly fire
```

## Restore Default Server Settings

If you want to restore the server defaults, run `/rcon exec server.cfg`. This restores only the base game settings. To set the map rotations you'll still have to exec the gametype files specified above.

