# Skport / Gryphline API Documentation

[中文版](./README.md)

> Unofficial API reference for third-party Arknights: Endfield integrations.
>
> **Translation note:** Common game terms (Promotion = 精英化, Potential = 潛能/命座, Caster = 術師, Sanity = 理智, etc.) have been verified against the official Arknights: Endfield English client. Entries marked **"unverified translation"** (Enigma Matter, Black Box, Repair Inspiration Point, Main Control Hub, Glory Road) are best-effort guesses, not confirmed official terms — verify in-game before relying on them.

---

## Domains

| Domain | Purpose |
|--------|---------|
| `as.gryphline.com` | Account authentication, OAuth |
| `ef-webview.gryphline.com` | Gacha history, banner metadata |
| `web-api.skport.com` | ACCOUNT_TOKEN refresh |
| `zonai.skport.com` | All in-game data APIs |
| `static.skport.com` | Static assets (image CDN) |
| `game.skport.com` | In-game tools pages |

---

## Authentication Flow

### 0. Email/password login (optional)
```
POST as.gryphline.com/user/auth/v1/token_by_email_password
Header: x-language: zh-tw
Body: { email, password, captchaToken }
→ ACCOUNT_TOKEN
```

### 0.5. Verify ACCOUNT_TOKEN validity
```
GET as.gryphline.com/user/info/v1/basic
Header: Authorization: Bearer {ACCOUNT_TOKEN}
→ confirms token is valid (basicUser data)
```

### 1. OAuth Grant
```
POST as.gryphline.com/user/oauth2/v2/grant
Body: { token: ACCOUNT_TOKEN, appCode: "6eb76d4e13aa36e6", type: 0 }
→ code
```

### 2. Generate cred
```
POST zonai.skport.com/web/v1/user/auth/generate_cred_by_code
Body: { kind: 1, code }
→ cred
```

### 3. Refresh to obtain salt
```
GET zonai.skport.com/web/v1/auth/refresh
Header: cred
→ salt (used for all signature-required APIs)
```

---

## Request Signing

### V2 (primary endpoints)
```
headerJson = { platform:"3", timestamp:"...", dId:"", vName:"1.0.0" }
sign = MD5( HEX( HMAC-SHA256( path + body + timestamp + headerJson, salt ) ) )
```

### V1 (lightweight endpoints)
```
sign = MD5( path + query + timestamp + salt )
```

**Auto-selection rule**: the following paths use V2, all others use V1:
- `/api/v1/game/player/binding`
- `/api/v1/game/endfield/card/detail`
- `/web/v1/wiki/*`
- `/web/v1/game/endfield/enums`
- `/web/v1/game/endfield/attendance*`
- `/web/v2/*`

| Request Type | `body` value |
|---------------|---------------|
| GET (no params) | empty string `""` |
| GET (with query params) | query string, e.g. `roleId=xxx&serverId=yyy` |
| POST | JSON string |

---

## Standard Headers

```
Content-Type: application/json
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
sec-ch-ua: "Google Chrome";v="145", "Not-A.Brand";v="99"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
sec-fetch-dest: empty
sec-fetch-mode: cors
sec-fetch-site: same-site        # becomes cross-site for gryphline domains
Origin: https://game.skport.com
Referer: https://game.skport.com/
sk-language: zh-tw               # gryphline domains use x-language: zh-tw instead
```

---

## Game API (zonai.skport.com)

### Get bound role list
```
GET zonai.skport.com/api/v1/game/player/binding
Signature: required (V2, body="")
```

**Response shape**
```json
{
  "data": {
    "list": [
      {
        "appCode": "endfield",
        "bindingList": [
          {
            "defaultRole": {
              "roleId": "...",
              "serverId": "...",
              "nickname": "...",
              "level": 0,
              "serverName": "..."
            },
            "roles": []
          }
        ]
      }
    ]
  }
}
```
> `gameRole` format: `"3_{roleId}_{serverId}"`

---

### Get full game data card
```
GET zonai.skport.com/api/v1/game/endfield/card/detail
Params: roleId, serverId (uid optional)
Header: sk-game-role = "3_{roleId}_{serverId}"
Signature: required (V2, body = query string)
```

**Response shape: `detail`**
```
detail
├── base
├── chars[]
├── achieve
├── spaceShip
│   └── rooms[]
├── domain[]
├── dungeon
├── bpSystem
├── dailyMission
├── config
└── quickaccess[]
```

**`base` fields**
| Field | Description |
|-------|--------------|
| `roleId` | UID |
| `name` | Player name |
| `serverName` | Server name |
| `level` | Permission level |
| `worldLevel` | Exploration level |
| `exp` | Current EXP |
| `gender` | Protagonist gender (1=male, 2=female) |
| `avatarUrl` | Avatar image URL |
| `mainMission.description` | Main mission log text |
| `charNum` | Operator count |
| `weaponNum` | Weapon count |
| `docNum` | Document count |
| `createTime` | Awakening date (timestamp) |
| `lastLoginTime` | Last login (timestamp) |
| `saveTime` | Data save time (timestamp) |

**`dungeon` fields**
| Field | Description |
|-------|--------------|
| `curStamina` | Current Sanity |
| `maxStamina` | Max Sanity |
| `maxTs` | Sanity-full timestamp |

**`bpSystem` fields**
| Field | Description |
|-------|--------------|
| `curLevel` | Battle pass current level |
| `maxLevel` | Battle pass max level |

**`dailyMission` fields**
| Field | Description |
|-------|--------------|
| `dailyActivation` | Current daily mission progress |
| `maxDailyActivation` | Daily mission cap |

**`achieve` fields**
| Field | Description |
|-------|--------------|
| `count` | "Glory Road" count *(unverified translation)* |

**`spaceShip.rooms[]` fields**
| Field | Description |
|-------|--------------|
| `type` | Room type (`0` = "Main Control Hub", *unverified translation*) |
| `level` | Room level |

**`chars[]` fields**
```
chars[]
├── charData
│   ├── id
│   ├── name
│   ├── avatarSqUrl        # square avatar
│   ├── avatarRtUrl        # round avatar
│   ├── illustrationUrl    # character illustration
│   ├── rarity.key         # rarity_6 / rarity_5 / rarity_4 / rarity_3
│   ├── profession.value   # Official 6 classes: Vanguard / Guard / Defender / Caster / Sniper / Medic
│   │                      # (verify against actual API string — Chinese class names are
│   │                      #  先鋒/近衛/重裝/術師/狙擊/醫療)
│   ├── property.value     # elemental property
│   ├── weaponType.value   # weapon type
│   ├── skills[]
│   │   ├── id, name
│   │   ├── type.key       # skill_type_normal_attack / normal_skill / combo_skill / ultimate_skill
│   │   ├── iconUrl
│   │   └── desc
│   ├── labelType          # label_type_up (limited), etc.
│   └── tags[]
├── id
├── level
├── evolvePhase            # Promotion (0–4) — official term, formerly mistranslated as "Elite Phase"
├── potentialLevel         # Potential (0–5)
├── userSkills{}           # { skillId: { level, maxLevel } }
├── bodyEquip              # armor
├── armEquip               # gauntlet
├── firstAccessory         # accessory 1
├── secondAccessory        # accessory 2
├── tacticalItem           # tactical item
├── weapon
│   ├── weaponData{ id, name, iconUrl, rarity, type, skills[], labelType }
│   ├── level
│   ├── refineLevel        # Weapon Potential (0–5)
│   ├── breakthroughLevel  # Weapon breakthrough (0–4)
│   └── gem{ id, icon }
├── gender
└── ownTs                  # acquisition time (timestamp)
```

**`domain[]` fields**
```
domain[]
├── name                   # region name
├── level                  # region level
├── settlements[]
│   ├── name               # settlement name
│   └── level               # settlement level
└── collections[]
    ├── puzzleCount        # "Enigma Matter" (unverified translation)
    ├── trchestCount       # "Storage Chest" (unverified translation)
    ├── pieceCount         # "Repair Inspiration Point" (unverified translation)
    └── blackboxCount      # "Black Box" (unverified translation)
```

**`quickaccess[]` fields**
| Name | Description |
|------|--------------|
| Growth Guide | `game.skport.com/tools/endfield/list?routeId=1` |
| Team Recommendation | `game.skport.com/tools/endfield/list?routeId=0` |
| Map Tool | `game.skport.com/map/endfield` |

---

### Get game enum constants
```
GET zonai.skport.com/web/v1/game/endfield/enums
Signature: required (V2, body="")
```
> Returns all game enums — profession, property, rarity, skill type, etc. — used to map fields like `rarity.key` / `profession.value` to display text.

---

### Daily attendance

> All three endpoints below require:
> - Header: `sk-game-role: 3_{roleId}_{serverId}`
> - Header: `Cookie: {logged-in cookie}`
> - Signature: required (V2)

```
GET  zonai.skport.com/web/v1/game/endfield/attendance
→ Get this month's check-in calendar (days checked, reward list)

POST zonai.skport.com/web/v1/game/endfield/attendance
→ Perform today's check-in (body = "{}")

GET  zonai.skport.com/web/v1/game/endfield/attendance/records
→ Get historical check-in records
```

---

### Wiki database

```
GET zonai.skport.com/web/v1/wiki/char-pool
→ current character banner metadata (incl. up6_name)

GET zonai.skport.com/web/v1/wiki/weapon-pool
→ weapon banner metadata

GET zonai.skport.com/web/v1/wiki/item/catalog
→ item database (full list)

GET zonai.skport.com/web/v1/wiki/item/info?id={item_id}
→ specific item details
```

---

### Get user profile
```
GET zonai.skport.com/web/v2/user
Signature: required (V2, body="")
```

**Response shape**
```
{
  user
  ├── basicUser
  │   ├── id               # platform UID
  │   ├── nickname         # nickname
  │   ├── profile          # bio (may be empty string)
  │   ├── avatar           # avatar URL
  │   ├── avatarCode
  │   ├── gender           # 0=unset
  │   ├── birthday         # timestamp
  │   ├── status
  │   ├── identity
  │   ├── createdAt        # account creation time
  │   └── latestLoginAt    # last login time
  ├── pendant
  │   ├── id
  │   ├── iconUrl
  │   ├── title            # avatar frame name
  │   └── description
  └── background           # may be null
  userRts
  ├── follow               # following count
  ├── fans                 # fan/follower count
  └── liked                # likes received
  userSanctionList[]
  userInfoApply{}
  moderator{}
}
```

---

## Gacha Record API (ef-webview.gryphline.com)

> This API group uses a token embedded in a URL exported from the game client, and does **not** go through the cred/salt signing flow.

**Common URL parameters**

| Parameter | Description |
|-----------|--------------|
| `token` / `u8_token` | Auth token exported from the game client |
| `server_id` / `server` | Server ID (Global = `2`) |
| `lang` | Language (default `en-us`) |

### Get banner metadata
```
GET ef-webview.gryphline.com/api/content
Params: token, server_id, lang
→ banner info (incl. up6_name, up6_item_name, usable for featured-item detection)
```

### Character pull history
```
GET ef-webview.gryphline.com/api/record/char
Params: token, server_id, lang, seq_id (pagination cursor, pass 0 on first call)
→ list of character pull records
```

### Weapon banner list
```
GET ef-webview.gryphline.com/api/record/weapon/pool
Params: token, server_id, lang
→ list of all weapon banners (gives each banner's pool_id)
```

### Weapon pull history
```
GET ef-webview.gryphline.com/api/record/weapon
Params: token, server_id, pool_id, lang, seq_id (pagination cursor)
→ pull history for the specified weapon banner
```

**Pity grouping rules**

| Type | Pity Group |
|------|------------|
| Limited / Special banner | `SpecialShared` (shared across all limited banners) |
| Beginner banner | `Beginner` (calculated independently) |
| Standard banner | independent per `poolId` |

---

## Terminology

| Endfield Term (EN) | Field | Description | Cap |
|---------------------|-------|--------------|-----|
| Promotion | `evolvePhase` | Operator promotion / elite tier | E4 |
| Potential | `potentialLevel` | Operator potential | 5 |
| Weapon Potential | `refineLevel` | Weapon potential | 5 |
| Weapon Breakthrough | `breakthroughLevel` | Weapon breakthrough stage | 4 |
| Skill: Basic Attack | `skill_type_normal_attack` | — | — |
| Skill: Battle Skill | `normal_skill` | — | — |
| Skill: Combo Skill | `combo_skill` | — | — |
| Skill: Ultimate | `ultimate_skill` | — | — |
| Limited Operator | `labelType: label_type_up` | — | — |

**Operator classes** (official, verified against EN client):

| Chinese | English |
|---------|---------|
| 先鋒 | Vanguard |
| 近衛 | Guard |
| 重裝 | Defender |
| 術師 | Caster |
| 狙擊 | Sniper |
| 醫療 | Medic |
