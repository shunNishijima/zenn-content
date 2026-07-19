---
title: "Rails の暗号化とハッシュ：master.key と Devise が内部でやっていること"
emoji: "🤖"
type: "tech"
topics: ["rails", "ai"]
published: true
---

# Rails の暗号化とハッシュ：master.key と Devise が内部でやっていること

**この記事の要点**
- `master.key` は AES-256-GCM で暗号化された `credentials.yml.enc` を復号するための対称鍵であり、紛失すると credentials は永久に読めなくなる
- Devise はパスワードをハッシュ（不可逆変換）で保存するため、「元のパスワードに戻す」ことは原理的にできない
- 暗号化（可逆）とハッシュ化（不可逆）は目的が違う。Rails はユースケースに応じて両方を使い分けている

---

## 暗号化とハッシュ：まず言葉を整理する

Rails を使っていると「暗号化」と「ハッシュ化」という言葉が混在して登場します。どちらも「平文をそのまま保存しない」という点は共通ですが、根本的な目的が異なります。

| 操作 | 方向性 | 代表例 |
|---|---|---|
| 暗号化（Encryption） | 可逆（鍵があれば復元できる） | credentials.yml.enc、ActiveRecord::Encryption |
| ハッシュ化（Hashing） | 不可逆（元には戻せない） | Devise のパスワード保存 |

「なぜパスワードをハッシュにするのか」「なぜ API キーは暗号化なのか」——この区別を理解すると、Rails のセキュリティ設計が一本の筋として見えてきます。

---

## Rails Credentials の仕組み：master.key は何をしている鍵か

### credentials.yml.enc の正体

Rails 5.2 以降、機密情報の管理には Credentials という仕組みが使われています。`config/credentials.yml.enc` がリポジトリに入っており、`config/master.key` は `.gitignore` に追加して管理します。

```bash
# 暗号化済みファイルを編集する
rails credentials:edit
```

このコマンドを実行すると、`master.key` を使って一時的に復号した YAML ファイルをエディタで開き、保存時に再暗号化します。

### 暗号アルゴリズム：AES-256-GCM

Rails が使うのは **AES-256-GCM**（Advanced Encryption Standard、256 ビット鍵、GCM モード）です。これは対称暗号の一種で、暗号化・復号に同じ鍵（master.key）を使います。

GCM（Galois/Counter Mode）はストリーム暗号モードの一種で、認証タグを付与することで改ざん検出も行います。単純に「秘密にする」だけでなく「内容が書き換えられていないことを保証する」という特性も持ちます。

`master.key` の中身は 32 バイトのランダムな16進数文字列（例: `a3f2c1...`）です。これが流出すると `credentials.yml.enc` の内容がすべて読まれてしまうため、本番環境では環境変数 `RAILS_MASTER_KEY` として注入するのが一般的です。

### ActiveSupport::MessageEncryptor を手で動かす

Rails が credentials の暗号化に内部で使っているのは `ActiveSupport::MessageEncryptor` です。コンソールで実際に触れます。

```ruby
require "active_support/message_encryptor"

key = ActiveSupport::KeyGenerator.new("passphrase").generate_key("salt", 32)
encryptor = ActiveSupport::MessageEncryptor.new(key)

encrypted = encryptor.encrypt_and_sign("my secret api key")
# => "eyJfcmFpbHMiOnsibWVzc2...（長い文字列）"

decrypted = encryptor.decrypt_and_verify(encrypted)
# => "my secret api key"
```

`encrypt_and_sign` は暗号化と署名を同時に行い、`decrypt_and_verify` は署名検証のあとに復号します。改ざんされた暗号文を渡すと `ActiveSupport::MessageVerifier::InvalidSignature` が発生するので、安全に使えます。

実際にコンソールで試してみると、同じ平文でも呼び出すたびに異なる暗号文が生成されることが確認できます。GCM モードでは暗号化のたびに **IV（初期化ベクトル）** がランダムに生成されるためです。これにより、同じ値が暗号化されても暗号文が推測できません。

---

## Devise のパスワード保存：なぜ元のパスワードに戻せないのか

### bcrypt とは何か

Devise はデフォルトでパスワードを **bcrypt** でハッシュ化して保存します（[bcrypt-ruby gem](https://github.com/bcrypt-ruby/bcrypt-ruby)）。bcrypt は暗号学的ハッシュ関数ではなくパスワードハッシュ専用のアルゴリズムで、以下の特性を持ちます。

1. **一方向性**：ハッシュ値から元のパスワードを計算することは（現実的には）不可能
2. **ソルト**：同じパスワードでも毎回異なるハッシュが生成される。レインボーテーブル攻撃を防ぐ
3. **ストレッチング（コストパラメータ）**：計算を意図的に遅くすることでブルートフォース攻撃を困難にする

Devise のデフォルトコストは `11` です（`Devise.stretches` で変更可能）。コスト 11 では 2^11 = 2048 回の内部反復が行われます。

### データベースに保存される値

`users` テーブルの `encrypted_password` カラムには、こんな文字列が入っています。

```
$2a$11$hKUL...（60 文字）
```

この文字列は次の部分に分かれています。

```
$2a$   - bcrypt のバージョン識別子
$11$   - コストパラメータ（= 2^11 回反復）
hKUL..(22文字)  - ソルト（自動生成・ランダム）
残り31文字       - 実際のハッシュ値
```

ソルトはハッシュの中に一緒に埋め込まれているため、検証時に別テーブルで管理する必要はありません。

### パスワード検証の内部フロー

ユーザーがログインフォームにパスワードを入力したとき、Devise は次のことをしています。

```ruby
# Devise が内部でやっていること（概念的な疑似コード）
def valid_password?(plain_password)
  bcrypt_hash = BCrypt::Password.new(self.encrypted_password)
  # BCrypt::Password#== が内部で以下を行う
  # 1. 保存済みハッシュからソルトとコストを取り出す
  # 2. 入力パスワードを同じソルト・コストでハッシュ化
  # 3. 2つのハッシュを比較
  bcrypt_hash == plain_password
end
```

重要なのは「保存済みハッシュと入力値のハッシュを比較している」ことです。平文に戻す操作は一切行われません。

bcrypt-ruby gem を直接使うとこの動きを確認できます。

```ruby
require "bcrypt"

hash = BCrypt::Password.create("my_password", cost: 11)
# => "$2a$11$..."

BCrypt::Password.new(hash) == "my_password"
# => true

BCrypt::Password.new(hash) == "wrong_password"
# => false
```

---

## 暗号化とハッシュを使い分けるRailsの設計思想

### なぜパスワードをハッシュにするのか

答えは単純です。「元のパスワードをアプリケーションが知る必要がない」からです。ログイン検証には「同じ値かどうかを確認できれば十分」であり、復元できてしまうと DB 漏洩時の被害が大きくなります。

一方 API キーや外部サービスのトークンは、Rails から外部 API に送信する必要があるため、復号できなければ使えません。だから credentials（暗号化）で管理します。

### ActiveRecord::Encryption（Rails 7 以降）

Rails 7.0 で導入された `ActiveRecord::Encryption` を使うと、モデル属性を透過的に暗号化できます。

```ruby
class User < ApplicationRecord
  encrypts :phone_number
end

user = User.create!(phone_number: "090-xxxx-xxxx")
# DB には暗号化された値が保存される

user.phone_number
# => "090-xxxx-xxxx"  （読み出し時に自動復号）
```

この機能は内部で `ActiveSupport::MessageEncryptor` と同じ仕組みを使っており、鍵の管理も credentials 経由で行います。個人情報や機密属性を保存するときに有効です（[Rails Guides: Active Record Encryption](https://guides.rubyonrails.org/active_record_encryption.html)）。

### セッションストアとの関係

Rails のデフォルトセッションストア（Cookie Store）も同様に `secret_key_base`（credentials に保存）を使った署名・暗号化が行われています。セッションデータは暗号化されたうえでクライアントの Cookie に保存されますが、`secret_key_base` が変わるとすべての既存セッションが無効化されます。これも「可逆な暗号化」の一例です。

---

## まとめ

Rails が提供する暗号化・ハッシュ機能を整理すると次のようになります。

| 機能 | 方式 | アルゴリズム | 目的 |
|---|---|---|---|
| credentials.yml.enc | 暗号化（可逆） | AES-256-GCM | API キーなど機密情報の保存 |
| Devise パスワード | ハッシュ（不可逆） | bcrypt | パスワード検証 |
| ActiveRecord::Encryption | 暗号化（可逆） | AES-256-GCM | DB カラムの機密保護 |
| Cookie セッション | 暗号化 + 署名 | AES-256-GCM + HMAC | セッションの改ざん防止 |

「後で取り出す必要があるか」という問いが、暗号化とハッシュを選ぶときの判断軸になります。フレームワークを使うだけでなく内部の仕組みを理解しておくと、セキュリティ設計のレビューやインシデント対応で迷わなくなります。

---

## よくある質問

**Q. master.key を紛失したら credentials.yml.enc の内容は取り出せますか？**

取り出せません。AES-256-GCM は対称暗号であり、鍵なしでの復号は現実的に不可能です。本番環境では `RAILS_MASTER_KEY` を環境変数として安全な場所（AWS Secrets Manager など）に保管し、master.key 本体のバックアップも別途確保しておくことが推奨されます。

**Q. Devise のパスワードを「リセット」するときはどうしているのですか？**

「復号」は行っていません。パスワードリセット時は、一時的に有効なトークンをメールで送り、そのリンクから新しいパスワードを再設定させるフローです。トークン自体もデータベースには `digest`（ハッシュ）で保存されており、`Devise::TokenGenerator` を通じて検証されます。

**Q. bcrypt のコストパラメータを上げるとどうなりますか？**

コストを 1 増やすと計算時間が約 2 倍になります。コスト 11（デフォルト）→ コスト 12 にすると、パスワードハッシュの計算が 2 倍遅くなる代わり、総当たり攻撃も 2 倍困難になります。ただしログイン処理のレスポンスタイムにも影響するため、サーバースペックと攻撃耐性のバランスで決めます。本番環境の実測値をもとに調整するのが現実的なアプローチです（`Devise.stretches` で変更可能）。
