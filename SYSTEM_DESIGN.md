# AI支援用語集システム設計書

## 🎯 **システム概要**

技術に明るくない読者でも簡単に用語集に貢献できるAI支援システム

## 🏗️ **推奨構成**

### **フロントエンド：GitHub Issue Template**

```markdown
---
name: 用語追加・編集
about: 新しい用語の追加や既存用語の編集
title: "[用語] "
labels: terminology
assignees: ''
---

## 用語名
<!-- 例：メイドロイド -->

## カテゴリ（わからなければ空欄でOK）
<!-- 例：ハードウェア、人物、組織、技術 など -->

## 説明
<!-- 自由な文章で説明してください。箇条書きでもOKです -->

## 関連する用語（あれば）
<!-- 例：ミニマルコンピュータ、TRANS-S など -->

## その他の情報
<!-- 価格、仕様、場所、人物の年齢など、何でも -->
```

### **バックエンド：GitHub Actions**

```yaml
name: AI用語集処理
on:
  issues:
    types: [opened, edited]
    
jobs:
  process_terminology:
    runs-on: ubuntu-latest
    if: contains(github.event.issue.labels.*.name, 'terminology')
    
    steps:
    - name: AIによる構造化処理
      uses: ./.github/actions/ai-processor
      with:
        openai_api_key: ${{ secrets.OPENAI_API_KEY }}
        issue_body: ${{ github.event.issue.body }}
        
    - name: Wiki更新
      uses: ./.github/actions/wiki-updater
      with:
        structured_data: ${{ steps.ai-processor.outputs.markdown }}
```

## 🤖 **AIプロンプト設計**

### **用語分類プロンプト**
```
あなたは「Kuon Node Project」の用語集管理AIです。
以下のユーザー入力を分析し、適切なMarkdown形式に構造化してください。

入力: {user_input}

出力形式:
# {用語名}
{説明文}

## カテゴリ
{自動判定されたカテゴリ}

**関連用語**: {関連用語リスト}
**タグ**: {検索用タグ}

判定基準:
- システム・インフラ: Kuon Node, 端末類
- ハードウェア: メイドロイド, ミニコン類  
- 接続規格: TRANS-S, 火力正義
- 商業・文化: 店舗, 組織, 文化
- 人物: キャラクター, 開発者
```

## 🔧 **実装オプション比較**

### **Option 1: GitHub Actions + OpenAI API** ⭐️推奨
**利点**:
- GitHub Wikiとの統合が完璧
- 無料枠内で運用可能（月10〜20ドル程度）
- メンテナンス不要
- バージョン管理自動

**欠点**:
- API遅延（30秒〜2分）
- OpenAI依存

**コスト**: $10-30/月

### **Option 2: Vercel + OpenAI API** 
**利点**:
- リアルタイム処理
- カスタムUI可能
- スケーラブル

**欠点**:
- 開発コスト高
- GitHub Wiki連携が複雑

**コスト**: $20-50/月

### **Option 3: ローカルLLM（Ollama/LM Studio）**
**利点**:
- 完全無料
- プライバシー保護
- カスタマイズ自由

**欠点**:
- セットアップ複雑
- 性能要件高（GPU必要）
- メンテナンス必要

**コスト**: $0（ハードウェア要件：RTX 4060以上推奨）

### **Option 4: 混合方式**
**利点**:
- コスト効率最高
- 段階的導入可能
- 運用負荷最小

**構成**:
- 即座の更新：GitHub Actions + OpenAI
- 大量処理：週次バッチでローカル処理

## 📋 **実装フェーズ**

### **Phase 1: MVP（最小限実装）**
1. GitHub Issue Templateの作成
2. 基本的なGitHub Actionsワークフロー
3. OpenAI APIとの連携
4. 自動Wiki更新

**期間**: 1-2週間  
**コスト**: $10/月

### **Phase 2: UI改善**
1. GitHub Discussionsとの連携
2. より直感的な入力フォーム
3. プレビュー機能

**期間**: 2-3週間

### **Phase 3: 高度化**
1. 用語間のリンク自動生成
2. 重複検出・マージ機能
3. 多言語対応

**期間**: 1-2ヶ月

## 🚀 **具体的な開始手順**

1. **GitHub Secretsの設定**
   - `OPENAI_API_KEY`の追加

2. **Issue Templateの作成**
   - `.github/ISSUE_TEMPLATE/`配下に配置

3. **GitHub Actionsの実装**
   - 基本的なAI処理ワークフロー

4. **テスト用語での動作確認**

5. **ユーザー向けドキュメントの作成**

この構成により、「テキスト入力だけでプロ品質の用語集が自動生成される」システムが実現できます。