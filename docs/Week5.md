# 第5回 ツール利用・function calling・外部API 演習記録（week05）

- 学生ID: x24013
- 氏名: 占部颯
- 対象回: SMR2026 第5回
- 使用ツール: `city_tools.py` / `city_weather_sample.json` / `city_stats_sample.json`

> 注意: 本記録で扱うデータはすべて教材用の疑似データであり、実際の気象・防災判断には使用しない。実際の災害時は自治体・気象庁等の公式情報を確認する。APIキー・個人情報は記載しない。

---

## 1. 演習1：API / ローカルJSON 確認

| 項目 | 内容 |
|---|---|
| Python環境確認 | OK |
| 使用した方式 | ローカルJSON |
| 使用ファイル | `run_demo_local.py`, `city_tools.py` |
| 入力した都市名 | Tokyo |
| 取得できた項目 | date, weather, temperature_c, rainfall_mm, warning_level ほか |
| APIキー使用 | いいえ |

### 実行の経過

最初の実行は失敗した。原因は実行ディレクトリの指定が異なっており、データファイルが見つからなかったこと。

```
{'ok': False, 'error_type': 'FileNotFoundError',
 'message': "[Errno 2] No such file or directory: '.../data/city_weather_sample.json'"}
```

ディレクトリを修正して再実行したところ成功し、構造化データと説明文が得られた。

```
{'ok': True, 'tool_name': 'get_city_weather', 'city': 'Tokyo',
 'date': '2026-05-01', 'weather': '晴れ（教材用）', 'temperature_c': 23.4,
 'rainfall_mm': 0.0, 'warning_level': 'none',
 'source': 'SMR2026 dummy dataset created for class exercise',
 'updated_at': '2026-05-01', 'unit': 'temperature_c: Celsius, rainfall_mm: mm',
 'disclaimer': '教材用の疑似気象データ。実際の気象・防災判断には使わない。'}
```

説明文:

> Tokyo の教材用気象データでは、2026-05-01 の状況は 晴れ（教材用）、気温は 23.4 ℃、降水量は 0.0 mm です。出典は SMR2026 dummy dataset、更新日は 2026-05-01 です。これは教材用データであり、実際の防災判断には自治体や気象庁などの公式情報を確認してください。

### 学んだこと

エラーが起きたとき、どこまで処理が通っているか分からなかった。手順ごとに print を入れてエラー箇所を特定できるようにするとよい。また、指定していたディレクトリが実際の場所と全く異なっていたため、実行前に作業ディレクトリを確認する習慣をつけたい。

次に確認したいこと: 外部APIとの連携のやり方。

---

## 2. 演習2：都市データ取得

| 項目 | 内容 |
|---|---|
| 関数名 | `get_city_weather` |
| 入力 | `city="Tokyo"` |
| データ種別 | 気象 |
| 取得方法 | ローカルJSON |
| 取得できた値 | `temperature_c: 23.4` |
| 単位 | ℃ |
| 出典 / ファイル名 | `city_weather_sample.json` |
| 取得日 | 2026-05-01 |
| 更新日 | 2026-05-01 |

### 役割分担

- LLMに任せた処理: 取得済みデータを自然言語の説明文に言い換える。
- プログラムで確定した処理: 都市名の照合と、日付に応じた付随情報（天気・気温・降水量・出典・更新日・単位）を確定して返す。

出典・取得日・更新日・単位（`source`, `retrieved_at`, `updated_at`, `unit`）を戻り値に含めることを確認した。

### 失敗例・改善案

- 失敗した入力例: 特になし（正常系のみ確認）。
- 改善案: エラー対応の表記を追加し、異常系でも安全に返せるようにする。→ 演習4で実装。

---

## 3. 演習3：ツール仕様定義（function calling）

| 項目 | 内容 |
|---|---|
| ツール名 | `get_city_weather` |
| 目的 | 都市名を受け取り、教材用の疑似気象情報を返す（実際の防災判断には使わない） |
| 入力引数 | `city`（string） |
| 必須引数 | `city` |
| 任意引数 | `data_path`（内部用・省略時は標準JSONを使用。LLM向けスキーマには含めない） |
| 入力スキーマ形式 | JSON Schema |
| 戻り値の項目 | city, date, weather, temperature_c, rainfall_mm, warning_level, source, updated_at, unit, disclaimer |
| エラー時の戻り値 | ok=false, error_type, message |
| LLMに任せる処理 | 意図理解・説明文生成 |
| プログラムで確定する処理 | 入力検証、JSON読み込み、単位保持 |
| 入力検証方法 | 空文字不可 |
| 正常系テスト | Tokyo |
| 異常系テスト | UnknownCity |
| ログに残す項目 | timestamp, tool_name, input, status |
| 安全上の注意 | 教材用データ。実際の防災判断には使わない |

### 入力スキーマ（JSON Schema）

```json
{
  "type": "function",
  "name": "get_city_weather",
  "description": "都市名を受け取り、教材用の疑似気象情報を返す。実際の防災判断には使わない。",
  "parameters": {
    "type": "object",
    "properties": {
      "city": { "type": "string", "description": "都市名。例: Tokyo, Osaka, 東京, 大阪" }
    },
    "required": ["city"],
    "additionalProperties": false
  },
  "strict": true
}
```

---

## 4. 演習4：ペア改造とテスト

- ペア相手: 竹内新
- 改造したツール: `get_city_weather`

### 改造前の問題点

空文字・空白のみの入力時、`error_type` が汎用の `ValueError` になっていた。この名前では「都市名が空だった」のか「別の原因」なのか区別できず、未対応都市エラーとも粒度がそろわなかった。

### 改造内容

空文字・空白のみの入力を専用の `error_type: "EMPTY_CITY"` として返すよう分岐を追加した。専用メッセージと実行ログも記録する。

```python
# 改造(A案): 空文字/空白のみの入力を専用エラーとして区別する
if city is None or not str(city).strip():
    result = {"ok": False, "error_type": "EMPTY_CITY",
              "message": "都市名が空です。都市名を入力してください。"}
    log_event(tool_name, {"city": str(city)}, "error", "empty city", "EMPTY_CITY")
    return result
```

### テスト結果

| 区分 | 入力 | 期待結果 | 実行結果 | 合否 |
|---|---|---|---|---|
| 正常系1 | Tokyo | ok=True, city=Tokyo | ok=True, city=Tokyo, weather=晴れ（教材用） | 合格 |
| 正常系2 | 大阪 | ok=True, city=Osaka | ok=True, city=Osaka, weather=くもり（教材用） | 合格 |
| 異常系1 | Atlantis | ok=False, error_type=UNSUPPORTED_CITY | ok=False, error_type=UNSUPPORTED_CITY | 合格 |
| 異常系2 | 空文字（"   "） | ok=False, error_type=EMPTY_CITY | ok=False, error_type=EMPTY_CITY | 合格 |

- 実行ログ: あり（`logs/tool_execution_log.jsonl` に4件、JSON Lines形式で記録）。
- デモで説明する点: 何をプログラム側で確定したか、空文字を専用エラーで安全に返す点、出典・更新日・単位を戻り値に含める点。
- 残った課題: `warning_level` の日本語表示、`date`（日付）の妥当性検証は未対応。

---

## 5. 第6回 RAG に向けた資料候補メモ

- API（構造化データ）だけでは、自治体の防災計画・避難所一覧などの説明文書はカバーできない。これらは第6回のRAGで文書検索として扱う候補。
- 本演習で記録した `source` / `updated_at` / `unit` は、RAGの根拠提示・評価にもそのまま使える。
- 必要になりそうな外部知識: 自治体の防災計画・避難所一覧などの公式文書。

---

## 6. 安全上の注意（提出前チェック）

- [x] APIキーを記載していない
- [x] 個人情報・非公開情報を記載していない
- [x] `.env` をGitHubに上げていない
- [x] 教材用データを実際の災害判断に使わない旨を明記した
