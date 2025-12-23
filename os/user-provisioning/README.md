# 手動 Run Command 一発で、複数ユーザー＋鍵＋sudo＋パスワード無効化

## 全体方針

 - 完全手動実行
 EC2 起動後、オペレータが Run Command を実行するだけ

 - S3 に公開鍵を置く
 s3://<鍵用バケット>/<prefix>/<ユーザー名>.pub のように管理

 - Run Command で実行する カスタム SSM ドキュメント（aws:runShellScript） を 1 本作る

    そのドキュメントの中で
    1. ユーザー作成
    2. S3 から公開鍵取得 → ~/.ssh/authorized_keys 設定
    3. sudo 権限付与（/etc/sudoers.d）
    4. SSH パスワード認証無効化（/etc/ssh/sshd_config 編集＆リロード）

### 鍵管理の前提（S3）
例として、次のように管理します。

 - バケット名：pj-ops-ssh-keys
 - プレフィックス：env/prod/（環境ごとでもプロジェクトごとでも OK）
 
 S3 上のファイル構成例
```
s3://pj-ops-ssh-keys/env/prod/sysuser.pub
s3://pj-ops-ssh-keys/env/prod/appuser.pub
```

ファイル名は OS ユーザー名と同じ + .pub にしておくと、スクリプト側が楽です。


### カスタム SSM ドキュメント（YAML）

Systems Manager > Documents から Command document を新規作成し、名前を DOC-INIT-OS-USERS とします。

内容イメージ（Linux 用）: doc-init-users.yml


## 実行手順（オペレーション観点）

### 1. 前提確認

 - EC2 インスタンスにSSM Agent インストール済み
 - IAM ロール AmazonSSMManagedInstanceCore 付与
 - インスタンスから S3（鍵バケット）へ到達可能（インターネット or VPC エンドポイント）
 - OS に awscli がインストール済み(Amazon Linux 系ならだいたい入ってますが、RHEL なら個別に入れておく)

### 2. S3 に公開鍵を配置

ローカルの公開鍵例：sysuser.pub, appuser.pub

アップロード例
```
aws s3 cp sysuser.pub s3://pj-ops-ssh-keys/env/prod/sysuser.pub
aws s3 cp appuser.pub  s3://pj-ops-ssh-keys/env/prod/appuser.pub
```

### 3. SSM ドキュメントを登録

 - 前述の YAML を貼り付けて DOC-INIT-OS-USERS を作成
 - 作成後、一度テスト用インスタンスで試すことを推奨

### 4. Run Command を実行

 - Systems Manager > Run Command > 「コマンドの実行」
 - ドキュメント：DOC-INIT-OS-USERS
 - コマンドのパラメータ例
     - SshKeyBucket：pj-ops-ssh-keys
     - SshKeyPrefix：env/prod
     - UserNames：sysuser,appuser
     - SudoUsers：sysuser（sysuser,appuser としても OK）
     - DisablePasswordAuth：yes
     
 - ターゲット：該当インスタンスを選択
 - 実行後、コマンドの出力ログ／インスタンス側の /var/log/messages などで確認

### 5. 動作確認

 - クライアント側から公開鍵で SSH 接続
    → sysuser@app-ip / appuser@app-ip
 - sudo -l で sudo 権限の確認
 - /etc/ssh/sshd_config に PasswordAuthentication no が入っているか確認