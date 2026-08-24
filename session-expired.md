# セッションが頻繁に切れるとき

`API Error: Your session has expired. Please reauthenticate.` が15〜30分おきに出る場合、原因はほぼ **同じ AWS ログインセッションを複数のプロセスが同時にリフレッシュしている** こと。AWS 側の設定変更ではない。

数時間おきに1回程度なら通常の期限切れなので、[README のセッション切れ手順](README.md#セッション切れで使えなくなったとき)で復旧する。

## 何が起きているか

`aws login` で受け取るのは、15分で切れる一時クレデンシャルとリフレッシュトークンの組。リフレッシュトークンは一度使うと新しいものに入れ替わる使い捨てで、古いトークンをもう一度提示すると AWS が盗用(replay)とみなし、そのセッション自体を失効させる。CloudTrail には `CreateOAuth2Token` の `AccessDenied`、メッセージ `Refresh token replay detected` として残る。

Claude Code は AWS CLI を経由せず自分でリフレッシュする(`aws-sdk-js`)。そのため Claude Code や VS Code のウィンドウを複数開いていると、各プロセスが `~/.aws/login/cache` の同じトークンを読んで同時に更新しようとする。先に成功した1つ以外は古いトークンを出すことになり、セッションごと落ちる。`aws login` し直した直後だけ動いて15分後にまた切れるのは、これが毎サイクル繰り返されている状態。

## 復旧手順

Claude Code と VS Code をすべて終了し、残っていないか確認する。

```sh
pgrep -fl claude
```

古いトークンキャッシュを捨てて、入り直す。

```sh
rm -rf ~/.aws/login/cache
aws login --profile smilesurvey
aws sts get-caller-identity --profile smilesurvey
```

AWS CLI を最新にする。

```sh
brew upgrade awscli
```

そのうえで **まず1セッションだけ** で使い、切れないか確認する。切れなければ同時起動が原因で確定。

> [!TIP]
> 複数同時に使いたい場合の応急処置として、`~/.claude/settings.json` に `awsAuthRefresh` を入れておくと失効時に自動で再認証が走る(ブラウザは開く)。
>
> ```json
> {
>   "awsAuthRefresh": "aws login --profile smilesurvey"
> }
> ```

`~/.aws` が iCloud Drive や Dropbox の同期対象下にある場合も同じ症状になる。古いトークンファイルが後から復元されるため。同期対象外に移す。

## キャッシュを分けて複数起動する

トークンの取り合いを避けるだけなら、プロセスごとにキャッシュの置き場所を変える手もある。`AWS_LOGIN_CACHE_DIRECTORY` でキャッシュ先を上書きできる。

```sh
export AWS_LOGIN_CACHE_DIRECTORY=~/.aws/login/cache-$$
aws login --profile smilesurvey
```

分けたディレクトリごとに `aws login` が必要になる代わり、複数ウィンドウを開いたままにできる。1セッション運用が業務上つらい場合の折衷案。

## それでも直らないとき

上をすべて試して15〜30分おきの失効が続くなら、ブラウザ認証そのものをやめる選択肢がある。[Bedrock API キーで認証する](bedrock-api-key.md)を参照。期限付きの長期キーに切り替えると `aws login` が動かなくなるため、replay は起きなくなる。MFA が効かなくなる引き換えがあるので、常用ではなく個別対応として使う。

## 切り分け(管理者向け)

CloudTrail の `CreateOAuth2Token` の成否を集計すると、端末側かアカウント側かが分かれる。

| 失敗の出方 | 原因の所在 | 対処 |
|------------|-----------|------|
| 特定のユーザーだけ | その端末 | 上の復旧手順 |
| 全ユーザー | アカウント | IAM ポリシーの変更履歴を確認 |

1日あたり数件の失敗はどのユーザーにも出る。プロセスの取り合いがたまたま起きただけで、そのたびに1回再ログインすれば済むため、正常の範囲として扱ってよい。

<details>
<summary>集計スクリプト</summary>

```python
import json, subprocess, collections

tok, ev = None, []
while True:
    cmd = ["aws", "cloudtrail", "lookup-events",
           "--lookup-attributes", "AttributeKey=EventName,AttributeValue=CreateOAuth2Token",
           "--start-time", "2026-08-01T00:00:00Z",
           "--profile", "smilesurvey", "--region", "ap-northeast-1",
           "--max-results", "50", "--output", "json"]
    if tok:
        cmd += ["--next-token", tok]
    r = json.loads(subprocess.run(cmd, capture_output=True, text=True).stdout or "{}")
    ev += [json.loads(e["CloudTrailEvent"]) for e in r.get("Events", [])]
    tok = r.get("NextToken")
    if not tok:
        break

c = collections.Counter()
for e in ev:
    user = (e.get("userIdentity") or {}).get("userName", "?")
    c[(e["eventTime"][:10], user, e.get("errorCode") or "OK")] += 1
for k in sorted(c):
    print(k, c[k])
```

`userAgent` も見ると発生元が分かる。`aws-cli/...` は `aws login` 本体、`aws-sdk-js/...` は Claude Code のリフレッシュ。

</details>

IAM ポリシーの変更履歴は次で確認する。`CreateDate` が症状の出た日より古ければ、アカウント側は無関係。

```sh
aws iam list-policy-versions --profile smilesurvey \
  --policy-arn arn:aws:iam::809411919297:policy/bedrock-users-group-invoke-policy
aws iam list-policy-versions --profile smilesurvey \
  --policy-arn arn:aws:iam::809411919297:policy/SelfManagedMfaPolicy
```

## 事例: 2026-08-07 からの `ing_uzawa`

「先週の金曜からセッションがよく切れる」という申告を CloudTrail で追った結果。

| 日付 | 成功 | 失敗 |
|------|-----:|-----:|
| 8/3 - 8/5 | 105 | 0 |
| 8/6 | 20 | 1 |
| 8/7 | 2 | 51 |
| 8/10 | 4 | 35 |

失敗はすべて `Refresh token replay detected` だった。同じ時刻(8/10 15:05:16)に2件の失敗が並んでいるので、少なくとも2プロセスが同時にリフレッシュしている。

他のメンバーは同じ期間で正常だった(kamiya は92回中0失敗、garcia は63回中0失敗、mitoma は148回中6失敗)。IAM ポリシーも7月21日以降変更がなく、アカウント側は無関係と判断した。

8月6日までのイベントは全て `darwin 25.5.0`、8月7日以降は全て `darwin 25.6.0` で、macOS のアップデート時期と一致する。ただし同じ `25.6.0` の kamiya は正常なので、OS そのものが原因ではない。再起動でウィンドウが複数復元されて同時起動数が増えた、あたりが引き金だろう。

その後、この件は [Bedrock API キー](bedrock-api-key.md)に切り替えて対応した(2026-08-24)。
