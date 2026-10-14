# pfUI-ThreatPlates

[**English**](README.md) | [Français](README.fr.md)

A "thirdparty" module for pfUI (Turtle WoW) that shows threat directly on nameplates, using the same server API as TW Threatmeter.

## Overview
On Turtle WoW the server computes every player's threat and sends it to addons that ask for it. pfUI-ThreatPlates puts that information **on the nameplates**: at a glance, on every mob, you see where you stand relative to the tank, without opening a threat window.

How it works, in four steps:
1. **Request**: in combat and in a group, the addon queries the server every X seconds (`TWT_UDTSv4_TM`, the same request TWThreat uses).
2. **Reply**: the server returns the threat list of your target and of the mobs you tank.
3. **Sharing**: the addon relays your target's list to the group (`TWT` channel, TWThreat format). Every player thus receives the threat of mobs *other people* are targeting.
4. **Display**: each nameplate is matched to its mob (by GUID, otherwise by name) and gets a thin bar, a text and, optionally, a recoloured health bar.

```
server ──TWTv4 / TMTv1──▶ you ──TWTRelayV2 (party/raid)──▶ other players
                           │                                  │
                           └──────▶ pfUI nameplates ◀─────────┘
```

## Installation
1. Install **pfUI** (required).
2. Copy the `pfUI-ThreatPlates` folder into `Interface/AddOns/`. If you downloaded the GitHub zip, rename the `pfUI-ThreatPlates-main` folder to `pfUI-ThreatPlates`.
3. Restart the game or type `/reload`.

Compatible with:
- **brues-code's pfUI + ClassicAPI** (recommended): nameplates tracked through `NAME_PLATE_UNIT_ADDED/REMOVED`, GUID through `UnitGUID`.
- **Original pfUI (shagu) + SuperWoW**: GUID through `GetName(1)` on the nameplate.
- Neither: only the target's nameplate is reliable.

TWThreat is optional. When it is present, both addons split the work without duplicating messages.

Settings: `/pfui` → **Thirdparty** → **Threat Plates** (enable, role, query interval, sharing, bar position, thresholds, colors).

## What you see
Each nameplate gets a thin threat bar (by default just **above** the health bar; position and spacing are adjustable) and a text:

| Situation | Display |
|---|---|
| You do not have aggro | your threat % relative to the tank (green → red) |
| You have aggro (role Auto/Tank) | next player + their % (red = they are about to pull) |
| You have aggro (role DPS/Healer) | `AGGRO` |
| Tank role, aggro lost | current tank's name + your %, "aggro lost" color |

### Health bar color ("Color Health Bar By Aggro", on by default)
| Role | Situation | Health bar |
|---|---|---|
| Tank (or Auto while you hold the mob) | you hold aggro, next player below the threshold | green |
| | you hold aggro, next player above the threshold | orange |
| Tank | aggro lost | red |
| DPS / Healer (or Auto without aggro) | threat below the threshold | normal pfUI color |
| | threat above the threshold | orange |
| | you have aggro | red |

The warning threshold is adjustable (80% by default). The thin bar and the text keep the %-based gradient.

### Short cache when you lose a target
When you switch or clear your target, the last known threat of the previous mob stays on its nameplate for **1 second** before disappearing, instead of vanishing at once (the data is no longer refreshed, so it is not kept longer). Adjust it with "Keep Threat After Target Loss (seconds, 0 = off)".

## Raw server data
Every X seconds in combat and in a group the addon sends the `TWT_UDTSv4_TM` request (like TWThreat in tank mode). The server answers with:
- the threat list of your **target** (`TWTv4=`);
- the list of **every mob you tank**, with the next player and their % (`TMTv1=`).

## Group sharing (TWThreat compatible)
Option "Share Threat With Group" (on by default). Sharing uses **TWThreat's channel and relay format** (prefix `TWT`, octo-addons version 1.3.0-octo1 and later):
- **Sending**: after each server reply, your target's threat list goes out as `TWTRelayV2:<mob>#player:tank:threat:%:melee;…`, the message TWThreat uses for its "Healer Master Target". If TWThreat already relays that reply, nothing is sent twice. If another player just shared the same list, the send is skipped.
- **Receiving**: relays from **any** group member, whether they come from TWThreat or pfUI-ThreatPlates. The mob is identified through the sender's target (its GUID, checked against the name).
- **Tank mode**: TWThreat has no equivalent; tanks running pfUI-ThreatPlates share their list as `PFTPTankV1:creature:guid:next:%;…` on the same channel (TWThreat ignores it).
- **Channel**: always `RAID` in a raid and `PARTY` in a party, chosen at send time; nothing is sent outside a group.
- **With TWThreat already sharing** (`/twt share`, octo-addons 1.3.0-octo2 and later): pfUI-ThreatPlates does not send duplicate relays, it only reads the group's.
- **Without TWThreat**: pfUI-ThreatPlates answers healers' `/twthmt` requests itself (`TWT_HMT` / `TWT_HMT_OK` / `TWT_HMT_REM`) and relays threat to them, even if sharing is disabled. When TWThreat is loaded, it handles that instead.

## Limits
- The server only returns details for **your target** plus the mobs **you tank**. Without sharing, a DPS therefore only sees threat on their own target; with sharing, on any mob targeted by a player who relays (pfUI-ThreatPlates, or TWThreat when a healer requested it).
- A mob nobody relays stays without information. Relays need ClassicAPI or SuperWoW to identify the mob. Shared data can be up to ~1 s old (request interval).
- Without ClassicAPI or SuperWoW only the target's nameplate is reliable; tanked mobs are matched by name (only if the name is unique).
- Nothing is sent when solo (the server only answers in a party/raid, like TWThreat).
- If TWThreat is running too, both addons query the server; you can raise the interval in the options.

## Troubleshooting
- **Nothing is displayed**: you must be in a party or raid and in combat; check that the module is enabled in the settings.
- **Threat only visible on my target**: expected without sharing. Enable "Share Threat With Group" and make sure the other players run pfUI-ThreatPlates or TWThreat.
- **Wrong mob colored**: install ClassicAPI or SuperWoW so mobs are matched by GUID.
- **Too many requests**: raise "Query Interval (seconds)", especially if TWThreat also runs.

## Updates
Releases are published in this repository; replace the addon folder with the latest version.
