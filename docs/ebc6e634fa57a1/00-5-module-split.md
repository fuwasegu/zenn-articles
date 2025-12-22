# 今回のアプリケーションにおけるモジュール分割

## モジュール一覧

| モジュール | 種類 | 含まれるドメイン | 備考 |
|-----------|------|-----------------|------|
| Member | 基本 | Member, Capacity | メンバー管理。稼働可能量も一緒に |
| Department | 基本 | Department | 組織構造。Member とは別の関心事 |
| Project | 基本 | Project, Requirement | プロジェクト管理。必要人数も一緒に |
| Assignment | 基本 | Assignment | アサイン管理。Member と Project をつなぐ |
| MemberSearch | 複合 | - | 複雑な条件でメンバーを検索 |
| WorkloadOverview | 複合 | - | 稼働率の集計・可視化 |

---

## ディレクトリ構造（モジュールレベル）

```
modules/
├── Contract/           # 公開 API
│   ├── Kernel/         # SharedKernel（全モジュール共通）
│   ├── Member/
│   ├── Department/
│   ├── Project/
│   └── Assignment/
│
├── Member/             # 基本モジュール
├── Department/         # 基本モジュール
├── Project/            # 基本モジュール
├── Assignment/         # 基本モジュール
│
├── MemberSearch/       # 複合モジュール（Contract なし）
└── WorkloadOverview/   # 複合モジュール（Contract なし）
```

---

## Mermaid 図（Contract と Module の関係）

```mermaid
graph TB
    subgraph Contract
        CK[Kernel]
        CM[Member]
        CD[Department]
        CP[Project]
        CA[Assignment]
    end

    subgraph 基本モジュール
        M[Member Module]
        D[Department Module]
        P[Project Module]
        A[Assignment Module]
    end

    subgraph 複合モジュール
        MS[MemberSearch]
        WO[WorkloadOverview]
    end

    %% 基本モジュールが Contract を実装
    M -.->|implements| CM
    D -.->|implements| CD
    P -.->|implements| CP
    A -.->|implements| CA

    %% 全モジュールが Kernel に依存
    M --> CK
    D --> CK
    P --> CK
    A --> CK
    MS --> CK
    WO --> CK

    %% Assignment が Member と Project の Contract に依存
    A --> CM
    A --> CP

    %% 複合モジュールが複数の Contract に依存
    MS --> CM
    MS --> CA
    WO --> CM
    WO --> CP
    WO --> CA
```

---

## 記事での書き方（案）

```markdown
## 今回のアプリケーションにおけるモジュール分割

想定アプリケーションを以下のようにモジュール分割しました．

| モジュール | 種類 | 含まれるドメイン |
|-----------|------|-----------------|
| Member | 基本 | Member, Capacity |
| Department | 基本 | Department |
| Project | 基本 | Project, Requirement |
| Assignment | 基本 | Assignment |
| MemberSearch | 複合 | - |

（mermaid 図）

- **基本モジュール**は自身の Contract を持ち，他モジュールから参照される
- **複合モジュール**は Contract を持たず，他モジュールの Contract に依存するだけ
- すべてのモジュールは **Kernel**（SharedKernel）に依存する
```

