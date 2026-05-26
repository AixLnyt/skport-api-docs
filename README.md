# Skport / Gryphline API 文件

## Domains

| Domain | 用途 |
|--------|------|
| `as.gryphline.com` | 帳號認證、OAuth |
| `ef-webview.gryphline.com` | 抽卡紀錄、卡池 metadata |
| `web-api.skport.com` | ACCOUNT_TOKEN 刷新 |
| `zonai.skport.com` | 所有遊戲資料 API |
| `static.skport.com` | 靜態資源（圖片 CDN） |
| `game.skport.com` | 遊戲內工具頁面 |

---

## 認證流程

### 0. 帳號密碼登入（可選）
```
POST as.gryphline.com/user/auth/v1/token_by_email_password
Header: x-language: zh-tw
Body: { email, password, captchaToken }
→ ACCOUNT_TOKEN
```

### 0.5. 驗證 ACCOUNT_TOKEN 有效性
```
GET as.gryphline.com/user/info/v1/basic
Header: Authorization: Bearer {ACCOUNT_TOKEN}
→ 確認 token 有效（basicUser 資料）
```

### 1. OAuth Grant
```
POST as.gryphline.com/user/oauth2/v2/grant
Body: { token: ACCOUNT_TOKEN, appCode: "6eb76d4e13aa36e6", type: 0 }
→ code
```

### 2. 生成 cred
```
POST zonai.skport.com/web/v1/user/auth/generate_cred_by_code
Body: { kind: 1, code }
→ cred
```

### 3. 刷新取得 salt
```
GET zonai.skport.com/web/v1/auth/refresh
Header: cred
→ salt（用於所有需要簽名的 API）
```

---

## 簽名算法

### V2（主要端點用）
```
headerJson = { platform:"3", timestamp:"...", dId:"", vName:"1.0.0" }
sign = MD5( HEX( HMAC-SHA256( path + body + timestamp + headerJson, salt ) ) )
```

### V1（輕量端點用）
```
sign = MD5( path + query + timestamp + salt )
```

**自動選擇規則**：以下 path 使用 V2，其餘使用 V1：
- `/api/v1/game/player/binding`
- `/api/v1/game/endfield/card/detail`
- `/web/v1/wiki/*`
- `/web/v1/game/endfield/enums`
- `/web/v1/game/endfield/attendance*`
- `/web/v2/*`

| 請求類型 | body 值 |
|----------|---------|
| GET（無 params） | 空字串 `""` |
| GET（有 query params） | query string，例如 `roleId=xxx&serverId=yyy` |
| POST | JSON 字串 |

---

## 標準 Headers

```
Content-Type: application/json
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
sec-ch-ua: "Google Chrome";v="145", "Not-A.Brand";v="99"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
sec-fetch-dest: empty
sec-fetch-mode: cors
sec-fetch-site: same-site        # gryphline 域名改為 cross-site
Origin: https://game.skport.com
Referer: https://game.skport.com/
sk-language: zh-tw               # gryphline 域名改用 x-language: zh-tw
```

---

## 遊戲 API（zonai.skport.com）

### 取得綁定角色列表
```
GET zonai.skport.com/api/v1/game/player/binding
簽名: 需要（V2，body=""）
```

**回傳結構**
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
> gameRole 格式：`"3_{roleId}_{serverId}"`

---

### 取得完整遊戲資料卡
```
GET zonai.skport.com/api/v1/game/endfield/card/detail
Params: roleId, serverId（uid 可選）
Header: sk-game-role = "3_{roleId}_{serverId}"
簽名: 需要（V2，body = query string）
```

**回傳結構 `detail`**
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

**`base` 欄位**
| 欄位 | 說明 |
|------|------|
| `roleId` | UID |
| `name` | 玩家名稱 |
| `serverName` | 服務器名稱 |
| `level` | 權限等階 |
| `worldLevel` | 探索等級 |
| `exp` | 當前經驗值 |
| `gender` | 主角性別（1=男 2=女） |
| `avatarUrl` | 頭像圖片 URL |
| `mainMission.description` | 使命紀事文字 |
| `charNum` | 幹員數量 |
| `weaponNum` | 武器數量 |
| `docNum` | 文檔數量 |
| `createTime` | 甦醒日（timestamp） |
| `lastLoginTime` | 最後登入（timestamp） |
| `saveTime` | 資料儲存時間（timestamp） |

**`dungeon` 欄位**
| 欄位 | 說明 |
|------|------|
| `curStamina` | 當前理智 |
| `maxStamina` | 最大理智 |
| `maxTs` | 理智滿溢時間點（timestamp） |

**`bpSystem` 欄位**
| 欄位 | 說明 |
|------|------|
| `curLevel` | 通行證當前等級 |
| `maxLevel` | 通行證最大等級 |

**`dailyMission` 欄位**
| 欄位 | 說明 |
|------|------|
| `dailyActivation` | 每日任務當前進度 |
| `maxDailyActivation` | 每日任務上限 |

**`achieve` 欄位**
| 欄位 | 說明 |
|------|------|
| `count` | 光榮之路數量 |

**`spaceShip.rooms[]` 欄位**
| 欄位 | 說明 |
|------|------|
| `type` | 房間類型（`0` = 總控中樞） |
| `level` | 房間等級 |

**`chars[]` 欄位**
```
chars[]
├── charData
│   ├── id
│   ├── name
│   ├── avatarSqUrl        # 方形頭像
│   ├── avatarRtUrl        # 圓形頭像
│   ├── illustrationUrl    # 立繪
│   ├── rarity.key         # rarity_6 / rarity_5 / rarity_4 / rarity_3
│   ├── profession.value   # 突擊 / 術師 / ...
│   ├── property.value     # 元素屬性
│   ├── weaponType.value   # 武器類型
│   ├── skills[]
│   │   ├── id, name
│   │   ├── type.key       # skill_type_normal_attack / normal_skill / combo_skill / ultimate_skill
│   │   ├── iconUrl
│   │   └── desc
│   ├── labelType          # label_type_up（限定）等
│   └── tags[]
├── id
├── level
├── evolvePhase            # 精英化階段（0–4）
├── potentialLevel         # 潛能（0–5）
├── userSkills{}           # { skillId: { level, maxLevel } }
├── bodyEquip              # 護甲
├── armEquip               # 護手
├── firstAccessory         # 配件1
├── secondAccessory        # 配件2
├── tacticalItem           # 戰術道具
├── weapon
│   ├── weaponData{ id, name, iconUrl, rarity, type, skills[], labelType }
│   ├── level
│   ├── refineLevel        # 武器潛能（0–5）
│   ├── breakthroughLevel  # 武器突破（0–4）
│   └── gem{ id, icon }
├── gender
└── ownTs                  # 獲得時間（timestamp）
```

**`domain[]` 欄位**
```
domain[]
├── name                   # 地區名稱
├── level                  # 地區等級
├── settlements[]
│   ├── name               # 據點名稱
│   └── level              # 據點等級
└── collections[]
    ├── puzzleCount        # 謎質
    ├── trchestCount       # 儲藏箱
    ├── pieceCount         # 維修靈感點
    └── blackboxCount      # 黑匣子
```

**`quickaccess[]` 欄位**
| 名稱 | 說明 |
|------|------|
| 養成建議 | `game.skport.com/tools/endfield/list?routeId=1` |
| 編隊推薦 | `game.skport.com/tools/endfield/list?routeId=0` |
| 地圖工具 | `game.skport.com/map/endfield` |

---

### 取得遊戲枚舉常數
```
GET zonai.skport.com/web/v1/game/endfield/enums
簽名: 需要（V2，body=""）
```
> 回傳職業、屬性、稀有度、技能類型等所有遊戲枚舉，可用於替換 `rarity.key` / `profession.value` 等為顯示文字。

---

### 每日簽到

> 以下三個端點均需：
> - Header: `sk-game-role: 3_{roleId}_{serverId}`
> - Header: `Cookie: {已登入 cookie}`
> - 簽名: 需要（V2）

```
GET  zonai.skport.com/web/v1/game/endfield/attendance
→ 取得本月簽到日曆（已簽天數、獎勵清單）

POST zonai.skport.com/web/v1/game/endfield/attendance
→ 執行當日簽到（body = "{}"）

GET  zonai.skport.com/web/v1/game/endfield/attendance/records
→ 取得歷史簽到紀錄
```

---

### Wiki 資料庫

```
GET zonai.skport.com/web/v1/wiki/char-pool
→ 當前角色卡池 metadata（含 up6_name）

GET zonai.skport.com/web/v1/wiki/weapon-pool
→ 武器卡池 metadata

GET zonai.skport.com/web/v1/wiki/item/catalog
→ 道具資料庫（全清單）

GET zonai.skport.com/web/v1/wiki/item/info?id={item_id}
→ 特定道具詳情
```

---

### 取得用戶基本資料
```
GET zonai.skport.com/web/v2/user
簽名: 需要（V2，body=""）
```

**回傳結構**
```
{
  user
  ├── basicUser
  │   ├── id               # 平台 UID
  │   ├── nickname         # 暱稱
  │   ├── profile          # 自介（可能為空字串）
  │   ├── avatar           # 頭像 URL
  │   ├── avatarCode
  │   ├── gender           # 0=未設定
  │   ├── birthday         # timestamp
  │   ├── status
  │   ├── identity
  │   ├── createdAt        # 帳號建立時間
  │   └── latestLoginAt    # 最後登入時間
  ├── pendant
  │   ├── id
  │   ├── iconUrl
  │   ├── title            # 頭像框名稱
  │   └── description
  └── background           # 可能為 null
  userRts
  ├── follow               # 追蹤數
  ├── fans                 # 粉絲數
  └── liked                # 獲讚數
  userSanctionList[]
  userInfoApply{}
  moderator{}
}
```

---

## 抽卡紀錄 API（ef-webview.gryphline.com）

> 此組 API 使用遊戲客戶端匯出的 URL 內建 token，**不走 cred/salt 簽名流程**。

**URL 通用參數**

| 參數 | 說明 |
|------|------|
| `token` / `u8_token` | 從遊戲客戶端匯出的認證 token |
| `server_id` / `server` | 伺服器 ID（Global = `2`） |
| `lang` | 語言（預設 `en-us`） |

### 取得卡池 metadata
```
GET ef-webview.gryphline.com/api/content
Params: token, server_id, lang
→ 卡池資訊（含 up6_name、up6_item_name，可用於 featured 偵測）
```

### 角色抽卡紀錄
```
GET ef-webview.gryphline.com/api/record/char
Params: token, server_id, lang, seq_id（分頁游標，首次傳 0）
→ 角色抽卡紀錄列表
```

### 武器卡池列表
```
GET ef-webview.gryphline.com/api/record/weapon/pool
Params: token, server_id, lang
→ 所有武器卡池清單（取得各池的 pool_id）
```

### 武器抽卡紀錄
```
GET ef-webview.gryphline.com/api/record/weapon
Params: token, server_id, pool_id, lang, seq_id（分頁游標）
→ 指定武器卡池的抽卡紀錄
```

**Pity 分組規則**

| 類型 | Pity Group |
|------|------------|
| 限定 / 特別卡池 | `SpecialShared`（所有限定池共享） |
| 新手卡池 | `Beginner`（獨立計算） |
| 常駐池 | 依 `poolId` 各自獨立 |

---

## 遊戲術語對照

| 終末地術語 | 欄位 | 說明 | 上限 |
|-----------|------|------|------|
| 精英化 | `evolvePhase` | 幹員突破 | E4 |
| 潛能 | `potentialLevel` | 幹員命座 | 5 |
| 武器潛能 | `refineLevel` | 武器命座 | 5 |
| 武器突破 | `breakthroughLevel` | 武器突破階段 | 4 |
| 技能：普攻 | `skill_type_normal_attack` | — | — |
| 技能：技能 | `normal_skill` | — | — |
| 技能：聯合技 | `combo_skill` | — | — |
| 技能：終結技 | `ultimate_skill` | — | — |
| 限定幹員 | `labelType: label_type_up` | — | — |
