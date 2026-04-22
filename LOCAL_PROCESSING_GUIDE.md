# 📋 ローカルLLM用語分類 手順書

## 🎯 **概要**

testing branchに蓄積されたユーザーコントリビューションを、ローカルLLMで分析・分類してmain branchに統合する手順

## 🛠️ **事前準備**

### **1. ローカルLLM環境のセットアップ**

#### **推奨Option A: Ollama（軽量・簡単）**
```bash
# Windows
winget install Ollama.Ollama

# モデルのダウンロード（初回のみ）
ollama pull llama3.1:8b
# または軽量版
ollama pull llama3.1:3b
```

#### **推奨Option B: LM Studio（GUI・使いやすい）**
1. [LM Studio](https://lmstudio.ai)をダウンロード・インストール
2. 推奨モデル：
   - `microsoft/Phi-3.5-mini-instruct-GGUF` (軽量・高性能)
   - `meta-llama/Llama-3.2-3B-Instruct-GGUF` (バランス型)

### **2. 作業用ディレクトリの準備**
```bash
mkdir kuon-node-processing
cd kuon-node-processing
```

## 🔄 **処理フロー**

### **Phase 1: testing branchの取得**

```bash
# 1. リポジトリのクローン
git clone https://github.com/SideNoteJP/KuonNodeProject_OriginalScripts.git
cd KuonNodeProject_OriginalScripts

# 2. testing branchに切り替え
git checkout testing

# 3. 最新の変更を取得
git pull origin testing

# 4. 変更内容の確認
git log --oneline main..testing
git diff main..testing
```

### **Phase 2: 変更内容の分析**

```bash
# 新しいIssueやファイル変更の抽出
git diff main..testing --name-only > changed_files.txt

# 新規追加内容の抽出
git diff main..testing > changes.diff
```

### **Phase 3: ローカルLLMによる分類処理**

#### **Ollama使用時**
```bash
# プロンプト作成（prompt.txt）
cat > prompt.txt << 'EOF'
あなたは「Kuon Node Project」の用語集管理専門家です。
以下のGit差分から新しい用語や変更を分析し、適切なMarkdown形式に構造化してください。

分析対象:
{DIFF_CONTENT}

出力形式:
# {用語名}

{説明文（自然な日本語で）}

## 基本情報
**カテゴリ**: {以下から選択}
- システム・インフラ: Kuon Node, 端末類
- ハードウェア: メイドロイド, ミニコン類  
- 接続規格・技術: TRANS-S, 火力正義
- 商業・文化: パーツ天国, のっぽさん
- 人物・キャラクター: 登場人物
- 組織・団体: KIO, CENI
- OS・ソフトウェア: MinimalOS類
- 法制度・政治: 法改正案類

**関連用語**: {関連する既存用語があれば}

## 詳細
{追加の詳細情報があれば}

---
（複数の用語がある場合、上記形式を繰り返し）

重要: 必ず日本語で、既存の世界観に合致するよう分類してください。
EOF

# 差分内容をプロンプトに挿入
DIFF_CONTENT=$(cat changes.diff)
sed -i "s/{DIFF_CONTENT}/$DIFF_CONTENT/g" prompt.txt

# LLM実行
ollama run llama3.1:8b < prompt.txt > classified_terms.md
```

#### **LM Studio使用時**
```bash
# LM Studioのサーバーモードを起動後
curl -X POST http://localhost:1234/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "local-model",
    "messages": [
      {
        "role": "system",
        "content": "あなたは技術文書の分類専門家です。日本語で正確に分類してください。"
      },
      {
        "role": "user", 
        "content": "'"$(cat prompt.txt)"'"
      }
    ],
    "temperature": 0.3,
    "max_tokens": 2000
  }' > classified_terms.json

# JSON形式から内容抽出
cat classified_terms.json | jq -r '.choices[0].message.content' > classified_terms.md
```

### **Phase 4: 結果の統合と検証**

```bash
# 1. 既存のGLOSSARY.mdをバックアップ
cp GLOSSARY.md GLOSSARY.backup.md

# 2. 新しい用語をGLOSSARYに追加
echo "" >> GLOSSARY.md
echo "<!-- 以下、$(date '+%Y-%m-%d')処理分 -->" >> GLOSSARY.md
cat classified_terms.md >> GLOSSARY.md

# 3. 手動レビューと修正
# お好みのエディタでGLOSSARY.mdを確認・修正
code GLOSSARY.md  # VS Code
# または
notepad GLOSSARY.md  # メモ帳
```

### **Phase 5: main branchへの反映**

```bash
# 1. main branchに切り替え
git checkout main

# 2. 変更をマージ（手動確認）
git merge testing

# 3. 処理済みのマークを追加
git add .
git commit -m "📖 用語集更新: testing branchから$(git rev-list --count main..testing@{1})件の変更を統合

- ローカルLLM処理日: $(date '+%Y-%m-%d %H:%M')
- 処理したコミット: $(git log --oneline main..testing@{1} | wc -l)件
- 新規用語: [手動で記入]"

# 4. リモートにプッシュ
git push origin main

# 5. testing branchをクリア（オプション）
git push origin :testing  # リモートのtesting branchを削除
git push origin main:testing  # testing branchをmainと同期
```

## 🔧 **自動化スクリプト（オプション）**

### **一括処理スクリプト: `process_terminology.sh`**

```bash
#!/bin/bash
set -e

echo "🚀 Kuon Node Project 用語処理開始..."

# 設定
MODEL_NAME="llama3.1:8b"
REPO_URL="https://github.com/SideNoteJP/KuonNodeProject_OriginalScripts.git"

# リポジトリ更新
echo "📥 リポジトリ更新中..."
git fetch origin
git checkout testing
git pull origin testing

# 変更内容分析
echo "🔍 変更内容分析中..."
CHANGES=$(git diff main..testing)

if [ -z "$CHANGES" ]; then
    echo "ℹ️ testing branchに新しい変更はありません"
    exit 0
fi

echo "📝 変更数: $(echo "$CHANGES" | wc -l) lines"

# LLM処理
echo "🤖 ローカルLLM処理中..."
cat > temp_prompt.txt << EOF
あなたは「Kuon Node Project」の用語集管理専門家です。
以下のGit差分から新しい用語や変更を分析し、適切なMarkdown形式に構造化してください。

$CHANGES

[出力形式は前述の通り]
EOF

ollama run "$MODEL_NAME" < temp_prompt.txt > new_terms.md

# 結果統合
echo "📋 用語集更新中..."
git checkout main
echo "" >> GLOSSARY.md
echo "<!-- $(date '+%Y-%m-%d %H:%M') LLM処理分 -->" >> GLOSSARY.md
cat new_terms.md >> GLOSSARY.md

# コミット
git add .
git commit -m "📖 自動用語処理: $(date '+%Y-%m-%d %H:%M')"

echo "✅ 処理完了！手動レビュー後にpushしてください"
echo "📖 確認: git diff HEAD~1"
echo "🚀 プッシュ: git push origin main"

# クリーンアップ
rm temp_prompt.txt new_terms.md
```

### **使用方法**
```bash
chmod +x process_terminology.sh
./process_terminology.sh
```

## 📊 **処理時間・リソース目安**

| 処理内容 | 時間 | CPU使用率 | メモリ使用量 |
|---------|------|----------|------------|
| git操作 | 30秒 | 低 | <100MB |
| Ollama 3B | 1-3分 | 50-80% | 4GB |
| Ollama 8B | 3-5分 | 60-90% | 8GB |
| LM Studio | 2-4分 | 40-70% | 6GB |

## 🆘 **トラブルシューティング**

### **LLMが遅い**
- より軽いモデルを使用: `llama3.1:3b`
- 量子化レベルを下げる: `Q4_K_M` → `Q3_K_M`

### **日本語出力がおかしい**
- プロンプトに「必ず日本語で出力してください」を追加
- 温度パラメータを下げる: `temperature: 0.1`

### **GitHubへのプッシュが失敗**
```bash
# 認証情報の確認
git config --list | grep credential

# personal access tokenの再生成が必要な場合
git config credential.helper store
```

## 📅 **推奨スケジュール**

- **日次**: testing branchの変更確認
- **週次**: 蓄積された変更の処理・統合
- **月次**: 用語集の品質レビュー・重複整理

これで手動処理による効率的な用語集管理が可能になります！