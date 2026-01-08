# DeepSeek-R1

**DeepSeek-R1**は、DeepSeekが開発した推論特化型のオープンソースAIモデルです。高度な論理推論能力を備えながら、優れたコストパフォーマンスを実現しています。

## 概要

- **パラメータ数**: 14B/32B
- **特徴**: 推論特化、コスト効率的、MITライセンス
- **開発元**: DeepSeek（中国のAI企業）
- **日本語対応**: 強い（CyberAgent版推奨）
- **コンテキスト長**: 128k トークン
- **ライセンス**: MIT

## スペック比較

| 項目 | 14B版 | 32B版 |
|------|-------|-------|
| パラメータ数 | 14B | 32B |
| メモリ要件 | 16GB+ | 32GB+ |
| 推論速度 | 高速 | 中程度 |
| 性能 | バランス型 | 高精度 |

## クイックスタート

### Ollama経由のインストール

```bash
# 14B版（推奨）
ollama pull deepseek-r1:14b

# 32B版（高精度）
ollama pull deepseek-r1:32b
```

### 実行

```bash
ollama run deepseek-r1:14b
```

## 推奨用途

- **複雑な論理問題の解決**
- **数学・科学的な推論**
- **アルゴリズム設計**
- **詳細なコード生成**
- **日本語での複雑なタスク**

## 特徴

- **推論能力**: 深い思考プロセスを実装
- **コストパフォーマンス**: 大規模モデルと比較して低リソース消費
- **日本語強化**: CyberAgent版は日本語に最適化
- **オープンソース**: 研究・商用利用が可能

## 関連リンク

- [DeepSeek公式サイト](https://www.deepseek.com/)
- [HuggingFace (deepseek-ai)](https://huggingface.co/deepseek-ai)
- [Ollama - DeepSeek](https://ollama.ai/library/deepseek-r1)

---

生成日: 2026-01-09
