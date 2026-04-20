# Testnet Multisig Manual Test Guide

この手順書は、`shoestring` を使って Symbol testnet (`sai`) 上の既存マルチシグアカウントを対象に、`setup` からリンク系トランザクション生成、オフライン署名、ネットワークアナウンスまでを手動で確認するためのものです。

対象範囲は以下です。

- `init` による設定ファイル生成
- マルチシグ前提の `transaction.signerPublicKey` 設定
- `min-cosignatures-count` による必要署名数の反映
- `setup` による `linking_transaction.dat` 生成
- `signer` による複数署名
- `announce-transaction` による testnet 送信

この手順はテストネット専用です。`shoestring` は `--ca-key-path` に渡した PEM を証明書生成にも署名にも使います。マルチシグ検証では、証明書用キーとコサイナー鍵が同一になります。本番運用の前提にする設計ではありません。

## 前提条件

- 対象ネットワークは `sai`。
- 既存のマルチシグアカウントが testnet 上に作成済み。
- マルチシグは単一階層。多段マルチシグだと `min-cosignatures-count` は自動検出できません。
- 署名に使う各コサイナーの秘密鍵を PEM で用意できる。
- 手数料支払い用の XYM を十分に保有している。
- `imports.harvester` と `imports.voter` は空にしておく。既存キーを import すると、新規リンク確認にならない。

## 事前に理解しておくべきこと

- `setup` が生成するのは `linking_transaction.dat` です。
- `transaction.signerPublicKey` には、`--ca-key-path` の公開鍵ではなく、リンク対象にしたいマルチシグアカウントの公開鍵を設定します。
- `signer` は最初の署名時に aggregate 外側署名者を実際のコサイナー公開鍵へ置き換えます。内側トランザクションの署名者はマルチシグ公開鍵のままです。
- `minCosignaturesCount` が `2` 以上なら aggregate bonded になり、初回署名時に `linking_transaction.hash_lock.dat` が生成されます。
- 既存リンクがある場合、生成される aggregate には `unlink` と `link` が両方入ります。新規リンクだけとは限りません。

## 例で使うパス

以下では `product/tools/shoestring` 配下で作業します。

```sh
cd PATH/TO/product/tools/shoestring
```

作業用ディレクトリ例:

```sh
mkdir -p work
```

以降の例では次を使います。

- 設定ファイル: `work/sai.multisig.ini`
- overrides: `work/overrides.ini`
- ノード出力先: `work/node`
- 署名開始コサイナー鍵: `work/cosigner1.pem`
- 追加コサイナー鍵: `work/cosigner2.pem`, `work/cosigner3.pem`

## 1. 実行環境を用意する

必要に応じて仮想環境を作成し、依存関係を入れます。

```sh
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

`shoestring` 実行例はすべて `python -m shoestring` を使います。

## 2. 設定ファイルを生成する

```sh
python -m shoestring init --package sai work/sai.multisig.ini
```

生成された INI を編集します。最低限、以下を確認します。

### `[transaction]`

- `signerPublicKey = <マルチシグアカウントの公開鍵>` を追加または更新する。
- `minCosignaturesCount` はこの時点では仮値でよい。後で自動更新する。

例:

```ini
[transaction]
feeMultiplier = 200
timeoutHours = 1
minCosignaturesCount = 0
hashLockDuration = 1440
currencyMosaicId = 0x72C0212E67A08BCE
lockedFundsPerAggregate = 10000000
signerPublicKey = <MULTISIG_ACCOUNT_PUBLIC_KEY>
```

### `[node]`

- `features` に `HARVESTER` と `VOTER` が含まれていることを確認する。
- リンク対象を harvesting のみにしたいなら `HARVESTER` のみでもよい。
- voting link も見たいなら `VOTER` を含める。

例:

```ini
[node]
features = API | HARVESTER | VOTER
```

## 2.5 `overrides.ini` を用意する

`setup` は `config-node.properties` の `localnode.host` を必ず解決しにいきます。未設定のままだと内部のプレースホルダ値 `read or ask 1` が残り、以下のエラーになります。

```text
RuntimeError: could not resolve address for host: read or ask 1
```

これは実装どおりの失敗です。`--overrides` を渡して `node.localnode.host` と `friendlyName` を埋めろ。

例:

```ini
[node.localnode]
host = your-node.example.com
friendlyName = testnet-multisig-node
```

ローカル検証で名前解決できるなら `localhost` でも通ります。

```ini
[node.localnode]
host = localhost
friendlyName = testnet-multisig-node
```

確認条件:

- `host` は実際に名前解決できるホスト名にする。
- `apiHttps = true` の場合、IP アドレスは不可。ホスト名が必要。
- `friendlyName` も埋める。未設定のままプレースホルダを残すな。

## 3. コサイナー PEM を用意する

既存秘密鍵を PEM 化するなら `pemtool` を使います。新規作成でも構いませんが、testnet 上の実在コサイナーでないとマルチシグ署名に使えません。

```sh
python -m shoestring pemtool --output work/cosigner1.pem --input /path/to/cosigner1-private-key.txt
python -m shoestring pemtool --output work/cosigner2.pem --input /path/to/cosigner2-private-key.txt
python -m shoestring pemtool --output work/cosigner3.pem --input /path/to/cosigner3-private-key.txt
```

暗号化 PEM を使う場合:

- `pemview` は `--ask-pass` を付ける。
- `setup`、`signer`、`min-cosignatures-count` は INI の `[node]` に `caPassword` を設定する。

例:

```ini
[node]
caPassword = pass:YOUR_PEM_PASSWORD
```

`file:` 指定も使えます。

例:

```sh
printf '%s' 'YOUR_PEM_PASSWORD' > work/cosigner.pass
chmod 600 work/cosigner.pass
```

```ini
[node]
caPassword = file:work/cosigner.pass
```

`caPassword` は openssl passphrase 形式です。`pemview` は `caPassword` を参照せず、暗号化 PEM の確認時は毎回 `--ask-pass` が必要です。平文を INI に置きたくないなら `file:` を使う。どちらにせよ、パスワードやパスワードファイルを git 管理下へ入れるな。

内容確認:

```sh
python -m shoestring pemview --input work/cosigner1.pem --network testnet --ask-pass
python -m shoestring pemview --input work/cosigner2.pem --network testnet --ask-pass
```

確認ポイント:

- 各 PEM のアドレスが、対象マルチシグのコサイナーと一致していること。
- `signerPublicKey` に設定した公開鍵は、PEM の公開鍵ではなくマルチシグアカウント本体の公開鍵であること。

## 4. 必要署名数を設定ファイルへ反映する

`signerPublicKey` を設定済みなら、`--ca-key-path` に渡す PEM はコサイナー鍵で構いません。検出対象はマルチシグアカウントになります。

```sh
python -m shoestring min-cosignatures-count \
  --config work/sai.multisig.ini \
  --ca-key-path work/cosigner1.pem \
  --update
```

期待結果:

- ログに対象アドレスと必要コサイン数が出る。
- `work/sai.multisig.ini` の `minCosignaturesCount` が更新される。

失敗条件:

- 多段マルチシグだと自動検出は失敗する。
- `signerPublicKey` を入れていないと、`cosigner1.pem` の通常アカウントを見にいって誤判定する。

## 5. `setup` を実行してリンクトランザクションを生成する

`--directory` は空ディレクトリを使います。既存の `userconfig/resources` があると `setup` は止まります。

```sh
mkdir -p work/node
python -m shoestring setup \
  --config work/sai.multisig.ini \
  --package sai \
  --directory work/node \
  --overrides work/overrides.ini \
  --security default \
  --ca-key-path work/cosigner1.pem
```

確認ポイント:

- `work/node/linking_transaction.dat` が生成される。
- `work/node/keys/remote.pem` が生成される。
- `work/node/keys/vrf.pem` が生成される。
- `VOTER` を有効にした場合は `work/node/keys/voting/private_key_tree1.dat` が生成される。

補足:

- `HARVESTER` を有効にしていれば `Account Key Link` と `VRF Key Link` が候補になる。
- `VOTER` を有効にしていれば `Voting Key Link` が候補になる。
- 既存リンクがあると `unlink` が先に入る。

## 6. 再生成が必要なら `output-transaction-only` を使う

ノード出力を作り直さず、リンクトランザクションだけ再生成したい場合はこれを使います。

```sh
python -m shoestring setup \
  --config work/sai.multisig.ini \
  --directory work/node \
  --ca-key-path work/cosigner1.pem \
  --output-transaction-only
```

使いどころ:

- `minCosignaturesCount` を更新した後に aggregate 種別を作り直したい。
- 既存リンク状態が変わった後に `linking_transaction.dat` を再取得したい。

## 7. オフライン署名を集める

最初の署名者は aggregate の外側署名者になり、aggregate bonded の場合は hash lock の支払い者にもなります。ここを無計画に選ぶな。XYM を持っているコサイナーを最初に使う。

### 7-1. 先頭コサイナーで署名する

```sh
python -m shoestring signer \
  --config work/sai.multisig.ini \
  --ca-key-path work/cosigner1.pem \
  --save \
  work/node/linking_transaction.dat
```

`minCosignaturesCount >= 2` の場合、追加で以下が生成されます。

- `work/node/linking_transaction.hash_lock.dat`

### 7-2. 残りのコサイナーで同じ payload に追記署名する

```sh
python -m shoestring signer \
  --config work/sai.multisig.ini \
  --ca-key-path work/cosigner2.pem \
  --save \
  work/node/linking_transaction.dat

python -m shoestring signer \
  --config work/sai.multisig.ini \
  --ca-key-path work/cosigner3.pem \
  --save \
  work/node/linking_transaction.dat
```

確認ポイント:

- 同じ `linking_transaction.dat` にコサインが追記される。
- 既に同じコサイナーで署名済みなら重複追加されない。
- aggregate bonded の hash lock ファイルは再生成・上書きされない。

## 8. testnet にアナウンスする

### 8-1. `minCosignaturesCount >= 2` の場合

まず hash lock を送る。

```sh
python -m shoestring announce-transaction \
  --config work/sai.multisig.ini \
  --transaction work/node/linking_transaction.hash_lock.dat
```

hash lock 確認後、aggregate bonded を送る。

```sh
python -m shoestring announce-transaction \
  --config work/sai.multisig.ini \
  --transaction work/node/linking_transaction.dat
```

### 8-2. `minCosignaturesCount <= 1` の場合

aggregate complete になるので `linking_transaction.dat` のみ送る。

```sh
python -m shoestring announce-transaction \
  --config work/sai.multisig.ini \
  --transaction work/node/linking_transaction.dat
```

## 9. 成功判定

最低限、以下を確認します。

- `announce-transaction` がエラーを返していない。
- testnet エクスプローラまたはノード API で対象 aggregate を確認できる。
- 内包トランザクションに期待する link/unlink が含まれている。
- `HARVESTER` を有効にした場合:
  - `Account Key Link`
  - `VRF Key Link`
- `VOTER` を有効にした場合:
  - `Voting Key Link`
- 既存リンクを差し替えるケースでは、古いキーへの `unlink` と新しいキーへの `link` が両方確認できる。

## 10. 手動テスト記録テンプレート

以下だけ埋めれば、再現性のある記録になります。

```text
実施日:
実施者:
ネットワーク: sai
対象マルチシグアドレス:
minCosignaturesCount:
使用コサイナー:
有効 feature:
setup 出力:
  - linking_transaction.dat: 生成 / 未生成
  - linking_transaction.hash_lock.dat: 生成 / 未生成
期待した inner transaction:
実際に確認した inner transaction:
hash lock 送信結果:
aggregate 送信結果:
エクスプローラ確認結果:
問題点:
```

## よくあるミス

- `signerPublicKey` を設定せずに実行して、通常アカウント向けリンクトランザクションを作ってしまう。
- `minCosignaturesCount` を更新せずに実行して、aggregate complete を作ってしまう。
- 先頭署名者に XYM がなく、hash lock を送れない。
- 既存リンクがあるのに `link` だけを期待して、`unlink + link` を不具合扱いする。
- `imports.harvester` または `imports.voter` を設定したままで、新規リンクが生成されない。

## 切り分けの順序

不具合に見えたら、順番を固定して確認します。

1. `pemview` で各コサイナー PEM のアドレスを確認する。
2. INI の `transaction.signerPublicKey` がマルチシグ本体の公開鍵か確認する。
3. `minCosignaturesCount` が実際の `minApproval` と一致しているか確認する。
4. `setup` 実行後に `linking_transaction.dat` が生成されているか確認する。
5. 初回 `signer` 後に `hash_lock.dat` の要否が aggregate 種別と一致しているか確認する。
6. `announce-transaction` 実行順が `hash lock -> aggregate bonded` になっているか確認する。
