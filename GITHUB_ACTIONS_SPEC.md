# 🤖 GitHub Actions自動化仕様書

## 📋 **概要**

現在の手動処理から段階的に自動化を導入するためのGitHub Actions仕様。小規模から大規模まで対応可能な拡張可能設計。

## 🎯 **自動化段階**

### **Stage 1: 手動処理支援（現在）** ✅
- testing branchの変更通知
- 処理状況の可視化
- 手動プロセスの支援

### **Stage 2: セミオート処理**
- testing branchの自動分析  
- 処理待ちの通知・レポート生成
- 承認後の自動統合

### **Stage 3: フルオート処理**
- ローカルLLM/クラウドLLMでの完全自動処理
- 品質チェック・自動統合
- 例外処理・ロールバック対応

## 📁 **ファイル構成**

```
.github/
├── workflows/
│   ├── stage1-manual-support.yml       # Stage 1: 手動支援
│   ├── stage2-semi-auto.yml           # Stage 2: セミオート  
│   ├── stage3-full-auto.yml           # Stage 3: フルオート
│   └── rollback.yml                   # 緊急ロールバック
├── scripts/
│   ├── analyze_changes.py             # 変更分析スクリプト
│   ├── local_llm_process.py           # ローカルLLM処理
│   └── quality_check.py               # 品質チェック
└── templates/
    ├── processing_report.md           # 処理レポート
    └── notification.md                # 通知テンプレート
```

---

## 🥇 **Stage 1: 手動処理支援**

### **ワークフロー: `.github/workflows/stage1-manual-support.yml`**

```yaml
name: "📋 手動処理支援システム"

on:
  push:
    branches: [testing]
  schedule:
    # 毎日9時に処理状況確認
    - cron: '0 0 * * *'
  workflow_dispatch:
    inputs:
      action:
        description: '実行アクション'
        required: true
        type: choice
        options:
        - 'analyze_changes'
        - 'generate_report'
        - 'cleanup_testing'

jobs:
  analyze_testing_branch:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' || github.event.inputs.action == 'analyze_changes'
    
    steps:
    - name: "📥 リポジトリチェックアウト"
      uses: actions/checkout@v4
      with:
        fetch-depth: 0  # 全履歴取得

    - name: "🔍 変更内容分析"
      id: analyze
      run: |
        # main..testingの差分を分析
        CHANGES=$(git diff main..testing --name-only)
        COMMIT_COUNT=$(git rev-list --count main..testing)
        LAST_PROCESSING=$(git log --grep="📖 用語集更新" --oneline -1 --format="%cd" --date=short || echo "未処理")
        
        echo "changed_files<<EOF" >> $GITHUB_OUTPUT
        echo "$CHANGES" >> $GITHUB_OUTPUT
        echo "EOF" >> $GITHUB_OUTPUT
        echo "commit_count=$COMMIT_COUNT" >> $GITHUB_OUTPUT
        echo "last_processing=$LAST_PROCESSING" >> $GITHUB_OUTPUT
        
        # 処理優先度を判定
        PRIORITY="低"
        if [ $COMMIT_COUNT -gt 10 ]; then
          PRIORITY="高"
        elif [ $COMMIT_COUNT -gt 5 ]; then
          PRIORITY="中"
        fi
        echo "priority=$PRIORITY" >> $GITHUB_OUTPUT

    - name: "📊 処理レポート生成"
      uses: actions/github-script@v7
      with:
        script: |
          const changedFiles = `${{ steps.analyze.outputs.changed_files }}`;
          const commitCount = '${{ steps.analyze.outputs.commit_count }}';
          const lastProcessing = '${{ steps.analyze.outputs.last_processing }}';
          const priority = '${{ steps.analyze.outputs.priority }}';
          
          const report = `## 📋 Processing Status Report
          
          ### 📊 **変更サマリー**
          - **処理待ちコミット数**: ${commitCount}件
          - **最終処理日**: ${lastProcessing}
          - **処理優先度**: ${priority}
          
          ### 📝 **変更ファイル**
          \`\`\`
          ${changedFiles}
          \`\`\`
          
          ### 🚀 **推奨アクション**
          ${priority === '高' ? 
            '⚠️ **至急処理推奨**: 大量の変更が蓄積されています' : 
            priority === '中' ? 
            '📅 **今週中に処理**: 適度な変更が蓄積されています' : 
            '✅ **余裕のある時に処理**: 少量の変更です'}
          
          ### 🔧 **処理コマンド**
          \`\`\`bash
          git checkout testing
          git pull origin testing
          # LOCAL_PROCESSING_GUIDE.mdの手順に従って処理
          \`\`\`
          `;
          
          // Discussionを作成
          await github.rest.teams.addOrUpdateRepoPermissions({
            org: context.repo.owner,
            team_slug: 'maintainers',
            owner: context.repo.owner,
            repo: context.repo.repo,
            permission: 'admin'
          }).catch(() => {}); // チーム存在しない場合はスキップ
          
          // Issueとして投稿
          await github.rest.issues.create({
            owner: context.repo.owner,
            repo: context.repo.repo,
            title: `📋 処理状況レポート (${new Date().toISOString().split('T')[0]})`,
            body: report,
            labels: ['processing-report', `priority-${priority}`]
          });

  scheduled_reminder:
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'
    
    steps:
    - name: "⏰ 定期リマインダー"
      uses: actions/github-script@v7  
      with:
        script: |
          // 未処理の処理レポートをチェック
          const issues = await github.rest.issues.listForRepo({
            owner: context.repo.owner,
            repo: context.repo.repo,
            labels: 'processing-report',
            state: 'open'
          });
          
          if (issues.data.length > 0) {
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issues.data[0].number,
              body: '⏰ **定期リマインダー**: testing branchの処理をお忘れではありませんか？'
            });
          }
```

---

## 🥈 **Stage 2: セミオート処理**

### **ワークフロー: `.github/workflows/stage2-semi-auto.yml`**

```yaml
name: "🔄 セミオート処理システム"

on:
  workflow_dispatch:
    inputs:
      processing_mode:
        description: '処理モード'
        required: true
        type: choice
        options:
        - 'analyze_only'      # 分析のみ
        - 'process_with_approval'  # 承認後統合
        - 'emergency_rollback'     # 緊急ロールバック

jobs:
  semi_auto_processing:
    runs-on: ubuntu-latest
    environment: 'terminology-processing'  # 手動承認環境
    
    steps:
    - name: "📥 準備"
      uses: actions/checkout@v4
      with:
        fetch-depth: 0
        token: ${{ secrets.GITHUB_TOKEN }}

    - name: "🐍 Python環境準備"
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
        
    - name: "📦 依存関係インストール"
      run: |
        pip install requests openai anthropic ollama-python
        
    - name: "🔍 変更分析・LLM処理"
      id: process
      run: |
        cat > process_changes.py << 'EOF'
        import subprocess
        import json
        import sys
        from datetime import datetime

        def get_git_changes():
            """testing branchの変更を取得"""
            result = subprocess.run(['git', 'diff', 'main..testing'], 
                                  capture_output=True, text=True)
            return result.stdout

        def call_local_llm(changes_text):
            """ローカルLLM（Ollama）を呼び出し"""
            try:
                import requests
                
                prompt = f"""
                あなたは「Kuon Node Project」の用語集管理専門家です。
                以下のGit差分から新しい用語や変更を分析し、適切なMarkdown形式に構造化してください。
                
                変更内容:
                {changes_text}
                
                出力形式:
                # {{用語名}}
                
                {{説明文}}
                
                ## 基本情報
                **カテゴリ**: {{カテゴリ名}}
                **関連用語**: {{関連用語}}
                
                重要: 日本語で出力し、既存の世界観に合致するよう分類してください。
                """
                
                # Ollamaローカルサーバーに接続を試行
                response = requests.post('http://localhost:11434/api/generate',
                    json={
                        'model': 'llama3.1:8b',
                        'prompt': prompt,
                        'stream': False
                    },
                    timeout=300)  # 5分タイムアウト
                
                if response.status_code == 200:
                    return response.json().get('response', '処理エラー')
                else:
                    return f"ローカルLLMエラー (HTTP {response.status_code})"
                    
            except Exception as e:
                return f"ローカルLLM接続エラー: {e}"

        def main():
            changes = get_git_changes()
            
            if not changes.strip():
                print("変更なし")
                sys.exit(0)
                
            # LLM処理
            processed = call_local_llm(changes)
            
            # 結果保存
            with open('processed_terms.md', 'w', encoding='utf-8') as f:
                f.write(processed)
                
            # GitHub Actions用の出力
            print(f"::set-output name=has_changes::true")
            print(f"::set-output name=processed_content::{processed[:1000]}...")  # 最初の1000文字

        if __name__ == "__main__":
            main()
        EOF
        
        python process_changes.py

    - name: "👀 人間による承認待ち"
      if: steps.process.outputs.has_changes == 'true'
      uses: trstringer/manual-approval@v1
      with:
        secret: ${{ github.TOKEN }}
        approvers: ${{ github.repository_owner }}
        minimum-approvals: 1
        issue-title: "🔍 用語処理内容の確認・承認"
        issue-body: |
          ## 🤖 自動処理結果
          
          以下の内容でGLOSSARY.mdを更新します：
          
          <details>
          <summary>📋 処理結果プレビュー</summary>
          
          ```markdown
          ${{ steps.process.outputs.processed_content }}
          ```
          
          </details>
          
          ### ✅ 承認する場合
          このIssueに「approve」とコメントしてください
          
          ### ❌ 却下する場合  
          このIssueに「reject」とコメントして理由を記載してください

    - name: "📝 承認後の統合処理"
      if: success()
      run: |
        # GLOSSARY.mdに統合
        echo "" >> GLOSSARY.md
        echo "<!-- $(date '+%Y-%m-%d %H:%M') セミオート処理分 -->" >> GLOSSARY.md
        cat processed_terms.md >> GLOSSARY.md
        
        # コミット
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add GLOSSARY.md
        git commit -m "📖 セミオート用語処理: $(date '+%Y-%m-%d %H:%M')

        - 処理モード: セミオート（承認済み）
        - 処理済みコミット数: $(git rev-list --count main..testing)件
        - 処理日時: $(date)"
        
        git push origin main
        
        # testing branchをリセット
        git checkout testing  
        git reset --hard main
        git push -f origin testing
```

---

## 🥇 **Stage 3: フルオート処理**

### **ワークフロー: `.github/workflows/stage3-full-auto.yml`**

```yaml
name: "⚡ フルオート処理システム"

on:
  push:
    branches: [testing]
  schedule:
    # 毎日深夜2時に自動処理
    - cron: '0 17 * * *'  # UTC時間（日本時間2時）

jobs:
  full_auto_processing:
    runs-on: self-hosted  # セルフホストランナー（ローカルLLM用）
    # または: runs-on: ubuntu-latest  # クラウドLLM使用時
    
    steps:
    - name: "🚀 フル自動処理開始"
      uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: "🔍 変更量チェック"
      id: change_check
      run: |
        COMMIT_COUNT=$(git rev-list --count main..testing)
        CHANGE_LINES=$(git diff main..testing | wc -l)
        
        echo "commit_count=$COMMIT_COUNT" >> $GITHUB_OUTPUT
        echo "change_lines=$CHANGE_LINES" >> $GITHUB_OUTPUT
        
        # 安全性チェック（大量変更時は手動処理にフォールバック）
        if [ $COMMIT_COUNT -gt 50 ] || [ $CHANGE_LINES -gt 1000 ]; then
          echo "safety_check=failed" >> $GITHUB_OUTPUT
        else
          echo "safety_check=passed" >> $GITHUB_OUTPUT
        fi

    - name: "⚠️ 大量変更検出時のフォールバック"
      if: steps.change_check.outputs.safety_check == 'failed'
      uses: actions/github-script@v7
      with:
        script: |
          await github.rest.issues.create({
            owner: context.repo.owner,
            repo: context.repo.repo,
            title: '⚠️ 大量変更検出：手動処理が必要',
            body: `## 🚨 安全性チェック失敗
            
            フルオート処理の安全しきい値を超過しました：
            - コミット数: ${{ steps.change_check.outputs.commit_count }}件
            - 変更行数: ${{ steps.change_check.outputs.change_lines }}行
            
            **手動での処理をお願いします。**`,
            labels: ['urgent', 'manual-processing-required']
          });
          
          throw new Error('Safety check failed - manual processing required');

    - name: "🤖 AI自動処理"
      if: steps.change_check.outputs.safety_check == 'passed'
      run: |
        # 高品質AI処理スクリプト実行
        python .github/scripts/full_auto_process.py
        
    - name: "✅ 品質チェック"
      run: |
        python .github/scripts/quality_check.py
        
    - name: "🔄 自動統合"
      run: |
        git config user.name "KuonBot"
        git config user.email "bot@kuon-node-project.com"
        git add .
        git commit -m "🤖 フルオート用語処理: $(date '+%Y-%m-%d %H:%M')
        
        自動処理統計:
        - 処理済みコミット: ${{ steps.change_check.outputs.commit_count }}件
        - 処理済み行数: ${{ steps.change_check.outputs.change_lines }}行
        - 処理モード: フルオート
        - AI品質スコア: $(cat quality_score.txt)"
        
        git push origin main
        git push origin main:testing  # testing branchをリセット

    - name: "📊 処理レポート"
      if: success()
      uses: actions/github-script@v7
      with:
        script: |
          const qualityScore = require('fs').readFileSync('quality_score.txt', 'utf8');
          
          await github.rest.issues.create({
            owner: context.repo.owner,
            repo: context.repo.repo,
            title: `✅ フルオート処理完了 (${new Date().toLocaleDateString('ja-JP')})`,
            body: `## 🤖 自動処理完了レポート
            
            ### 📊 処理統計
            - **処理済みコミット**: ${{ steps.change_check.outputs.commit_count }}件
            - **処理済み行数**: ${{ steps.change_check.outputs.change_lines }}行
            - **AI品質スコア**: ${qualityScore}/100
            - **処理時間**: $(date)
            
            ### 🔗 関連リンク
            - [更新されたGLOSSARY.md](../blob/main/GLOSSARY.md)
            - [処理コミット](../commit/${context.sha})
            
            ${qualityScore > 90 ? '🎉 高品質な処理が完了しました！' : qualityScore > 70 ? '⚠️ 品質に注意が必要な可能性があります' : '🚨 手動レビューを推奨します'}`,
            labels: ['auto-processed', `quality-${Math.floor(qualityScore/20)*20}`]
          });
```

---

## 📜 **段階的導入スケジュール**

### **Phase 1: 準備期間（1-2週間）**
- [ ] Stage 1ワークフローの導入
- [ ] ローカルLLM環境の構築
- [ ] 手動プロセスの習熟

### **Phase 2: セミオート試行（2-4週間）**
- [ ] Stage 2ワークフローの導入
- [ ] 承認フローの試行・改善
- [ ] 品質基準の確立

### **Phase 3: フルオート本格運用（1-2ヶ月後）**
- [ ] セルフホストランナーのセットアップ
- [ ] Stage 3ワークフローの導入
- [ ] 運用監視・改善

## 🔧 **カスタマイズ設定**

### **環境変数**
```yaml
# リポジトリSecrets設定
OPENAI_API_KEY: (クラウドLLM使用時)
ANTHROPIC_API_KEY: (Claude使用時)
LOCAL_LLM_ENDPOINT: http://localhost:11434 (Ollama使用時)
QUALITY_THRESHOLD: 70 (品質閾値)
MAX_AUTO_COMMITS: 20 (自動処理最大コミット数)
```

### **設定ファイル: `.github/config/terminology.yml`**
```yaml
processing:
  local_llm:
    model: "llama3.1:8b"
    timeout: 300
    temperature: 0.3
  
  quality_check:
    min_score: 70
    required_fields: ["category", "description"]
  
  safety_limits:
    max_commits: 20
    max_change_lines: 500
    
  notifications:
    discord_webhook: (optional)
    slack_webhook: (optional)
```

この設計により、小規模プロジェクトから始めて段階的に自動化を導入できます！