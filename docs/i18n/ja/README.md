# AI Dev OS Rules — Python

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)

> Python プロジェクト（FastAPI、Flask、Typer CLI 等）向けの 4 層ガイドライン。
> git submodule として追加し、AI コーディングアシスタントから参照します。

**[AI Dev OS](https://github.com/yunbow/ai-dev-os) エコシステムの一部です。**

## なぜこのルールか？

AI Dev OS Rules は、AI コーディングアシスタントに曖昧な指示ではなく**具体的で検証可能な基準**を与えます:

- **13 の共通ルール** — 命名、エラーハンドリング、セキュリティ、テスト、ロギング、i18n など
- **フレームワーク固有ルール** — FastAPI パターン、Python CLI ベストプラクティス
- **競合解決が組み込み** — Specificity Cascade がルール優先度を自動的に解決
- **バージョン管理・監査可能** — タグに固定、差分確認、PR でレビュー

## クイックスタート

```bash
npx ai-dev-os init --rules python --plugin claude-code
```

> すべて自動でセットアップされます。詳細は [AI Dev OS CLI](https://github.com/yunbow/ai-dev-os-cli) を参照。

<details>
<summary>手動セットアップ</summary>

**submodule として追加**

```bash
cd /path/to/your-project
git submodule add https://github.com/yunbow/ai-dev-os-rules-python.git docs/ai-dev-os
git submodule update --init
```

**テンプレートでセットアップ（Python CLI の場合）**

```bash
bash docs/ai-dev-os/templates/python-cli/submodule-setup.sh
```

**CLAUDE.md を編集**

`templates/python-cli/CLAUDE.md.template` を `./CLAUDE.md` にコピーし、プロジェクト名と固有ガイドラインを記入します。

**submodule の更新**

```bash
git submodule update --remote docs/ai-dev-os
```

</details>

## 含まれる内容

| レイヤー | パス | 内容 |
|---------|------|------|
| L1 — 設計思想 | `01_philosophy/` | 原則、メンタルモデル、アンチパターン |
| L2 — 判断基準 | `02_decision-criteria/` | 抽象化、技術選定、アーキテクチャ、エラー、セキュリティ |
| L3 — 共通ガイドライン | `03_guidelines/common/` | 13 ルール: コード、命名、バリデーション、エラー、ロギング、セキュリティ、テスト等 |
| L3 — FW ガイドライン | `03_guidelines/frameworks/` | [FastAPI](03_guidelines/frameworks/fastapi/README.md)、[Python CLI](03_guidelines/frameworks/python-cli/README.md) |
| テンプレート | `templates/` | [Python CLI スキャフォールディング](templates/python-cli/README.md) |

## Specificity Cascade

ルールが競合した場合、**番号が小さいほど優先**されます。

| 優先度 | レイヤー | 例 |
|--------|---------|------|
| 1（最優先） | フレームワーク固有ガイドライン | `03_guidelines/frameworks/python-cli/*` |
| 2 | 共通ガイドライン | `03_guidelines/common/*` |
| 3 | 判断基準 | `02_decision-criteria/*` |
| 4 | 設計思想 | `01_philosophy/*` |

<details>
<summary>ディレクトリ構造</summary>

```
ai-dev-os/
├── docs/
│   ├── operation-guide.md        # 運用・コントリビューションガイド
│   └── i18n/                     # 多言語ガイド
│       ├── ja/                   #   日本語
│       ├── zh-CN/                #   简体中文
│       ├── ko/                   #   한국어
│       └── es/                   #   Español
│
├── 01_philosophy/                # 設計思想 [サンプル — 母国語に書き換え]
│   ├── principles.md             #   三本柱: Correctness, Observability, Pragmatism
│   ├── mental-models.md          #   10の思考フレームワーク
│   └── anti-patterns.md          #   避けるべきパターン（コード例付き）
│
├── 02_decision-criteria/         # 判断基準 [サンプル — 母国語に書き換え]
│   ├── abstraction.md            #   共通化・抽象化のタイミングと閾値
│   ├── technology-selection.md   #   技術選定の判断フレームワーク
│   ├── architecture.md           #   API設計、状態管理、モジュール構成
│   ├── error-strategy.md         #   エラー分類、リトライ、Result パターン
│   └── security-vs-ux.md        #   セキュリティ施策の優先度とバランス
│
├── 03_guidelines/                # ガイドライン [英語]
│   ├── common/                   #   共通（言語・FW非依存）
│   │   ├── code.md               #     コーディング規約
│   │   ├── naming.md             #     命名規則
│   │   ├── validation.md         #     バリデーション
│   │   ├── error-handling.md     #     エラーハンドリング
│   │   ├── logging.md            #     ロギング
│   │   ├── security.md           #     セキュリティ
│   │   ├── rate-limiting.md      #     レート制限
│   │   ├── testing.md            #     テスト
│   │   ├── performance.md        #     パフォーマンス
│   │   ├── cors.md               #     CORS
│   │   ├── env.md                #     環境変数
│   │   ├── cicd.md               #     CI/CD
│   │   └── i18n.md               #     多言語化
│   │
│   └── frameworks/               #   フレームワーク固有（各 README.md を参照）
│       ├── fastapi/              #     → [README.md](03_guidelines/frameworks/fastapi/README.md)
│       └── python-cli/           #     → [README.md](03_guidelines/frameworks/python-cli/README.md)
│
│
└── templates/                    # プロジェクトテンプレート [英語]
    └── python-cli/               #     → [README.md](templates/python-cli/README.md)
```

</details>

<details>
<summary>運用・バージョニング</summary>

更新方針、フレームワーク追加手順、バージョニングの詳細は **[docs/operation-guide.md](../../../docs/operation-guide.md)** を参照してください。

**更新頻度の目安**

| セクション | 更新頻度 | 影響範囲 |
|-----------|---------|---------|
| `01_philosophy/` | 極めて稀 | 全プロジェクト（MAJOR 変更） |
| `02_decision-criteria/` | 稀 | 全プロジェクト |
| `03_guidelines/common/` | 中頻度 | 全プロジェクト |
| `03_guidelines/frameworks/` | 高頻度 | 該当 FW のプロジェクトのみ |
| `templates/` | 中頻度 | 新規プロジェクトのみ |

**フレームワークの追加**

新しいフレームワーク（例: FastAPI, Flask, Django）を追加する場合:

1. `03_guidelines/frameworks/{framework}/` に `overview.md` と `project-structure.md` を作成
2. `common/` との責務分離を徹底（共通ルール → common、FW 固有パターン → frameworks）
3. `templates/{framework}/` にテンプレートを用意
4. 本 README のディレクトリ構造を更新

詳細な手順とチェックリストは [docs/operation-guide.md](../../../docs/operation-guide.md) を参照。

**バージョニング** — セマンティックバージョニング（git タグ）で管理します。

| 変更種別 | バージョン | 例 |
|---------|----------|------|
| 設計思想・判断基準の大幅変更 | MAJOR | v2.0.0 |
| ガイドライン追加・改善 | MINOR | v1.1.0 |
| 誤字修正、補足追加 | PATCH | v1.0.1 |

submodule を特定のタグに固定:

```bash
cd docs/ai-dev-os
git checkout v1.2.0
cd ../..
git add docs/ai-dev-os
git commit -m "chore: pin ai-dev-os to v1.2.0"
```

</details>

## 言語ポリシー

- `01_philosophy/` と `02_decision-criteria/` には**英語のサンプルコンテンツ**が含まれています。クローン後にチームの抽象的思考と意思決定フレームワークのニュアンスを保つため、**母国語で書き直してください**。
- その他のセクションは AI 互換性と国際的なアクセシビリティのため**英語**で記述しています。
- 多言語の運用ガイドは `docs/i18n/` にあります。

## 関連

| リポジトリ | 説明 |
|---|---|
| [ai-dev-os](https://github.com/yunbow/ai-dev-os) | フレームワーク仕様と理論 |
| [rules-typescript](https://github.com/yunbow/ai-dev-os-rules-typescript) | TypeScript / Next.js / Node.js ガイドライン |
| [plugin-claude-code](https://github.com/yunbow/ai-dev-os-plugin-claude-code) | Claude Code 向けの Skills、Hooks、Agents |
| [plugin-kiro](https://github.com/yunbow/ai-dev-os-plugin-kiro) | Kiro 向けの Steering Rules と Hooks |
| [plugin-cursor](https://github.com/yunbow/ai-dev-os-plugin-cursor) | ガイドライン駆動開発のための Cursor Rules (.mdc) |
| [cli](https://github.com/yunbow/ai-dev-os-cli) | セットアップ自動化 — `npx ai-dev-os init` |

## ライセンス

[MIT](../../../LICENSE)

---

Languages: [English](../../../README.md) | 日本語 | [简体中文](../zh-CN/README.md) | [한국어](../ko/README.md) | [Español](../es/README.md)
