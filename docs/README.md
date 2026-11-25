# docs

`sigil-anima-llm` の設計・仕様文書。

## 文書一覧

- **[generative-llm-contract.md](generative-llm-contract.md)**
  - 生成 LLM 接続層 API 仕様書（段0 契約）。
  - `sigil-vigil-ops` 等の運用部品から呼び出される診断・原因特定・ランブック選定の入出力スキーマ、形式検証規則、確信度閾値ポリシー、判断記録形式、および縮退動作（フェイルセーフ原則）を定義。
- **[classification-contract.md](classification-contract.md)**
  - 分類の契約（System One 形式）仕様書（段0 契約）。
  - 型つきの問いに確率と confidence で答える分類契約。3 種の問い型（Noul / Choice / Score）のリクエスト・レスポンス形式、既存 `POST /v1/systemone`（typesafe-ai）との対応関係、暫定バックエンドから将来の自前実装への差し替え方針を定義。
