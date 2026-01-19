# DeepSeek-R1 32B セットアップガイド

**目的**: DeepSeek-R1 32B をローカル環境（Ollama）でセットアップ
**推奨用途**: Atlas (Orchestrator) の推論・設計判断

⚠️ **重要**: このガイドは**手順のみ**を記載。**PC再起動後に実施すること**。

---

## システム要件

| 項目 | 最小要件 | 推奨 |
|------|---------|------|
| RAM | 32GB | 64GB |
| ディスク空き容量 | 25GB | 50GB |
| CPU | 8コア | 16コア |

**Mac Studio M2 Max 32GB の場合**:
- ギリギリ動作可能（スワップ発生のリスクあり）
- **他のアプリケーションを終了してから実行**

---

## 事前準備（必須）

### 1. メモリ解放

**PC再起動**が最も確実。または以下を終了:
- VSCode（3.8 GB 解放）
- Arc Browser のタブ整理（1-2 GB 解放）
- Slack、Discord（0.5 GB 解放）

**目標空きメモリ**: 10 GB 以上

### 2. ディスク空き容量確認

```bash
df -h ~
# 空き容量が 25GB 以上あることを確認
```

---

## セットアップ手順

### Step 1: モデルダウンロード

⚠️ **注意**: ダウンロードには 10-20分かかる

```bash
# DeepSeek-R1 32B をダウンロード
ollama pull deepseek-r1:32b
```

**ダウンロード中のメモリ使用**:
- ダウンロード自体は 1-2 GB
- 展開時に一時的に 5-10 GB 使用

---

### Step 2: インストール確認

```bash
# インストール済みモデルを確認
ollama list | grep deepseek-r1:32b

# 期待される出力:
# deepseek-r1:32b    [ID]    19 GB    [日時]
```

---

### Step 3: メモリ監視スクリプトの準備

動作検証時にメモリを監視するスクリプトを準備。

```bash
# メモリ監視スクリプトを作成
cat > ~/workspace-ai/nomuraya-llm/model-deepseek/monitor-memory.sh <<'EOF'
#!/bin/bash
# メモリ監視スクリプト

echo "メモリ監視開始（Ctrl+C で停止）"
echo "---"

while true; do
    date '+%Y-%m-%d %H:%M:%S'
    vm_stat | perl -ne '/page size of (\d+)/ and $size=$1; /Pages\s+([^:]+)[^\d]+(\d+)/ and printf("%-16s % 16.2f Mi\n", "$1:", $2 * $size / 1048576);' | grep -E "(free|active|wired)"
    echo "---"
    sleep 5
done
EOF

chmod +x ~/workspace-ai/nomuraya-llm/model-deepseek/monitor-memory.sh
```

---

### Step 4: 動作検証（軽量テスト）

⚠️ **重要**: 初回は**軽量プロンプト**でテスト

```bash
# ターミナル1: メモリ監視
~/workspace-ai/nomuraya-llm/model-deepseek/monitor-memory.sh

# ターミナル2: DeepSeek-R1 32B を起動
echo "Hello" | ollama run deepseek-r1:32b
```

**判定基準**:
- Free メモリが 2 GB 以上 → **安全、本格利用可能**
- Free メモリが 1-2 GB → **注意、短時間利用のみ**
- Free メモリが 1 GB 未満 → **危険、即座に停止**

---

### Step 5: 停止方法

```bash
# プロセスを停止
pkill -f "ollama.*deepseek-r1:32b"

# または Ctrl+C で終了
```

---

## 本格利用（動作検証後）

### テストスクリプト作成

```python
# ~/workspace-ai/nomuraya-llm/model-deepseek/test-deepseek-r1-32b.py
import subprocess
import json

def test_deepseek_r1_32b():
    """DeepSeek-R1 32B の動作確認"""

    messages = [
        {
            "role": "system",
            "content": "あなたは Atlas、Orchestrator です。全リポジトリ・全LLMの運用効率を最適化するメタレベルの存在です。"
        },
        {
            "role": "user",
            "content": "自己紹介してください。あなたの役割、思考の軸、判断基準を簡潔に説明してください。"
        }
    ]

    # Ollama API 呼び出し（JSON形式）
    payload = {
        "model": "deepseek-r1:32b",
        "messages": messages,
        "stream": False
    }

    # curl で API 呼び出し
    cmd = [
        "curl", "-X", "POST",
        "http://localhost:11434/api/chat",
        "-H", "Content-Type: application/json",
        "-d", json.dumps(payload)
    ]

    result = subprocess.run(cmd, capture_output=True, text=True)
    response = json.loads(result.stdout)

    print("=" * 60)
    print("Atlas からの応答:")
    print("=" * 60)
    print(response['message']['content'])
    print("=" * 60)

if __name__ == "__main__":
    test_deepseek_r1_32b()
```

```bash
# 実行
python ~/workspace-ai/nomuraya-llm/model-deepseek/test-deepseek-r1-32b.py
```

---

## トラブルシューティング

### エラー 1: メモリ不足

```
Error: out of memory
```

**対処**:
1. 他のアプリケーションを終了
2. PC再起動
3. DeepSeek-R1 8B に切り替え

---

### エラー 2: ダウンロード失敗

```
Error: failed to pull model
```

**対処**:
1. ディスク空き容量を確認
2. インターネット接続を確認
3. 再試行: `ollama pull deepseek-r1:32b`

---

### エラー 3: 起動が遅い

**症状**: `ollama run` が応答しない

**原因**: メモリスワップ発生

**対処**:
1. 即座に Ctrl+C で停止
2. `pkill -f ollama`
3. メモリ解放してから再試行

---

## ロールバック

### モデル削除

```bash
# DeepSeek-R1 32B を削除（20 GB 解放）
ollama rm deepseek-r1:32b

# 確認
ollama list | grep deepseek-r1:32b
# 何も表示されなければ削除成功
```

---

## 推奨運用

### Atlas の用途別モデル選択

| 用途 | 推奨モデル | 理由 |
|------|----------|------|
| ADR作成、設計判断 | DeepSeek-R1 32B | 推論能力が必要 |
| 通常のタスク調整 | DeepSeek-R1 8B | 軽量、安定 |
| 緊急時 | Claude Sonnet 4.5 | Claude Code統合 |

### メモリ監視の習慣化

```bash
# セッション開始時にメモリ確認
vm_stat | perl -ne '/page size of (\d+)/ and $size=$1; /Pages\s+([^:]+)[^\d]+(\d+)/ and printf("%-16s % 16.2f Mi\n", "$1:", $2 * $size / 1048576);' | grep free

# Free が 10 GB 以上なら 32B 実行可能
```

---

## 次のステップ

1. **PC再起動**
2. **このガイドに従ってセットアップ**
3. **軽量テストで動作確認**
4. **メモリ監視しながら本格利用**

---

**作成日**: 2026-01-11
**作成者**: Atlas (Orchestrator)
**ステータス**: PC再起動後に実施
