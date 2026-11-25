# 生成 LLM 接続層 API 仕様書（段0 契約）

本仕様書は、自律運用部品 `sigil-vigil-ops`（製品名 Mamoru）などの運用エージェントが、障害診断・原因特定・ランブック選定を行うために呼び出す「生成 LLM 接続層」の入出力形式、検証規則、記録形式、および縮退動作を厳密に固定する形式契約である。

## 1. 目的と設計原則

1. **裏側実体（推論ランタイム・モデル）の完全隠蔽**
   - 呼び出し側（`sigil-vigil-ops` 等）は本仕様に定められた構造化データのみを扱い、裏側の実体（オープンウェイトモデル、llama.cpp、vLLM、Ollama、あるいは将来の Sigil ネイティブ推論基盤）に依存しない。
   - バックエンドの差し替え・更新・チューニングが呼び出し側コードの改修なしに完結することを保証する。
2. **Fail-Safe（安全側に倒れる）原則**
   - 生成 LLM の出力は非決定的であり、ハルシネーションの可能性を排除できない。
   - 形式検証の失敗、確信度の不足、または応答異常時は**絶対に自動実行を行わず**、人間へのエスカレーションと監査ログ記録のみを行う。
3. **監査可能性（Auditability）**
   - 「誰が（どのランタイム・モデル・重み版で）」「いつ」「何の根拠で」判断したかを完全追跡可能にする。

---

## 2. 入力形式仕様（Request Specification）

診断要求時に生成 LLM 接続層へ送信する構造化データの定義。

### 2.1 データ構造（スキーマ）

```json
{
  "incident_id": "string (UUIDv4)",
  "timestamp": "string (ISO 8601 UTC)",
  "detection_event": {
    "source": "string",
    "severity": "critical | high | medium | low",
    "summary": "string",
    "detected_at": "string (ISO 8601 UTC)",
    "attributes": {
      "key": "string"
    }
  },
  "telemetry_summary": {
    "metrics": [
      {
        "name": "string",
        "current_value": "number",
        "threshold_value": "number",
        "unit": "string",
        "trend": "increasing | decreasing | fluctuating | constant"
      }
    ],
    "log_extracts": [
      {
        "timestamp": "string (ISO 8601 UTC)",
        "level": "FATAL | ERROR | WARN | INFO",
        "message": "string",
        "component": "string"
      }
    ]
  },
  "runbook_candidates": [
    {
      "runbook_id": "string",
      "title": "string",
      "description": "string",
      "parameters_schema": {
        "type": "object",
        "properties": {}
      },
      "risk_level": "safe | moderate | destructive"
    }
  ],
  "remediation_history": [
    {
      "incident_id": "string",
      "applied_runbook_id": "string",
      "outcome": "success | failure | partial",
      "executed_at": "string (ISO 8601 UTC)",
      "notes": "string"
    }
  ],
  "context_metadata": {
    "environment": "string",
    "cluster_id": "string",
    "service_mesh": "string"
  }
}
```

### 2.2 フィールド一覧・型・制約

| フィールド名 | 型 | 必須/任意 | 制約・バリデーション規則 | 説明 |
|---|---|---|---|---|
| `incident_id` | string | **必須** | RFC 4122 準拠の UUID 形式。空文字不可。 | インシデント固有の一意識別子。 |
| `timestamp` | string | **必須** | RFC 3339 / ISO 8601 UTC 形式（例: `2026-11-25T08:00:00Z`）。 | 診断リクエスト発行時刻。 |
| `detection_event` | object | **必須** | 下記のサブフィールドを満たすこと。 | 検知された障害イベントの詳細情報。 |
| `detection_event.source` | string | **必須** | 1〜64 文字。英数・ハイフン・ドットのみ（例: `sigil-argus-observe`）。 | イベント検知元の部品識別子。 |
| `detection_event.severity` | string (enum) | **必須** | `"critical"`, `"high"`, `"medium"`, `"low"` のいずれか。 | 障害の重大度。 |
| `detection_event.summary` | string | **必須** | 1〜500 文字。空文字不可。 | 障害概要テキスト。 |
| `detection_event.detected_at` | string | **必須** | ISO 8601 UTC 形式。 | 最初に異常が検知された時刻。 |
| `detection_event.attributes` | object | 任意 | キー・値ともに文字列。最大 50 キー。 | 追加のメタデータ（ホスト名、コンテナID等）。 |
| `telemetry_summary` | object | **必須** | メトリクスまたはログのいずれか少なくとも 1 つ以上を含むこと。 | テレメトリの要約。 |
| `telemetry_summary.metrics` | array | **必須** | 0〜50 要素。 | 異常に関連するメトリクスのリスト。 |
| `telemetry_summary.metrics[].name` | string | **必須** | 1〜128 文字。 | メトリクス名（例: `cpu_utilization_ratio`）。 |
| `telemetry_summary.metrics[].current_value` | number | **必須** | 有限数（NaN, Infinity 不可）。 | 測定された現在値。 |
| `telemetry_summary.metrics[].threshold_value` | number | **必須** | 有限数（NaN, Infinity 不可）。 | 警告または異常閾値。 |
| `telemetry_summary.metrics[].unit` | string | **必須** | 1〜32 文字（例: `"percent"`, `"bytes"`, `"ms"`）。 | 単位。 |
| `telemetry_summary.metrics[].trend` | string (enum) | **必須** | `"increasing"`, `"decreasing"`, `"fluctuating"`, `"constant"`。 | 直近の傾向。 |
| `telemetry_summary.log_extracts` | array | **必須** | 0〜50 要素。各メッセージ最大 1000 文字。 | 関連する重要ログ抜粋。 |
| `telemetry_summary.log_extracts[].timestamp` | string | **必須** | ISO 8601 UTC 形式。 | ログ出力時刻。 |
| `telemetry_summary.log_extracts[].level` | string (enum) | **必須** | `"FATAL"`, `"ERROR"`, `"WARN"`, `"INFO"`。 | ログレベル。 |
| `telemetry_summary.log_extracts[].message` | string | **必須** | 1〜1000 文字。 | ログ本文。 |
| `telemetry_summary.log_extracts[].component` | string | **必須** | 1〜64 文字。 | 出力元コンポーネント。 |
| `runbook_candidates` | array | **必須** | 1〜20 要素。空配列不可。重複する `runbook_id` 不可。 | 実行可能な復旧手順（ランブック）候補。 |
| `runbook_candidates[].runbook_id` | string | **必須** | 1〜64 文字。英数字・ハイフン・アンダースコアのみ。 | ランブック固有識別子。 |
| `runbook_candidates[].title` | string | **必須** | 1〜128 文字。 | ランブック表示名。 |
| `runbook_candidates[].description` | string | **必須** | 1〜1000 文字。 | 手順の詳細説明。 |
| `runbook_candidates[].parameters_schema` | object | 任意 | 有効な JSON Schema オブジェクト。 | 必要なパラメータ定義。 |
| `runbook_candidates[].risk_level` | string (enum) | **必須** | `"safe"`, `"moderate"`, `"destructive"`。 | 破壊度・リスク水準。 |
| `remediation_history` | array | 任意 | 0〜10 要素。 | 過去の対応履歴。 |
| `context_metadata` | object | 任意 | 最大 20 キー。環境名等の追加コンテキスト。 | システム実行環境情報。 |

---

## 3. 出力形式仕様（Response Specification）

生成 LLM 接続層が呼び出し元に返す構造化判断結果の定義。

### 3.1 データ構造（スキーマ）

```json
{
  "diagnosis_id": "string (UUIDv4)",
  "incident_id": "string (UUIDv4)",
  "root_cause_analysis": {
    "primary_cause": "string",
    "alternative_causes": [
      "string"
    ]
  },
  "selected_remediation": {
    "action_type": "execute_runbook | escalate_to_human | no_action_needed",
    "runbook_id": "string | null",
    "runbook_parameters": {
      "key": "any"
    }
  },
  "rationale": "string",
  "confidence": "number (0.00 to 1.00)"
}
```

### 3.2 フィールド一覧・型・制約

| フィールド名 | 型 | 必須/任意 | 制約・バリデーション規則 | 説明 |
|---|---|---|---|---|
| `diagnosis_id` | string | **必須** | UUIDv4 形式。空文字不可。 | 本診断トランザクション固有の ID。 |
| `incident_id` | string | **必須** | リクエストの `incident_id` と完全一致すること。 | 対象インシデント ID。 |
| `root_cause_analysis` | object | **必須** | 下記のサブフィールドを満たすこと。 | 原因分析結果。 |
| `root_cause_analysis.primary_cause` | string | **必須** | 1〜1000 文字。空文字不可。 | 最も確度が高いと判定された根本原因仮説。 |
| `root_cause_analysis.alternative_causes` | array | 任意 | 0〜5 要素。各要素最大 500 文字。 | 併せて考えられる他の原因候補。 |
| `selected_remediation` | object | **必須** | 下記の制約を満たすこと。 | 選択された対処アクション。 |
| `selected_remediation.action_type` | string (enum) | **必須** | `"execute_runbook"`, `"escalate_to_human"`, `"no_action_needed"` のいずれか。 | 実行アクションの区分。 |
| `selected_remediation.runbook_id` | string \| null | 条件付き必須 | 1. `action_type == "execute_runbook"` の場合: 入力 `runbook_candidates` に含まれる `runbook_id` のいずれかと**完全一致**すること（必須）。<br>2. それ以外の場合: `null` であること。 | 選択したランブックの ID。 |
| `selected_remediation.runbook_parameters` | object | 任意 | 指定されたランブックの `parameters_schema` に適合すること。 | ランブック実行用引数。 |
| `rationale` | string | **必須** | 10〜2000 文字。空文字不可。 | この判断・ランブック選定に至った論理的根拠・証拠。 |
| `confidence` | number (float) | **必須** | `0.0 <= confidence <= 1.0` の有限数（小数点以下最大 4 桁推奨）。 | 判断の確信度。 |

---

## 4. 形式検証規則（Schema Validation Rules）

生成 LLM 接続層から返された出力は、呼び出し元および接続層内部で厳格に検証される。
**出力が以下のいずれかの条件に該当した場合は形式検証失敗（バリデーションエラー）とし、その出力を絶対に判断として採用してはならない。**

### 4.1 検証で弾く条件の網羅的列挙

1. **構文不正**
   - JSON 形式としてパースできない（構文エラー、括弧の不整合、途中で切断された不完全な文字列等）。
2. **必須フィールドの欠落**
   - `diagnosis_id`, `incident_id`, `root_cause_analysis`, `selected_remediation`, `rationale`, `confidence` のいずれかが存在しない、またはキー名が誤っている。
3. **データ型の不一致**
   - 数値型に文字列が入っている（例: `"confidence": "0.85"`）。
   - 配列であるべき箇所にオブジェクトや文字列が入っている。
   - `confidence` が `NaN`, `Infinity`, `-Infinity` などの不正な浮動小数点数である。
4. **値の範囲制約違反**
   - `confidence` が `0.0` 未満、または `1.0` を超えている。
   - `rationale` または `primary_cause` が空文字（文字数 0）、または定義された最大文字数を超過している。
5. **未知のランブック指定（ハルシネーションの遮断）**
   - `action_type` が `"execute_runbook"` であるにもかかわらず、指定された `runbook_id` がリクエストの `runbook_candidates` 配列内に存在しない。
6. **アクション種別とランブック指定の不整合**
   - `action_type` が `"escalate_to_human"` または `"no_action_needed"` であるにもかかわらず、`runbook_id` に値が設定されている。
   - `action_type` が `"execute_runbook"` であるにもかかわらず、`runbook_id` が `null` または空文字である。
7. **ランブック引数のスキーマ不適合**
   - `selected_remediation.runbook_parameters` が、当該ランブックの `parameters_schema` 定義（JSON Schema）に適合しない。
8. **インシデント整合性違反**
   - レスポンスの `incident_id` が、リクエストの `incident_id` と一致しない。

---

## 5. 確信の度合い（Confidence）の扱い

1. **判断採用閾値（Confidence Threshold）の定義**
   - システム全体またはインシデント種別ごとに、判断採用閾値 `CONFIDENCE_THRESHOLD` を設定する。
   - 標準デフォルト値: **`0.70`**（70%）。
   - 高リスク操作（`risk_level == "destructive"`）のランブックに対する閾値: **`0.90`**（90%）。
2. **閾値未満の場合の厳格な扱い**
   - **`confidence < CONFIDENCE_THRESHOLD` の場合、自動実行判断は無条件で却下（不採用）される。**
   - LLM が `action_type: "execute_runbook"` を提案していた場合でも、システム側で強制的にアクション種別を `"escalate_to_human"` に上書きし、自動実行を抑止する。
   - 確信度不足による却下の事実は、判断記録に明記される。

---

## 6. 判断記録の形式（Decision Audit Record）

自律運用におけるすべての判断は、事後検証・責任追跡・モデル再学習（振る舞い蒸留）のために完全な形で永続化される。

### 6.1 記録スキーマ

```json
{
  "record_id": "string (UUIDv4)",
  "incident_id": "string (UUIDv4)",
  "timestamp": "string (ISO 8601 UTC)",
  "decision_source": {
    "provider": "string",
    "runtime_version": "string",
    "instance_id": "string",
    "host_environment": "string"
  },
  "model_metadata": {
    "model_name": "string",
    "weights_version": "string",
    "weights_sha256": "string",
    "quantization": "string",
    "context_window_tokens": 8192
  },
  "request_digest": "string (SHA256)",
  "raw_response_text": "string",
  "parsed_response": "object | null",
  "validation_result": {
    "passed": true,
    "errors": []
  },
  "confidence": 0.88,
  "threshold_applied": 0.70,
  "decision_adopted": true,
  "fallback_triggered": false,
  "fallback_reason": "null | string",
  "latency_ms": 1240
}
```

### 6.2 必須記録要件

- **判断の出所（`decision_source`）**: どの実体（プロバイダ種別、ランタイム版、実行インスタンスID）が判断を出力したかを必ず記録する。
- **モデルの版（`model_metadata`）**: 使用した基盤モデル名、重みバージョン、重みファイルの SHA256 ハッシュ値、量子化形式（例: `Q4_K_M`）を記録する。
- **生テキストと解析結果**: LLM が生成した生テキスト（`raw_response_text`）と、それをパースした構造化データを両方保持する。
- **検証と採用の可否**: 形式検証の合否、適用された確信度閾値、最終的に自動判断が採用されたか否かを記録する。

---

## 7. タイムアウトと異常応答の扱い

1. **タイムアウト仕様**
   - 推論呼び出しのタイムアウト標準値: **`30 秒`**。
   - タイムアウト発生時は、接続層が即座に処理を中断（プロセスシグナル送信または HTTP 接続切断）し、タイムアウト例外として処理する。
2. **異常応答の分類**
   - **通信・プロセス異常**: 推論ランタイムのプロセス異常終了（OOM-killer、クラッシュ）、ソケット切断、HTTP 5xx エラー。
   - **空応答**: レスポンスボディが 0 バイト、または空白のみ。
   - **文字化け・不正エンコーディング**: UTF-8 として解釈不可能なバイト列。
3. **異常発生時の処置**
   - すべての異常応答は即座に検知され、後述の「縮退動作」をトリガーする。再試行（リトライ）は最大 1 回までとし、復帰しない場合は直ちに縮退する。

---

## 8. 縮退動作（Degraded Operation / Fallback Behavior）

安全側の原則に基づき、例外状態における振る舞いを明確に規定する。

### 8.1 縮退動作の発動条件（3大条件）

以下のいずれか 1 つでも満たされた場合、直ちに縮退動作を発動する。

1. **LLM が応答しない（無応答・タイムアウト・異常終了）**
   - 推論タイムアウト超過、ランタイムプロセスのクラッシュ、通信エラー、空応答。
2. **形式検証に通らない（検証失敗）**
   - 第 4 節に列挙されたいずれかの検証規則違反（構文エラー、必須項目欠落、型不一致、未知のランブック指定等）。
3. **確信が低い（低確信）**
   - `confidence < CONFIDENCE_THRESHOLD`。

### 8.2 縮退時の統一動作（フェイルセーフ）

縮退動作が発動した場合、システムは以下を**厳格に実行**する。

1. **判断を行わない（自動実行の完全遮断）**
   - いかなるランブックの自動実行もトリガーしない。推論結果に含まれるランブック指定は完全に無効化される。
2. **安全側に倒れる（現状維持）**
   - 自動的な設定変更や環境変更は行わず、既存の稼働状態を維持する。
3. **人間へのエスカレーション**
   - 運用当番・管理者へ即時に緊急通知（アラート）を発報する。
   - 通知には、発生したインシデントの概要、LLM が正常判断できなかった理由（無応答 / 検証エラー / 低確信）、および関連ログを含める。
4. **監査記録の永続化**
   - 発動理由（`fallback_reason`）、受信した生データ、エラー詳細を監査ログ（第 6 節）に書き込む。
