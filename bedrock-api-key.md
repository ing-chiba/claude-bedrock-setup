# Bedrock API キーで認証する

`aws login` を使わず、期限付きの長期キーで Bedrock を呼ぶ方法。ブラウザ認証が一切走らないため、リフレッシュトークンの競合によるセッション切れが原理的に起きない。

[セッションが頻繁に切れるとき](session-expired.md)の対処を試しても直らない場合の代替手段として使う。全員に配るものではない。

## aws login との違い

| | `aws login` | Bedrock API キー |
|---|---|---|
| 再認証 | 最大12時間ごとにブラウザ | 期限まで不要 |
| MFA | 毎回強制される | 効かない |
| キーで触れる範囲 | そのユーザーの全権限 | Bedrock の呼び出しだけ |
| 複数プロセス同時起動 | トークンが競合して落ちる | 影響しない |
| 失効のさせ方 | セッションを切る | キー単体を削除・無効化 |

裏側は IAM の service-specific credential で、用途が Bedrock に固定されている。キーが漏れても S3 や IAM は触れない。そのぶん MFA は乗らないので、その1点を許容できるかで採否が決まる。

## キーの発行を千葉に依頼

千葉に依頼してキーを発行してもらい、`ABSK` で始まる120文字の文字列を受け取る。

<details>
<summary>キーの発行</summary>

対象ユーザーを指定してキーを作る。`--credential-age-days` で有効日数を決める。

```sh
aws iam create-service-specific-credential \
  --user-name ing_xxxx \
  --service-name bedrock.amazonaws.com \
  --credential-age-days 90 \
  --profile smilesurvey
```

レスポンスの `ServiceCredentialSecret` が本人に渡す値。

> [!IMPORTANT]
> AWS のドキュメントには `ServiceApiKeyValue` が返ると書かれているが、実際に返るフィールドは `ServiceCredentialSecret`。ドキュメントの記述と食い違う。
>
> この値を後から取得する手段はない。控え忘れた場合は `reset-service-specific-credential` で作り直す。

`ServiceSpecificCredentialId`(`ACCA` で始まる)も記録しておく。無効化と削除にはこちらを使う。

</details>

## 設定(本人)

`~/.claude/settings.json` を書き換える。

変更前。

```json
{
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "1",
    "AWS_REGION": "ap-northeast-1",
    "AWS_PROFILE": "smilesurvey"
  },
  "awsAuthRefresh": "aws login --profile smilesurvey"
}
```

変更後。

```json
{
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "1",
    "AWS_REGION": "ap-northeast-1",
    "AWS_BEARER_TOKEN_BEDROCK": "受け取ったキー"
  }
}
```

書き換えたら Claude Code を完全に終了して起動し直す。`/status` の API provider が `Amazon Bedrock`、AWS region が `ap-northeast-1` になっていれば移行できている。

## 期限と更新

発行時に指定した日数で失効する。切れると Claude Code から Bedrock が呼べなくなるので、期限前に差し替える。

```sh
aws iam reset-service-specific-credential \
  --user-name ing_xxxx \
  --service-specific-credential-id ACCAXXXXXXXXXXXXXXXXX \
  --profile smilesurvey
```

同じ ID のまま値だけが変わる。新しい `ServiceCredentialSecret` を本人に渡し、`settings.json` を更新してもらう。

発行済みキーの一覧と期限は次で確認する。

```sh
aws iam list-service-specific-credentials \
  --user-name ing_xxxx \
  --service-name bedrock.amazonaws.com \
  --profile smilesurvey
```

## 動作確認

キーが有効かどうかは curl で確かめられる。

```sh
curl -s -X POST \
  "https://bedrock-runtime.ap-northeast-1.amazonaws.com/model/jp.anthropic.claude-sonnet-4-5-20250929-v1:0/invoke" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $AWS_BEARER_TOKEN_BEDROCK" \
  -d '{"anthropic_version":"bedrock-2023-05-31","max_tokens":16,"messages":[{"role":"user","content":"say OK"}]}'
```

応答が返れば通っている。

```json
{"content":[{"type":"text","text":"OK"}],"stop_reason":"end_turn"}
```

## 参考

- [Amazon Bedrock API keys](https://docs.aws.amazon.com/bedrock/latest/userguide/api-keys.html)
- [Claude Code on Amazon Bedrock](https://code.claude.com/docs/ja/amazon-bedrock)
- [セッションが頻繁に切れるとき](session-expired.md)
