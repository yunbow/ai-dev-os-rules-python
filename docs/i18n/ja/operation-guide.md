# Operation Guide

本書は ai-dev-os の更新・拡張に関する運用ルールを定義します。

---

## 1. セクション別の更新方針

### 1.1 更新頻度と影響範囲

| セクション | 更新頻度 | 影響範囲 | 注意点 |
|-----------|---------|---------|--------|
| `01_philosophy/` | 極めて稀 | 全プロジェクト | 根幹思想の変更は MAJOR バージョンアップ |
| `02_decision-criteria/` | 稀 | 全プロジェクト | 判断基準の変更は下流ガイドラインとの整合性を確認 |
| `03_guidelines/common/` | 中頻度 | 全プロジェクト | 共通規約の変更は全フレームワークへの影響を検証 |
| `03_guidelines/frameworks/` | 高頻度 | 該当 FW のプロジェクト | フレームワークのアップデートに追随 |
| `templates/` | 中頻度 | 新規プロジェクトのみ | 既存プロジェクトには影響しない |

### 1.2 更新時のルール

**全セクション共通:**
- プロジェクト固有の記述（特定サービス名、特定ドメイン用語）を含めない
- 具体例が必要な場合は汎用的なプレースホルダ（`{domain}`, `{決済サービス}` 等）を使用
- 変更後は `README.md` のディレクトリ構造を最新状態に更新

**言語ポリシー:**
- `01_philosophy/` と `02_decision-criteria/` には**英語のサンプルコンテンツ**が含まれています — クローン後に**母国語で書き直してください**（抽象的な思考・判断基準はニュアンスを保つため母国語が最適）
- その他のセクション（`03_guidelines/`, `templates/`）は**英語**で記述 — AI互換性と国際的なアクセシビリティのため
- 多言語の運用ガイドは `docs/i18n/`（JA, ZH, KO, ES）で管理
- コンテンツの追加・更新時は常にこの言語ポリシーに従うこと

**`01_philosophy/` の更新:**
- サンプルコンテンツ（英語）が含まれています。クローン後に**母国語で書き直してください**
- 新しい原則の追加は既存原則との矛盾がないか確認
- 削除・大幅変更は MAJOR バージョンとして扱う
- `02_decision-criteria/` 以下との整合性も合わせて確認

**`02_decision-criteria/` の更新:**
- サンプルコンテンツ（英語）が含まれています。クローン後に**母国語で書き直してください**
- 判断基準の変更は `03_guidelines/` の該当セクションにも反映が必要か確認
- 新しい判断軸の追加は、対応するガイドラインの有無を確認

**`03_guidelines/` の更新:**
- `common/` と `frameworks/` で内容が重複しないよう注意
- 共通化すべき内容が framework 固有に書かれていないか定期的にレビュー
- 詳細は「2. フレームワークガイドラインの追加」を参照

**`03_guidelines/common/` のリポジトリ間同期:**
同期手順の詳細、ファイルごとの同期テーブル、差分確認用コマンドは [CONTRIBUTING.md](./CONTRIBUTING.md) を参照してください。

**`templates/` の更新:**
- テンプレートは新規プロジェクト向け。既存プロジェクトへの自動適用はしない
- 設定ファイル（pyproject.toml, ruff 設定等）は `03_guidelines/` の規約と一致させる

---

## 2. フレームワークガイドラインの追加

### 2.1 追加の判断基準

新しいフレームワークガイドラインを追加する条件:

| 条件 | 必須/推奨 |
|------|----------|
| 2つ以上のプロジェクトで使用実績がある | 必須 |
| `common/` では表現できないフレームワーク固有のパターンがある | 必須 |
| フレームワークが安定版（v1.0+）である | 推奨 |
| 長期的にメンテナンスされる見込みがある | 推奨 |

### 2.2 ディレクトリ構成

```
03_guidelines/frameworks/{framework-name}/
├── overview.md            # 技術スタック定義（必須）
├── project-structure.md   # ディレクトリ構成（必須）
└── ...                    # フレームワーク固有のガイドライン
```

### 2.3 追加手順

#### Step 1: overview.md の作成

全フレームワークガイドラインに必須のエントリポイント。以下を含める:

```markdown
# {Framework} Technology Stack

## Core
- Framework: {Name} v{Version}
- Language: Python（型ヒント付き、mypy/pyright strict モード）
- Package Manager: {uv / pip / poetry}

## Recommended Libraries
{カテゴリごとに推奨ライブラリを列挙}

## Prerequisites
{common/ ガイドラインとの関係を明記}
```

#### Step 2: project-structure.md の作成

ディレクトリ構成のガイドライン。以下の原則に従う:

- `common/` で定義済みの概念（垂直スライス、依存ルール等）はそのまま適用
- フレームワーク固有のディレクトリ規約のみ記述
- `{domain}` のようなプレースホルダで具体例を示す

#### Step 3: フレームワーク固有ガイドラインの追加

`common/` に対応するガイドラインがある場合の関係:

| パターン | 対応 |
|---------|------|
| common の内容をそのまま適用 | framework 側にはファイルを作らない |
| common を拡張する内容がある | framework 側に作成し、common を参照しつつ差分を記述 |
| framework 固有の概念 | framework 側にのみ作成 |

**ファイル命名規則:**
- `common/` と対応する場合は同名にする（例: `common/security.md` → `python-cli/security.md`）
- framework 固有の概念は説明的な名前を付ける（例: `dependency-injection.md`, `cli-commands.md`）

#### Step 4: テンプレートの作成

```
templates/{framework-name}/
├── CLAUDE.md.template     # CLAUDE.md テンプレート（必須）
├── submodule-setup.sh     # セットアップスクリプト（推奨）
├── .claude/skills/        # Claude Code スキル（推奨）
└── {設定ファイル群}        # pyproject.toml, ruff 設定等
```

#### Step 5: README.md の更新

- ディレクトリ構造に新フレームワークを追記
- 必要に応じて導入方法セクションにフレームワーク別の手順を追加

### 2.4 common/ との責務分離の原則

```
common/          → 「何をすべきか」（言語・FW 非依存のルール）
frameworks/xxx/  → 「どう実現するか」（FW 固有の実装パターン）
```

**例: バリデーション**
- `common/validation.md` → 「サーバーサイドバリデーション必須」「Pydantic 推奨」
- `frameworks/python-cli/validation.md` → 「Pydantic モデルと Typer 引数パースでのバリデーションパターン」
- `frameworks/fastapi/validation.md` → 「Pydantic モデルと依存性注入によるリクエストボディバリデーション」

**例: エラーハンドリング**
- `common/error-handling.md` → 「エラー分類」「ユーザー表示の原則」
- `frameworks/python-cli/error-handling.md` → 「カスタム例外階層と Typer 終了コード」

### 2.5 追加時のチェックリスト

- [ ] overview.md と project-structure.md が作成されている
- [ ] プロジェクト固有の記述が含まれていない
- [ ] `common/` との内容重複がない
- [ ] `common/` を参照すべき箇所で適切にリンクしている
- [ ] `templates/{framework}/CLAUDE.md.template` が作成されている
- [ ] `README.md` のディレクトリ構造が更新されている
- [ ] MINOR バージョン以上でリリースされている

---

## 3. バージョニングと変更管理

バージョニングルール（MAJOR/MINOR/PATCH）については [CONTRIBUTING.md](./CONTRIBUTING.md) を参照してください。

**破壊的変更の扱い:** ファイルの削除・リネームは MAJOR バージョンとして扱います。ファイルを移動する場合は、旧パスに一行リダイレクトを残し、次の MAJOR バージョンで削除します。

---

Languages: [English](../../operation-guide.md) | 日本語 | [简体中文](../zh-CN/operation-guide.md) | [한국어](../ko/operation-guide.md) | [Español](../es/operation-guide.md)
