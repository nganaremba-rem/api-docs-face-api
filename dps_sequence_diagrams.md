# 🚚 ドライバーポイントシステム · シーケンス図集

## Driver Point System (DPS) — Sequence Diagrams

> このドキュメントは DPS の内部仕様補足資料です。提供側・利用側・QAの3チームが共通の参照点として使用します。<br/>
> Supplementary documentation for DPS internals. Shared reference for producer, consumer, and QA teams.

すべての図は **Mermaid** 記法で書かれており、GitHub・GitLab・Notion・VSCode等で自動レンダリングされます。<br/>
All diagrams are written in **Mermaid** notation and render automatically in GitHub, GitLab, Notion, and VSCode.

---

## ⚠️ Errata & Open Questions · 未確認事項

このドキュメントを最終化する前に、提供側チームに確認が必要な項目です。図中の <code>❓</code> マーカーは未確認の箇所を示します。<br/>
*Items requiring producer-team confirmation before this doc is finalized. <code>❓</code> markers in diagrams indicate unconfirmed details.*

| # | 項目 / Item | 状態 / Status |
|:-:|---|---|
| Q1 | **Driver mobile login endpoint** — Doc shows `/auth/webLogin` but drivers are mobile-only. Face Auth confirmed separate. What's the driver mobile login endpoint? *(assumed `/auth/mobileLogin` in diagrams — please confirm)* | ❓ TBC |
| Q2 | **Post-withdrawal terminal state** — When manager approves, points are deducted from `driver_point`. Where do they go? Cash payout? Just removed? External payout system? | ❓ TBC |
| Q3 | **Monthly withdrawal limit** — Doc says "less than number of requests allowed per month" but doesn't give the number. Is it per-driver? Configurable per branch/role? | ❓ TBC |
| Q4 | **Permission model granularity** — Doc text says "give permission of function to role" (binary), but login response shows `{display, revise, print}` per function. Are both layers active? *(diagrams assume both)* | ❓ TBC |
| Q5 | **Branch balance update on credit** — When admin credits via `/points/creditPoints`, doc only says "Insert in cdt_pnt". Is `branch_info` also updated? *(diagrams assume yes — otherwise the give-to-driver step can't deduct from a non-existent balance)* | ❓ TBC |
| Q6 | **Endpoint naming inconsistencies in source doc:** `/user/deactivate` under Role Control API (typo for `/role/deactivate`?); `/branch/save` listed with both POST and PUT (PUT probably `/branch/update`?); `assineManager` (typo for `assignManager`?); `divCode` vs `divCd` vs `branchCode` used interchangeably. **Diagrams use the exact doc spelling — fix typos in source first, then update.** | ❓ TBC |
| Q7 | **Driver withdrawal deduction step** — Doc says points are deducted from driver on manager approval, but DB update list only mentions `Update req_pnt`. Where is the actual `driver_point` UPDATE? *(diagrams assume `driver_point` is also updated)* | ❓ TBC |

---

## 📋 Legend · 凡例

### Actors / アクター

| アイコン / Icon | アクター / Actor | 説明 / Description |
|:---:|---|---|
| 👤 | **Admin** 管理者 | スーパーユーザー · 全権限 / Super user with all permissions. Web only. |
| 🏢 | **Branch Manager** 支店長 | 支店を管理・ドライバーへポイント付与 / Manages branch, gives points to drivers. Web only. |
| 🚛 | **Driver** ドライバー | モバイルアプリのみ · ポイント受領と引出申請 / Mobile only. Receives points and requests withdrawals. |
| 👔 | **Custom Role** カスタムロール | 例: HR · 管理者が機能権限を付与 / e.g. HR. Admin grants function-level permissions. |
| 🌐 | **Web Frontend** Webフロント | Admin/Manager/Custom roles 用 |
| 📱 | **Mobile App** モバイルアプリ | Driver 専用 |
| ⚙️ | **Backend API** バックエンドAPI | DPS API server |
| 🗄️ | **Database** データベース | PostgreSQL / RDBMS |

### Entities / エンティティ

```
ROLE 担当       — ADMIN, BRANCH MANAGER, DRIVER (defaults, undeletable) + custom roles
BRANCH 営業所   — Organizational unit; holds point balance
USER ユーザー   — Belongs to a Branch; has a Role
```

### Phases / フェーズ色分け

```
🟦 Blue   — Client-side / クライアント側
🟨 Yellow — Authentication / 認証
🟧 Orange — Business logic / 業務ロジック
🟥 Red    — Failure paths / 失敗パス
🟩 Green  — Success / Response / 成功
```

---

## 1️⃣ システム概要 · System Overview

DPS の3つのコアエンティティと、各ロールがアクセスするインターフェースの関係です。<br/>
*The three core entities of DPS and the interfaces each role uses.*

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706',
    'primaryTextColor': '#1F2937',
    'lineColor': '#6B7280'
  }
}}%%
flowchart TB
    subgraph Web["🌐 Web Interface · Webインターフェース"]
        ADMIN[👤 Admin<br/>管理者]
        BM[🏢 Branch Manager<br/>支店長]
        HR[👔 Custom Role<br/>カスタムロール]
    end

    subgraph Mobile["📱 Mobile App · モバイルアプリ"]
        DRV[🚛 Driver<br/>ドライバー]
    end

    subgraph API["⚙️ DPS Backend API"]
        AUTH[Auth Module<br/>認証]
        UMGT[User Management<br/>ユーザー管理]
        RMGT[Role Management<br/>ロール管理]
        BMGT[Branch Management<br/>支店管理]
        PMGT[Point Management<br/>ポイント管理]
        MSG[Group Messaging<br/>グループメッセージ]
    end

    subgraph DB["🗄️ Database · データベース"]
        T1[(users<br/>roles<br/>user_roles)]
        T2[(branch_office<br/>branch_info)]
        T3[(cdt_pnt<br/>giv_pnt<br/>driver_point<br/>driver_point_record<br/>req_pnt)]
    end

    ADMIN --> UMGT
    ADMIN --> RMGT
    ADMIN --> BMGT
    ADMIN --> PMGT
    ADMIN --> MSG
    BM --> UMGT
    BM --> PMGT
    HR -.granted permissions.-> UMGT
    DRV --> AUTH
    DRV --> PMGT

    UMGT --> T1
    RMGT --> T1
    BMGT --> T2
    PMGT --> T3

    classDef webStyle fill:#DBEAFE,stroke:#1D4ED8,stroke-width:1.5px,color:#1E3A8A
    classDef mobStyle fill:#FED7AA,stroke:#C2410C,stroke-width:1.5px,color:#7C2D12
    classDef apiStyle fill:#FEF3C7,stroke:#D97706,stroke-width:1.5px,color:#78350F
    classDef dbStyle fill:#D1FAE5,stroke:#059669,stroke-width:1.5px,color:#064E3B

    class ADMIN,BM,HR webStyle
    class DRV mobStyle
    class AUTH,UMGT,RMGT,BMGT,PMGT,MSG apiStyle
    class T1,T2,T3 dbStyle
```

### Role responsibility matrix · ロール責任マトリクス

| 機能 / Function | Admin 管理者 | Branch Manager 支店長 | Driver ドライバー | Custom Role カスタム |
|---|:-:|:-:|:-:|:-:|
| Create/edit roles ロール管理 | ✅ | ❌ | ❌ | ⚙️ if permitted |
| Create branches 支店作成 | ✅ | ❌ | ❌ | ⚙️ if permitted |
| Create users ユーザー作成 | ✅ | ❌ | ❌ | ⚙️ if permitted |
| Credit points to branch ポイント生成 (Admin→Branch) | ✅ | ❌ | ❌ | ⚙️ if permitted |
| Give points to driver ポイント付与 (Branch→Driver) | ❌ | ✅ | ❌ | ❌ |
| Request withdrawal 引出申請 | ❌ | ❌ | ✅ | ❌ |
| Approve/deny withdrawal 引出承認 | ❌ | ✅ | ❌ | ❌ |
| Group messaging グループメッセージ | ✅ | ❌ | ❌ | ⚙️ if permitted |

---

## 2️⃣ 環境構築フロー · Bootstrap / Entity Creation Flow

新規システム導入時に Admin が実行する初期セットアップです。**Branch を作成 → Manager 作成 → Driver 作成** の順で行います。<br/>
*Initial setup performed by Admin when onboarding a new tenant: **Branch → Manager → Driver** in that order.*

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706',
    'actorBkg': '#FFF8EC',
    'actorBorder': '#D97706',
    'noteBkgColor': '#FEF3C7',
    'noteBorderColor': '#D97706',
    'noteTextColor': '#78350F',
    'sequenceNumberColor': '#FFFFFF',
    'labelBoxBkgColor': '#FEF3C7',
    'labelBoxBorderColor': '#D97706',
    'labelTextColor': '#78350F'
  }
}}%%
sequenceDiagram
    autonumber
    participant A as 👤 Admin<br/>管理者
    participant W as 🌐 Web Frontend
    participant API as ⚙️ Backend API
    participant DB as 🗄️ Database

    rect rgb(254, 243, 199)
        Note over A,DB: 🔐 Login · ログイン
        A->>W: ログイン画面 / Login screen
        W->>API: POST /auth/webLogin<br/>{ mailAddress, password }
        API->>DB: SELECT user, verify password<br/>+ load privileges
        DB-->>API: user + uiFunctionList + JWT
        API-->>W: 200 { basicUserInfo, uiFunctionList, jwt }
        Note over W: JWT保存 / Store JWT for subsequent requests
    end

    rect rgb(219, 234, 254)
        Note over A,DB: 🏢 Step 1 — Create Branch · 支店作成
        A->>W: 支店作成画面 / Branch creation form
        W->>API: POST /branch/save<br/>{ branchOfficeCode: "B1", branchOfficeName: "Branch One", lockCode: "10000" }
        API->>DB: INSERT branch_office<br/>INSERT branch_info (balance=0)
        DB-->>API: ✓
        API-->>W: 200 Success
    end

    rect rgb(254, 243, 199)
        Note over A,DB: 🧑‍💼 Step 2 — Create Branch Manager · 支店長作成
        A->>W: ユーザー作成 (role: BM)<br/>Create user form
        W->>API: POST /user/save<br/>{ userId: "BM1", roleCd: "BM", divCd: "B1", ... }
        Note over API: ⚠️ Validation:<br/>• userName/email unique<br/>• password policy check
        API->>DB: INSERT users<br/>INSERT user_roles
        DB-->>API: ✓
        API-->>W: 200 Success
    end

    rect rgb(254, 215, 170)
        Note over A,DB: 🚛 Step 3 — Create Driver · ドライバー作成
        A->>W: ユーザー作成 (role: DRIVER) / Create driver
        W->>API: POST /user/save<br/>{ userId: "DR123", roleCd: "DRIVER", divCd: "B1", ... }
        API->>DB: INSERT users<br/>INSERT user_roles<br/>INSERT driver_point (balance=0)
        DB-->>API: ✓
        API-->>W: 200 Success
    end

    rect rgb(220, 252, 231)
        Note over A,W: ✅ 環境構築完了 · Setup complete<br/>Branch + Manager + Driver ready for point flow
    end
```

### Alternative: assign existing manager · 既存マネージャーの割当

マネージャーは支店なしで作成し、後から割当することも可能です。<br/>
*Managers can be created without a branch and assigned later.*

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontSize':'14px','fontFamily':'"Helvetica Neue","Hiragino Sans","Yu Gothic","Noto Sans JP",sans-serif','primaryColor':'#FFF8EC','primaryBorderColor':'#D97706','actorBkg':'#FFF8EC','actorBorder':'#D97706','noteBkgColor':'#FEF3C7','noteBorderColor':'#D97706','sequenceNumberColor':'#FFFFFF','labelBoxBkgColor':'#FEF3C7'}}}%%
sequenceDiagram
    participant A as 👤 Admin
    participant API as ⚙️ Backend
    participant DB as 🗄️ DB

    A->>API: GET /user/managerWithNoBranch
    API->>DB: SELECT managers WHERE branch IS NULL
    DB-->>API: [BM7, BM12, ...]
    API-->>A: list of unassigned managers

    A->>API: GET /branch/getListForManagerChange
    API-->>A: list of branches

    A->>API: POST /branch/assineManager<br/>{ branchCd: "B7", managerId: "BM7" }
    Note right of A: ⚠️ Q6 — typo: "assine" → "assign"?
    API->>DB: UPDATE branch_info SET manager_id<br/>UPDATE users SET div_cd
    DB-->>API: ✓
    API-->>A: 200 Success
```

---

## 3️⃣ ポイントライフサイクル · Point Lifecycle (End-to-End)

**DPSの心臓部です。** ポイントが Admin から Branch、Branch から Driver、最終的に Driver の引出申請まで、4フェーズで流れます。<br/>
**The heart of DPS.** Points flow in 4 phases: Admin → Branch → Driver → withdrawal request → manager response.

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706',
    'actorBkg': '#FFF8EC',
    'actorBorder': '#D97706',
    'noteBkgColor': '#FEF3C7',
    'noteBorderColor': '#D97706',
    'noteTextColor': '#78350F',
    'sequenceNumberColor': '#FFFFFF',
    'labelBoxBkgColor': '#FEF3C7'
  }
}}%%
sequenceDiagram
    autonumber
    participant A as 👤 Admin
    participant BM as 🏢 Branch Manager
    participant DRV as 🚛 Driver
    participant API as ⚙️ Backend
    participant DB as 🗄️ Database

    rect rgb(254, 243, 199)
        Note over A,DB: 💰 Phase 1 — Admin → Branch (Initial Point Generation · 初期ポイント生成)
        A->>API: POST /points/creditPoints<br/>{ divCode: "B1", giverID: "admin", point: 10000 }
        Note over API: ⚠️ Points are GENERATED, not transferred.<br/>Admin doesn't need a balance.<br/>ポイントは生成される — 移転ではない
        API->>DB: INSERT cdt_pnt
        Note right of DB: ❓ Q5 — Is branch_info<br/>balance also updated here?
        API-->>A: 200 Success<br/>Branch balance: +10,000
    end

    rect rgb(219, 234, 254)
        Note over BM,DB: 🎁 Phase 2 — Branch → Driver (Point Transfer · ポイント付与)
        BM->>API: POST /points/givePoints<br/>{ givePointUser: "BM1", giveDivCode: "B1",<br/>  rcvUsrCode: "DR123", rcvDivCode: "B1", point: 1000 }

        API->>DB: SELECT branch_info<br/>WHERE divCd = 'B1'
        DB-->>API: balance = 10000

        alt 支店に十分なポイントあり · Branch has enough points
            API->>DB: INSERT giv_pnt<br/>UPDATE branch_info (−1000)<br/>UPDATE driver_point (+1000)<br/>INSERT driver_point_record
            DB-->>API: ✓
            API-->>BM: 200 Success
            Note over DB: Branch: 10,000 → 9,000<br/>Driver: 0 → 1,000
        else 残高不足 · Insufficient balance
            API-->>BM: 400 Error: 支店の残高不足<br/>"Branch has insufficient points"
        end
    end

    rect rgb(254, 215, 170)
        Note over DRV,DB: 📲 Phase 3 — Driver Withdrawal Request · 引出申請
        DRV->>API: POST /points/withdrawPointsRequest<br/>{ divCd: "B1", reqPoints: "8000" }

        API->>DB: SELECT COUNT(*) FROM req_pnt<br/>WHERE driver_id AND month = current
        DB-->>API: request_count
        Note right of API: ❓ Q3 — monthly limit value TBC

        alt 月間申請数が上限以内 · Within monthly limit
            API->>DB: INSERT req_pnt<br/>(status: pending)
            DB-->>API: reqPointsId = "RQ0000041"
            API-->>DRV: 200 Success
        else 月間上限超過 · Monthly limit exceeded
            API-->>DRV: 400 Error: 月間申請上限超過<br/>"Monthly request limit exceeded"
        end

        opt ドライバーが申請取消 · Driver cancels
            DRV->>API: POST /points/cancelWithdrawalRequest<br/>{ reqPointsId: "RQ0000041" }
            API->>DB: UPDATE req_pnt SET status = 'cancelled'
            API-->>DRV: 200 Success
        end
    end

    rect rgb(254, 226, 226)
        Note over BM,DB: ✅ Phase 4 — Manager Response · マネージャー応答
        BM->>API: POST /points/responseWithdrawRequest<br/>{ reqPointsId: "RQ0000041", modReqPoint: "8000" }

        Note over API: 3つの判定パターン (詳細は §4)<br/>3 decision branches (see §4)

        alt 全額承認 · Full grant (modReqPoint == reqPoints)
            API->>DB: UPDATE req_pnt SET status='approved', modReqPoint=8000<br/>UPDATE driver_point (−8000) ❓Q7
            API-->>BM: 200 Success — fully accepted
        else 部分承認 · Partial grant (0 < modReqPoint < reqPoints)
            API->>DB: UPDATE req_pnt SET status='partial', modReqPoint=N<br/>UPDATE driver_point (−N) ❓Q7
            API-->>BM: 200 Success — partially granted
        else 拒否 · Deny (modReqPoint == 0)
            API->>DB: UPDATE req_pnt SET status='denied', modReqPoint=0
            Note right of DB: No driver_point change
            API-->>BM: 200 Success — denied
        end
    end

    rect rgb(220, 252, 231)
        Note over A,DB: 🏁 End of lifecycle · ライフサイクル終了<br/>❓ Q2 — what happens to the withdrawn points next? Cash payout? External system?
    end
```

---

## 4️⃣ 引出判定ロジック · Withdrawal Decision Logic

マネージャーの応答 (`modReqPoint`) によって、3つの異なる結果が生まれます。<br/>
*Manager's response (`modReqPoint`) produces one of three outcomes.*

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '15px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706',
    'primaryTextColor': '#1F2937',
    'lineColor': '#6B7280'
  }
}}%%
flowchart TD
    START([🚛 Driver requests<br/>reqPoints = R<br/>引出申請]) --> WAIT{🏢 Manager response<br/>マネージャー応答<br/>modReqPoint = M}

    WAIT -->|M == R| FULL[✅ 全額承認<br/>Full Accept<br/>UPDATE req_pnt status=approved<br/>UPDATE driver_point −R]
    WAIT -->|0 < M < R| PARTIAL[🟡 部分承認<br/>Partial Grant<br/>UPDATE req_pnt status=partial<br/>UPDATE driver_point −M]
    WAIT -->|M == 0| DENY[❌ 拒否<br/>Deny<br/>UPDATE req_pnt status=denied<br/>driver_point unchanged]

    FULL --> END([End · 終了])
    PARTIAL --> END
    DENY --> END

    classDef startStyle fill:#FFF8EC,stroke:#D97706,stroke-width:2px,color:#1F2937
    classDef decision fill:#FEF3C7,stroke:#D97706,stroke-width:2px,color:#78350F
    classDef full fill:#D1FAE5,stroke:#059669,stroke-width:2.5px,color:#064E3B
    classDef partial fill:#FEF3C7,stroke:#D97706,stroke-width:2.5px,color:#78350F
    classDef deny fill:#FEE2E2,stroke:#DC2626,stroke-width:2.5px,color:#7F1D1D
    classDef endStyle fill:#F3F4F6,stroke:#6B7280,stroke-width:1.5px,color:#1F2937

    class START startStyle
    class WAIT decision
    class FULL full
    class PARTIAL partial
    class DENY deny
    class END endStyle
```

### Concrete examples · 具体例

| シナリオ / Scenario | reqPoints | modReqPoint | 結果 / Result | DB change to `driver_point` |
|---|:-:|:-:|---|---|
| ドライバー満額受領 / Driver gets all | 8,000 | 8,000 | ✅ Approved | −8,000 |
| マネージャーが半額に修正 / Manager halves it | 8,000 | 4,000 | 🟡 Partial | −4,000 |
| マネージャーが拒否 / Manager denies | 8,000 | 0 | ❌ Denied | 0 (no change) |
| ドライバーがキャンセル / Driver cancels first | 8,000 | (n/a) | 🚫 Cancelled | 0 (no change) |

> **QA注意点 / QA edge cases:**
> - `modReqPoint > reqPoints` → should this error? Doc doesn't specify. **❓**
> - Concurrent: driver cancels while manager is responding → race condition handling? **❓**
> - Driver `driver_point` balance < approved amount at time of approval → what then? **❓**

---

## 5️⃣ RBAC フロー · Role-Based Access Control Flow

カスタムロール (例: HR) を作成し、特定の機能権限を付与する流れです。<br/>
*Create a custom role (e.g. HR) and grant it specific function permissions.*

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706',
    'actorBkg': '#FFF8EC',
    'actorBorder': '#D97706',
    'noteBkgColor': '#FEF3C7',
    'noteBorderColor': '#D97706',
    'sequenceNumberColor': '#FFFFFF',
    'labelBoxBkgColor': '#FEF3C7'
  }
}}%%
sequenceDiagram
    autonumber
    participant A as 👤 Admin
    participant HR as 👔 HR User<br/>(new role)
    participant API as ⚙️ Backend
    participant DB as 🗄️ DB

    rect rgb(254, 243, 199)
        Note over A,DB: 🏗️ Step 1 — Create Custom Role · カスタムロール作成
        A->>API: POST /role/save<br/>{ privilegeCode: "HR", nameOfPrivilege: "Human Resources",<br/>  enteredByUser: "Admin" }
        API->>DB: INSERT role
        DB-->>API: ✓
        API-->>A: 200 Success
    end

    rect rgb(219, 234, 254)
        Note over A,DB: 🔑 Step 2 — Grant Function Permission · 機能権限付与
        A->>API: Permission page<br/>"Give Create New User to HR"
        API->>DB: INSERT role_function_permission<br/>(role: HR, function: EMPLOYEE_ADD, display=1, revise=1, print=0)
        Note right of API: ❓ Q4 — actual table name TBC<br/>{display,revise,print} per function
        DB-->>API: ✓
        API-->>A: 200 Success
    end

    rect rgb(254, 215, 170)
        Note over A,DB: 👤 Step 3 — Create User with Custom Role · ユーザー作成
        A->>API: POST /user/save<br/>{ userId: "HR1", roleCd: "HR", divCd: "B1", ... }
        API->>DB: INSERT users, INSERT user_roles (HR1 → HR)
        API-->>A: 200 Success
    end

    rect rgb(220, 252, 231)
        Note over HR,DB: ✨ Step 4 — HR User Exercises Granted Permission · 付与権限の行使
        HR->>API: POST /auth/webLogin<br/>{ mailAddress: "hr1@krt.com", password }
        API->>DB: SELECT user + uiFunctionList<br/>WHERE role=HR
        DB-->>API: user + functions [EMPLOYEE_ADD: {display:1, revise:1, print:0}]
        API-->>HR: 200 { JWT, uiFunctionList }

        Note over HR: UI shows only granted functions<br/>付与された機能のみ表示
        HR->>API: POST /user/save<br/>(create a new DRIVER · DRIVER 2)
        API->>API: Verify JWT + check HR has EMPLOYEE_ADD privilege
        API->>DB: INSERT users (DR2)
        API-->>HR: 200 Success
        Note over HR: DRIVER 2 can now participate in point flow<br/>(see §3)
    end
```

### Privilege model · 権限モデル

各機能ごとに、3軸の権限フラグを持ちます。<br/>
*Each function has a three-axis privilege flag.*

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontSize':'14px','fontFamily':'"Helvetica Neue","Hiragino Sans","Yu Gothic","Noto Sans JP",sans-serif','primaryColor':'#FFF8EC','primaryBorderColor':'#D97706'}}}%%
flowchart LR
    F[Function<br/>機能<br/>例: EMPLOYEE_ADD] --> D{display<br/>表示}
    F --> R{revise<br/>編集}
    F --> P{print<br/>印刷/出力}
    D -->|1| DV[UI上に表示<br/>Visible in UI menu]
    D -->|0| DH[UI上に非表示<br/>Hidden]
    R -->|1| RV[編集可能<br/>Can modify data]
    R -->|0| RH[読み取り専用<br/>Read-only]
    P -->|1| PV[印刷/出力可能<br/>Can print/export]
    P -->|0| PH[印刷不可<br/>Cannot print]

    classDef func fill:#FEF3C7,stroke:#D97706,stroke-width:2px
    classDef dim fill:#F3F4F6,stroke:#9CA3AF,color:#6B7280
    classDef enabled fill:#D1FAE5,stroke:#059669,color:#064E3B
    classDef disabled fill:#FEE2E2,stroke:#DC2626,color:#7F1D1D
    class F func
    class D,R,P func
    class DV,RV,PV enabled
    class DH,RH,PH disabled
```

> ❓ **Q4** — Doc text suggests simple "grant function to role" (binary). Login response shows `{display, revise, print}` (3-axis). Are both layers used, or did the doc text simplify the actual model? **Producer team to confirm.**

---

## 6️⃣ データベース関係図 · Database ER Diagram

DPSが参照する主要テーブルの関係です。<br/>
*Relationships between the main tables touched by DPS.*

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '13px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706'
  }
}}%%
erDiagram
    users ||--o{ user_roles : "has · 持つ"
    roles ||--o{ user_roles : "assigned to · 割当"
    branch_office ||--|| branch_info : "extends · 拡張"
    branch_info ||--o{ users : "houses · 所属"
    branch_info ||--o{ cdt_pnt : "credited by admin · 管理者により与えられる"
    branch_info ||--o{ giv_pnt : "source of · 元"
    users ||--o| driver_point : "if driver · ドライバーの場合"
    driver_point ||--o{ driver_point_record : "history · 履歴"
    users ||--o{ req_pnt : "requests · 申請"
    users ||--o{ giv_pnt : "receives · 受領"

    users {
        string userId PK
        string userName
        string passWd
        string roleCd FK
        string divCd FK "branch · 支店"
        string userSurnm
        string userGivnm
        string userSurnmk
        string userGivnmk
        string email
        string userSy "active/inactive"
        datetime createDt
    }
    roles {
        string privilegeCode PK
        string nameOfPrivilege
        string enteredByUser
        boolean isDefault "ADMIN/BM/DRIVER cannot delete"
    }
    user_roles {
        string user_id FK
        string role_id FK
    }
    branch_office {
        string divCd PK
        string divNm "branch name · 支店名"
        string lockCd "❓ purpose TBC"
        datetime createDt
        string createUs
    }
    branch_info {
        string divCd PK_FK
        int point_balance "❓ Q5 confirm"
        string manager_id FK
    }
    cdt_pnt {
        string id PK
        string divCode FK
        string giverID
        int point
        datetime createDt
    }
    giv_pnt {
        string id PK
        string givePointUser FK
        string giveDivCode FK
        string rcvUsrCode FK
        string rcvDivCode FK
        int point
        datetime createDt
    }
    driver_point {
        string user_id PK_FK
        int balance
    }
    driver_point_record {
        string id PK
        string user_id FK
        int delta "+ or −"
        string source "give/withdrawal"
        datetime createDt
    }
    req_pnt {
        string reqPointsId PK
        string user_id FK
        string divCd FK
        int reqPoints
        int modReqPoint "0=deny, =req=full, partial otherwise"
        string status "pending/approved/partial/denied/cancelled"
        datetime createDt
    }
```

> **Note:** Several column names above are inferred from request/response examples and may not match the actual schema. Producer team should validate against the real DDL.<br/>
> *上記の列名の一部はリクエスト/レスポンス例から推測されています。実際のDDLと照合してください。*

---

## 7️⃣ ログインと権限ロード · Login & Permission Loading

ログイン時に、ユーザーの権限が `uiFunctionList` として JWT と共に返却されます。フロントエンドはこれを使って UI を構築します。<br/>
*On login, user permissions are returned as `uiFunctionList` alongside the JWT. The frontend uses this to build the UI.*

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': '"Helvetica Neue", "Hiragino Sans", "Yu Gothic", "Noto Sans JP", sans-serif',
    'primaryColor': '#FFF8EC',
    'primaryBorderColor': '#D97706',
    'actorBkg': '#FFF8EC',
    'actorBorder': '#D97706',
    'noteBkgColor': '#FEF3C7',
    'noteBorderColor': '#D97706',
    'sequenceNumberColor': '#FFFFFF',
    'labelBoxBkgColor': '#FEF3C7'
  }
}}%%
sequenceDiagram
    autonumber
    participant U as 👤 User<br/>(any role)
    participant W as 🌐 Web Frontend
    participant API as ⚙️ Backend
    participant DB as 🗄️ DB

    U->>W: ログイン画面 / Login screen
    W->>API: POST /auth/webLogin<br/>{ mailAddress, password }

    API->>DB: SELECT users WHERE email = ?
    DB-->>API: user row (or null)

    alt user not found · ユーザー不在
        API-->>W: 401 Unauthorized
    else password mismatch · パスワード不一致
        API-->>W: 401 Unauthorized
    else valid · 有効
        API->>DB: SELECT functions + privileges<br/>JOIN role_function_permission<br/>WHERE role = user.roleCd
        DB-->>API: list of functions with<br/>{display, revise, print}

        API->>API: Generate JWT<br/>(sub: email, iat, exp)

        API-->>W: 200 {<br/>  basicUserInfo,<br/>  uiFunctionList: [...],<br/>  jwt<br/>}

        Note over W: UI builds menu from uiFunctionList<br/>menuType=1 → main menu<br/>menuType=2 → sub-menu<br/>privilege.display=0 → hidden
    end

    Note over U,DB: 後続のリクエストは Authorization: Bearer ${jwt}<br/>Subsequent requests use Bearer token
```

### Sample `uiFunctionList` entry · エントリ例

```json
{
  "funcId": "EMPLOYEE_ADD",
  "funcName": "Add Employee",
  "funcNameJp": "従業員 追加",
  "landingPage": "employee/create",
  "menuType": "2",
  "displayOrder": 2,
  "privilege": {
    "display": "1",
    "revise": "0",
    "print": "0"
  }
}
```

| Field | 説明 / Description |
|---|---|
| `funcId` | Stable function identifier · 機能ID |
| `funcName` / `funcNameJp` | Display label · 表示ラベル (English / Japanese) |
| `landingPage` | Route to navigate to · 遷移先URL |
| `menuType` | `1` = main menu / メインメニュー · `2` = sub-menu / サブメニュー |
| `displayOrder` | Sort order within menu · 表示順 |
| `privilege.display` | `1` show in UI · UI表示 · `0` hide · 非表示 |
| `privilege.revise` | `1` can modify · 編集可 · `0` read-only · 読み取り専用 |
| `privilege.print` | `1` can print/export · 印刷可 · `0` cannot · 不可 |

---

## 8️⃣ エラーレスポンス · Error Responses

DPS の標準エラーフォーマットです。<br/>
*Standard error formats in DPS.*

### Unauthorized (no/expired token) · 認証エラー

```json
{
  "timestamp": "2026-05-28T12:07:40.115+00:00",
  "status": 401,
  "error": "Unauthorized",
  "message": "Unauthorized",
  "path": "/branch/getAll"
}
```

### Session Expired · セッション切れ

```json
{
  "response": null,
  "status": {
    "code": "401",
    "message": "Session expired. Please log in again",
    "status": "FAILURE",
    "errorMessage": {
      "errNo": "EA001",
      "msgType": "",
      "icon": "",
      "errMsg": "Session expired. Please log in again",
      "errMsgJp": "セッションが切れました。再度ログインしてください"
    }
  },
  "infowarn": null
}
```

> **Consumer team note:** Two different 401 shapes exist (raw Spring-style vs wrapped `{response, status, infowarn}`). Frontend code must handle both. **❓ Why two shapes?** — possibly framework-level vs application-level error paths.

---

## 9️⃣ チーム別の使い方 · How Each Team Uses This Document

| チーム / Team | 主な参照箇所 / Primary sections | 目的 / Purpose |
|---|---|---|
| **🛠️ Producer (API team)** | §Errata + all `❓` markers · §6 ER · §8 Errors | Validate assumptions, fix typos in source doc, confirm schema |
| **🔌 Consumer (integrator)** | §1 Overview · §3 Point Lifecycle · §7 Login · §8 Errors | Understand endpoint sequencing, response shapes |
| **🧪 QA (testers)** | §3 Point Lifecycle (failure paths) · §4 Withdrawal Decision · §8 Errors | Design test scenarios incl. partial grants, denials, limits |

### Rendering & embedding · 描画と埋込

| ツール / Tool | 対応 / Support |
|---|---|
| GitHub (README, .md, wiki) | ✅ ネイティブ対応 / Native |
| GitLab | ✅ ネイティブ対応 / Native |
| Notion | ✅ `/code` block, language: mermaid |
| VSCode | ✅ Markdown Preview Mermaid Support 拡張 / extension |
| Confluence | ✅ Mermaid macro |
| プレビュー / Live preview | [mermaid.live](https://mermaid.live) |

---

## 🔄 バージョン履歴 · Version History

| 日付 / Date | バージョン / Version | 変更内容 / Changes |
|---|---|---|
| 2026-05-28 | v0.1 (DRAFT) | 初版 · Initial draft. **Open questions Q1–Q7 require producer-team validation before promoting to v1.0.** |

---

<p align="center"><sub>このドキュメントは DPS API 仕様書の補足資料です / This document is a supplement to the DPS API specification</sub></p>
<p align="center"><sub>レンダリングが崩れた場合は <a href="https://mermaid.live">mermaid.live</a> で確認してください / Use mermaid.live to verify if rendering breaks</sub></p>
