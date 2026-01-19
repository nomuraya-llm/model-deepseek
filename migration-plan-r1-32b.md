# DeepSeek-R1 32B 移行計画（メモリ安全性重視）

**対象**: Atlas (Orchestrator) の実行環境
**システムスペック**: Mac Studio M2 Max, 32GB RAM
**リスク**: メモリ不足によるシステムクラッシュ
**方針**: 段階的移行、メモリ監視、ロールバック準備

---

## 現状分析

### システムメモリ状況（2026-01-11）

```
Free:        946 MB
Active:      11 GB
Wired:       3.9 GB
Compressed:  22 GB → 5.3 GB
実質空き:    10-12 GB
```

### DeepSeek-R1 32B 要件

| 項目 | 要件 |
|------|------|
| ディスク | 20 GB（量子化版 Q4） |
| RAM（実行時） | 20-24 GB |
| **判定** | **現在のシステムでは不足** |

**根拠**:
- DeepSeek-R1 8B: 5.2 GB（ディスク）、約 8-10 GB（実行時RAM）
- 32B は 8B の 4倍 → 20-24 GB RAM 必要
- 現在の空きメモリ: 10-12 GB → **不足**

---

## リスク評価

### 高リスク要因

1. **メモリスワップ発生**
   - 物理RAM不足 → ディスクスワップ → システム激遅
   - SSD寿命への影響

2. **システムクラッシュ**
   - OOM Killer 発動 → プロセス強制終了
   - VSCode, Claude Code も巻き添え

3. **他アプリケーションへの影響**
   - Chrome, VSCode, Slack 等が強制終了される可能性

### 許容できるリスク

- モデルのダウンロード失敗（ロールバック可能）
- 軽量プロンプトでのテスト失敗（システムへの影響なし）

---

## 移行戦略: 3段階アプローチ

### Phase 1: 準備（リスク: 低）

**目的**: ダウンロードとメモリベンチマーク

**手順**:
1. **他アプリケーションを終了**
   - Chrome, Slack, その他メモリ消費アプリ
   - VSCode と Claude Code のみ残す

2. **メモリ監視スクリプト作成**
   ```bash
   # ~/.claude-config/scripts/monitor-memory.sh
   while true; do
     vm_stat | perl -ne '/page size of (\d+)/ and $size=$1; /Pages\s+([^:]+)[^\d]+(\d+)/ and printf("%-16s % 16.2f Mi\n", "$1:", $2 * $size / 1048576);'
     echo "---"
     sleep 5
   done
   ```

3. **モデルダウンロード**
   ```bash
   ollama pull deepseek-r1:32b
   ```

**判定基準**:
- ダウンロード完了 → Phase 2 へ
- ダウンロード失敗 → ディスク容量不足、要調査

---

### Phase 2: 軽量テスト（リスク: 中）

**目的**: 実行時メモリ使用量の実測

**手順**:
1. **ベースラインメモリ記録**
   ```bash
   bash ~/.claude-config/scripts/monitor-memory.sh > /tmp/memory-baseline.log &
   MONITOR_PID=$!
   ```

2. **軽量プロンプトで起動**
   ```bash
   echo "Hello" | ollama run deepseek-r1:32b --verbose
   ```

3. **メモリ使用量を確認**
   - Free メモリが 2 GB 以下 → **危険、Phase 3 中止**
   - Free メモリが 2-4 GB → **注意、短時間テストのみ**
   - Free メモリが 4 GB 以上 → Phase 3 へ

4. **モニター停止**
   ```bash
   kill $MONITOR_PID
   ```

**判定基準**:
- 実行時 Free メモリ ≥ 4 GB → Phase 3 へ
- 実行時 Free メモリ < 2 GB → **移行中止、代替案へ**

---

### Phase 3: 実用テスト（リスク: 高）

**注意**: Phase 2 で安全性が確認された場合のみ実施

**手順**:
1. **中程度のプロンプトでテスト**
   ```bash
   ollama run deepseek-r1:32b "以下のコードをレビューしてください: [100行程度のコード]"
   ```

2. **長時間実行テスト**
   ```bash
   # 複数ターンの会話をシミュレート
   ollama run deepseek-r1:32b <<EOF
   User: タスク1
   Assistant: [応答待ち]
   User: タスク2
   EOF
   ```

3. **メモリリーク確認**
   - 10分間実行して Free メモリが減り続けないか確認

**判定基準**:
- 安定動作 → 移行完了
- メモリ不足警告 → 移行中止

---

## ロールバック計画

### Phase 2/3 で問題が発生した場合

1. **即座にプロセス終了**
   ```bash
   pkill -f "ollama.*deepseek-r1:32b"
   ```

2. **モデル削除（オプション）**
   ```bash
   ollama rm deepseek-r1:32b
   ```

3. **代替案への切り替え**
   - DeepSeek-R1 8B（既にインストール済み）
   - Kimi K2 API（リモート実行）

---

## 代替案: リモート実行

### Option A: Kimi K2 API（推奨）

**メリット**:
- メモリ制約なし
- 256K コンテキスト
- エージェント特化

**セットアップ**:
- `~/workspace-ai/nomuraya-llm/model-kimi/api-setup.md` 参照
- Moonshot API キー取得
- OpenAI互換APIで即座に使用可能

### Option B: Google Colab / Modal

**メリット**:
- A100 GPU で高速実行
- メモリ制約なし

**セットアップ**:
- `~/workspace-ai/nomuraya-llm/model-kimi/cloud-setup.md` 参照
- Colab Pro: $10/月
- Modal: ~$3/時間

---

## 推奨アクション

### 即座に実施すべきこと

1. **Phase 1 を実施**（リスク: 低）
   - モデルダウンロードのみ
   - メモリ監視スクリプト作成

2. **Phase 2 で判定**
   - Free メモリ ≥ 4 GB → Phase 3 へ
   - Free メモリ < 2 GB → Kimi K2 API へ切り替え

### 長期的な推奨

- **Atlas 専用**: Kimi K2 API（メモリ制約なし、256K コンテキスト）
- **ローカルテスト**: DeepSeek-R1 8B（軽量、安定）
- **ローカル推論**: DeepSeek-R1 32B（メモリに余裕がある場合のみ）

---

## メモリ安全性チェックリスト

Phase 2/3 実施前に以下を確認:

- [ ] 他アプリケーション（Chrome, Slack等）を終了
- [ ] ベースラインメモリを記録（Free ≥ 10 GB）
- [ ] メモリ監視スクリプトを起動
- [ ] ロールバック手順を確認
- [ ] セーブポイント作成（重要な作業をコミット）

---

## Sources

- [DeepSeek-R1 Hardware Requirements Discussion](https://huggingface.co/deepseek-ai/DeepSeek-R1/discussions/19)
- [How Much Memory Does Deepseek R1 32b Use?](https://www.byteplus.com/en/topic/405047)
- [DeepSeek R1 RAM Requirements](https://dev.to/askyt/deepseek-r1-ram-requirements-105a)
- [Comprehensive Hardware Requirements Report](https://dev.to/ai4b/comprehensive-hardware-requirements-report-for-deepseek-r1-5269)

---

**作成日**: 2026-01-11
**作成者**: Atlas (Orchestrator)
**ステータス**: Phase 1 準備完了、Phase 2 判定待ち
