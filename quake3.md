---
layout: page
title: Quake 3 Server Guide
hidden: true
robots: noindex
---

Welcome to the dedicated server homepage for `quake3.jkrl.me`.

## Installing

- [Quake 3: Arena on GOG](https://www.gog.com/en/game/quake_iii_arena)
- [ioquake3 Downloads Page](https://ioquake3.org/get-it/) (windows/mac/linux)
- ioquake3 Setup Guides:
    - [Windows](https://ioquake3.org/help/players-guide/#windows)
    - [Mac](https://ioquake3.org/help/players-guide/#macos)

The server uses ioquake3, which is an open source Quake 3 game engine. The Quake 3 game files are still required.

You will have to download the Quake 3 game, install the ioquake3 engine, and then set up the engine to use the game files using one of the setup guides above.

## Using the Quake 3 Console

Once in game, you can open the console with `~`. Console commands are how you'll connect to the server, change maps, select game types, etc. All console commands start with `/`.

## Connecting

To connect to the server, set the password and then connect using the domain and port. The commands are:

```
/password <password>
/connect quake3.jkrl.me:27960
```

## RCON

RCON is Quake 3's remote console. You can authenticate to it with `/rconpassword <password>`. After you've authenticated you can run `/rcon <command>` using one of the commands below to change the server's configuration. If you run a command without the `/rcon` suffix you'll only be running it on your local machine and not for the whole server.

## Map Rotations

I have set up a couple preset map rotations for Team Deathmatch and CTF. The map rotations are stored in files on the server. You can run the different game modes with:

```
/rcon exec td.cfg       # for team deathmatch
/rcon exec ctf.cfg      # for ctf
```

## Commands

Here are some examples of useful in-game commands you can use with `/rcon`:

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

If you want to restore the server defaults, run `/rcon exec server.cfg`.

## Best Maps

Here's the best maps I'd recommend:

Team DM / FFA:

- q3dm5
- q3dm6
- q3dm7
- q3dm15
- q3dm16
- q3dm17

CTF:

- q3ctf1
- q3ctf4

