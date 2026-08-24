# Bedrock 利用状況を IAM ユーザー別に集計する

Claude Code (Bedrock) の利用量と概算金額を、IAM ユーザーごとに AWS CLI だけで集計する手順(管理者向け)。

## 仕組み

Bedrock のモデル呼び出しログ (Model invocation logging) を有効にしてあるため、Claude の全呼び出しが CloudWatch Logs のロググループ `/aws/bedrock/invocation-logs` (ap-northeast-1) に記録される。1 レコードが 1 呼び出しに対応し、これを CloudWatch Logs Insights で集計する。

| フィールド | 内容 |
|-----------|------|
| `identity.arn` | 呼び出し元 (IAM ユーザー or assumed-role) |
| `modelId` | 使用モデル (inference profile の ARN) |
| `input.inputTokenCount` | 入力トークン |
| `output.outputTokenCount` | 出力トークン |
| `input.cacheReadInputTokenCount` | プロンプトキャッシュ読み取りトークン |
| `input.cacheWriteInputTokenCount` | プロンプトキャッシュ書き込みトークン |

> [!NOTE]
> ログ設定は本文テキストの保存を無効化してある (`textDataDeliveryEnabled: false`)。プロンプト内容は記録されず、メタデータとトークン数のみ。

## 1. ログ設定を確認

```sh
aws bedrock get-model-invocation-logging-configuration --region ap-northeast-1 --profile smilesurvey
```

`loggingConfig.cloudWatchConfig.logGroupName` に `/aws/bedrock/invocation-logs` が出れば有効。

## 2. IAM × モデル別に集計

Logs Insights のクエリを CLI から実行する。期間は `--start-time` / `--end-time` (epoch 秒) で調整する。

```sh
# 例: 今日 0:00 〜 現在 (macOS の date)
start=$(date -j -f "%Y-%m-%dT%H:%M:%S" "$(date +%Y-%m-%d)T00:00:00" +%s)
end=$(date +%s)

qid=$(aws logs start-query --region ap-northeast-1 --profile smilesurvey \
  --log-group-name /aws/bedrock/invocation-logs \
  --start-time $start --end-time $end \
  --query-string 'stats count(*) as calls,
    sum(input.inputTokenCount) as inTok,
    sum(output.outputTokenCount) as outTok,
    sum(input.cacheReadInputTokenCount) as cacheRead,
    sum(input.cacheWriteInputTokenCount) as cacheWrite
    by identity.arn as arn, modelId' \
  --output text --query queryId)

# 完了までポーリングしてから結果取得
while [ "$(aws logs get-query-results --region ap-northeast-1 --profile smilesurvey \
      --query-id $qid --output text --query status)" != "Complete" ]; do sleep 3; done

aws logs get-query-results --region ap-northeast-1 --profile smilesurvey \
  --query-id $qid --output json > /tmp/bedrock_cost_raw.json
```

## 3. 金額に換算

単価を掛けて IAM 別に合算する。計算式は次のとおり。

| トークン種別 | 単価 |
|-------------|------|
| 入力 | モデルごとの入力単価 |
| 出力 | モデルごとの出力単価 |
| キャッシュ読み取り | 入力単価 × 0.1 |
| キャッシュ書き込み | 入力単価 × 1.25 |

<details>
<summary>換算スクリプト (python3 ヒアドキュメント)</summary>

```sh
python3 <<'EOF'
import json, re
from collections import defaultdict

data = json.load(open('/tmp/bedrock_cost_raw.json'))

# $/1M トークン: (input, output)。値上げ/新モデルが出たら要更新
PRICES = {
    'fable-5':    (10.0, 50.0),
    'opus-5':     (5.0, 25.0),
    'opus-4-8':   (5.0, 25.0),
    'opus-4-7':   (5.0, 25.0),
    'opus-4-6':   (5.0, 25.0),
    'opus-4-5':   (5.0, 25.0),
    'sonnet-5':   (2.0, 10.0),   # イントロ価格 (2026-08-31 まで。以降は 3.0/15.0)
    'sonnet-4-6': (3.0, 15.0),
    'sonnet-4-5': (3.0, 15.0),
    'haiku-4-5':  (1.0, 5.0),
    'sonnet-4-2025': (3.0, 15.0),
    '3-5-sonnet': (3.0, 15.0),
    '3-sonnet':   (3.0, 15.0),
}
RATE = 148  # USD/JPY 概算

def price_for(model_id):
    for key, p in PRICES.items():
        if key in model_id:
            return p
    return None

by_user = defaultdict(lambda: [0, 0.0])
for r in data['results']:
    d = {f['field']: f.get('value') for f in r}
    model = d.get('modelId') or ''
    p = price_for(model)
    if p is None:
        if model:  # 空 modelId は失敗系レコード (トークン 0) なので黙って除外
            print(f"!! 単価不明: {model}")
        continue
    inp, outp = p
    arn = d.get('arn', '')
    # user/名前 はそのまま、assumed-role/ロール名/セッション名 はロール名を採用
    m = re.search(r'user/(.+)$', arn) or re.search(r'assumed-role/([^/]+)', arn)
    user = m.group(1) if m else arn
    it = float(d.get('inTok') or 0); ot = float(d.get('outTok') or 0)
    cr = float(d.get('cacheRead') or 0); cw = float(d.get('cacheWrite') or 0)
    cost = (it*inp + ot*outp + cr*inp*0.1 + cw*inp*1.25) / 1e6
    by_user[user][0] += int(float(d.get('calls') or 0))
    by_user[user][1] += cost

print(f"{'IAM':32} {'calls':>6} {'USD':>8} {'JPY':>8}")
total = 0.0
for user, (calls, cost) in sorted(by_user.items(), key=lambda x: -x[1][1]):
    print(f"{user:32} {calls:6d} {cost:8.2f} {cost*RATE:8,.0f}")
    total += cost
print(f"{'合計':32} {'':6} {total:8.2f} {total*RATE:8,.0f}")
EOF
```

</details>

出力例。

```text
IAM                               calls      USD      JPY
ing_xxxx                            862   160.46   23,748
ing_yyyy                            803    77.68   11,496
合計                                       238.14   35,244
```

## 注意点

- CloudTrail ではなくこのログを使う。`lookup-events` でも呼び出し元は分かるが、スロットリングしやすくトークン数も取れない
- キャッシュトークンを忘れない。Claude Code はプロンプトキャッシュを多用するため、コストの過半はキャッシュ読み取り。`inputTokenCount` だけ見ると実態の数十分の一に見える
- Lambda 経由の利用は assumed-role 名で出る (例: `stg-ai-report`)。IAM ユーザー直はそのまま個人名
- 入力 8 トークン・出力 1 トークン程度の行が大量にあるのは正常。Claude Code 起動時のモデル疎通チェックで、コストはほぼゼロ
- 単価 (`PRICES`) は Bedrock の公表値を手動メンテ。モデル追加・改定時に更新する
- Logs Insights のクエリ結果は最大 10,000 行。IAM × モデルのグループ数が超えることは当面ない
