# DEGuild-Craft

A private guild-focused derivative of [GuildCrafts](https://github.com/dkruenbo/GuildCrafts), a World of Warcraft addon that tracks guild members' profession recipes and synchronizes them between addon users.

Built for WoW Classic Forever (Interface 11507).

## Repository layout

- `GuildCrafts/` — the addon folder to load from `Interface/AddOns/`
- `spec/` — development specifications and design notes
- `RFC/` — technical proposals

## Local installation

Copy or symlink the `GuildCrafts/` folder into your WoW Classic Forever client's `Interface/AddOns/` directory, then `/reload`.

The addon is intended for WoW Classic Forever and includes its required libraries.

## Upstream

This project started from the MIT-licensed GuildCrafts repository by `dkruenbo`. The upstream remote is retained as `upstream` for optional future updates.
