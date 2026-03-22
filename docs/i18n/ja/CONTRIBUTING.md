# AI Dev OS Rules — Python へのコントリビューション

コントリビューションに興味をお持ちいただきありがとうございます！

## コントリビューション方法

### Issue の報告

- [GitHub Issues](https://github.com/yunbow/ai-dev-os-rules-python/issues) を使用してください
- 影響を受けるガイドラインファイルとルールを明記してください

### Pull Request

1. リポジトリをフォーク
2. フィーチャーブランチを作成
3. 変更を実施
4. 説明的なコミットメッセージでコミット
5. Pull Request を作成

### コントリビューション対象

| ディレクトリ | 求めている内容 | ガイドライン |
|-------------|--------------|------------|
| `01_philosophy/` | サンプルコンテンツの改善 | 母国語で書き直してください。抽象的に保ち、ツール/フレームワーク名は含めないでください |
| `02_decision-criteria/` | 新しい判断軸、改善された閾値 | 母国語で書き直してください。フレームワーク固有の詳細は含めないでください |
| `03_guidelines/common/` | ルールの改善、新しい Before/After 例、新しいルール | **クロスリポジトリ同期が必要** — 変更は `ai-dev-os-rules-typescript` にもコピーする必要があります。下記参照 |
| `03_guidelines/frameworks/` | フレームワーク固有のパターン、新しいフレームワークのサポート | 責務分離に従ってください: common/ = 「何をすべきか」、frameworks/ = 「どう実装するか」 |
| `templates/` | テンプレートの改善、新しいフレームワークテンプレート | 新規プロジェクト向けのみ。既存プロジェクトには自動適用しないでください |

### `common/` のクロスリポジトリ同期

`03_guidelines/common/` ディレクトリは rules リポジトリ間で共有されています。ファイルを更新する際は:

1. まずこのリポジトリで変更を行う
2. 更新したファイルを `ai-dev-os-rules-typescript` にコピー
3. 各リポジトリで同じメッセージでコミット
4. 例外: `code.md` は言語固有の例を含む場合があります — ルールは同期し、例は調整してください

**差分の簡易チェック:**

```bash
diff -rq 03_guidelines/common/ ../ai-dev-os-rules-typescript/03_guidelines/common/
```

### 新しいフレームワークの追加

1. `03_guidelines/frameworks/{name}/overview.md` と `project-structure.md` を作成
2. `common/` との重複なし — フレームワーク固有のパターンのみ
3. `templates/{name}/` に CLAUDE.md.template を作成
4. README.md のディレクトリ構造を更新
5. MINOR バージョン以上としてリリース

### バージョニング

| 変更種別 | バージョン |
|---------|----------|
| Philosophy/decision-criteria の変更 | MAJOR |
| 新しいガイドライン、フレームワークの追加 | MINOR |
| 誤字修正、例の改善 | PATCH |

### 翻訳ガイド

- 翻訳は `docs/i18n/{lang}/` に配置
- 対応言語: `ja`, `zh-CN`, `ko`, `es`
- `03_guidelines/` のガイドラインは英語で記述（AI 互換性のため）
- `01_philosophy/` と `02_decision-criteria/` は任意の言語で記述可能

## 行動規範

敬意を持ち、建設的で、包括的であること。

---

Languages: [English](../../../CONTRIBUTING.md) | 日本語 | [简体中文](../zh-CN/CONTRIBUTING.md) | [한국어](../ko/CONTRIBUTING.md) | [Español](../es/CONTRIBUTING.md)
