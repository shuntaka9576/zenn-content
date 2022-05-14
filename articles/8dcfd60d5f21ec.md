---
title: "AWSのクレデンシャルをローテートしつつ、LitestreamでSQLite3をS3にレプリケートする"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["awsiot", "litestream", "reterminal", "raspberrypi"]
published: true
---

# はじめに

最近reTerminalを購入して、AWS IoT Coreと接続して遊んでいます。^[reTerminalの詳細は、[reTerminalが届いたので初期設定とかATECC608Aのシリアル取得してみる](https://zenn.dev/shuntaka/scraps/23b5e1fc6c5e0b)を参照]
SQLiteはIoT分野で軽量なSQLデータベースとして、デバイスの中に入れやすく(=オフライン利用でき)便利です。(組み込みの場合は状況が違うかもしれません)
LitestreamでS3へレプリケートできたら活用できる機会がありそう(?)なので試してみることにしました。

課題は、AWS IoTの証明書と秘密鍵を利用してクレデンシャルを発行する場合、クレデンシャル期限が最短900秒から最長43200秒であることです。^[[AWS IoT Core認証情報プロバイダーを使用して、AWSサービスの直接呼出しを認証](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/authorizing-direct-aws.html)より]

永続的なクレデンシャルを数百台のデバイスで利用する場合、以下の問題があります。
* 台数分のクレデンシャルを利用する場合、発行・管理コストが大きい
* 1つのクレデンシャルを全てのデバイスで利用する場合、クレデンシャル流出等でrevokeする必要があるとき、影響範囲が大きくコストが高い

Litestreamのissueで[Thoughts on using Litestream with time-limited AWS credentials?](https://github.com/benbjohnson/litestream/issues/246#issuecomment-962635025)を見ると、`Litestreamを再起動せずにSIGHUPを送信して設定ファイルを再読み込みするサポートを追加したい`とはあります。
現状は、前述した機能はないため、クレデンシャルが切れそうになったら更新しつつ、Litestreamを再起動することにします。再起動時に書き込まれている内容は、レプリケートされないので許容できる人向けです。

# 前提

* reTerminalのHSM(ATECC608A)の秘密鍵がRead不可なスロットを利用し、PKCS#11でCSRを発行、AWS IoTで証明書を発行済み ^[証明書発行方法は、[[reTerminal] PKCS#11を用いて秘密鍵を安全に管理しつつAWS IoTとMQTT通信する](https://dev.classmethod.jp/articles/shuntaka-reterminal-pkcs11/)を参考]
* [AWS IoT Core認証情報プロバイダーを使用して、AWSサービスの直接呼出しを認証](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/authorizing-direct-aws.html)設定導入済み

reTerminalのOSはRaspbianです。ラズパイでも似たようなことができますが、HSMはついていないので読み替える必要があります。
```bash
$ lsb_release -a
No LSB modules are available.
Distributor ID: Raspbian
Description:    Raspbian GNU/Linux 10 (buster)
Release:        10
Codename:       buster
```

# 設定

## systemdの設定

litestreamの設定は以下のようにします。

```text:/etc/litestream.yml
dbs:
 - path: /home/pi/reteminal-app/.db/test.db
   replicas:
     - type: s3
       bucket: YOUR_S3_NAME
       path: replica
       region: ap-northeast-1
```

Litestreamは、インストールすると[Unit定義ファイル](https://github.com/benbjohnson/litestream/blob/main/etc/litestream.service)を配備します。今回はクレデンシャル読み込みに環境変数を利用するため、EnviromentFile設定を追記します。
```text:/lib/systemd/system/litestream.service
[Unit]
Description=Litestream

[Service]
Restart=always
ExecStart=/usr/bin/litestream replicate
EnvironmentFile=/home/pi/.config/sysconfig/litestream

[Install]
WantedBy=multi-user.target
```

EnviromentFileにはAWSのクレデンシャルをアプリで定期的に更新、systemdでLitestreamを再起動します
```text:/home/pi/.config/sysconfig/litestream
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_SESSION_TOKEN=
AWS_DEFAULT_REGION=
```

# ソースコード

:::message
コードの分量が多いため、主要な部分のみ掲載します。要望があれば、GitHubで公開いたします。
:::


:::details SQLiteのデータベース、テーブル作成コード
```sqlite3
sqlite3 test.db
CREATE TABLE HealthCheck (id INTEGER PRIMARY KEY,created_at TEXT DEFAULT CURRENT_TIMESTAMP);
select id,datetime(created_at, '+9 hours') from HealthCheck;
```
:::

アプリはGoを利用します。

:::details エントリポイントのコード

```go:cmd/edgego/main.go
package main

import (
	"context"
	"fmt"
	"io/ioutil"
	"os"
	"os/exec"
	"time"

	"github.com/shuntaka9576/edgego/domain"
	"github.com/shuntaka9576/edgego/logger"
)

var (
	credential                     *domain.Credentials
	SYSTEMD_LITESTREAM_CONFIG_PATH = os.Getenv("SYSTEMD_LITESTREAM_CONFIG_PATH")
)

func main() {
	credentialCh := make(chan *domain.Credentials)

	go func() {
		for {
			select {
			case credential = <-credentialCh:
				logger.Info().Msgf("get credential main expired at %s", credential.Expiration.Format(time.RFC3339))
				litestreamEnvString := fmt.Sprintf(""+
					"AWS_ACCESS_KEY_ID=\"%s\"\n"+
					"AWS_SECRET_ACCESS_KEY=\"%s\"\n"+
					"AWS_SESSION_TOKEN=\"%s\"\n"+
					"AWS_DEFAULT_REGION=ap-northeast-1",
					credential.AccessKeyId, credential.SecretAccessKey, credential.SessionToken)

				err := ioutil.WriteFile(SYSTEMD_LITESTREAM_CONFIG_PATH, []byte(litestreamEnvString), 0666)
				if err != nil {
					logger.Info().Msgf("writte error", err)
					continue
				}

				cmd := exec.Command("sudo", "systemctl", "restart", "litestream")
				logger.Info().Msgf("start restart litestream")

				err = cmd.Start()
				if err != nil {
					logger.Info().Msgf("start litestream exec error", err)
					continue
				}

				err = cmd.Wait()
				if err != nil {
					logger.Info().Msgf("restart litestream exec error", err)
					continue
				} else {
					logger.Info().Msgf("restart sucess litestream")
				}
			}
		}
	}()

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	lotateCredentilasUseCase := &domain.LotateCredentilasUseCase{ExpiredAt: nil, UpdateAt: nil}
	go lotateCredentilasUseCase.Run(ctx, credentialCh)

	routineWriteSqlite3 := &domain.RoutineWriteSqlite3{}
	go routineWriteSqlite3.Run(ctx)

	<-ctx.Done()
}
```
:::

:::details 定期的にAWSクレデンシャルを更新するコード
PKCS#11を利用するため、`libcurl`のバインディングである[go-curl](https://github.com/andelf/go-curl)を利用します。

```go:クレデンシャルを取得する関数
package iot

import (
	"encoding/json"
	"os"

	curl "github.com/andelf/go-curl"
)

var (
	CREDENTIAL_ALIAS_ENDPOINT = os.Getenv("CREDENTIAL_ENDPOINT") +
		"/role-aliases/" +
		os.Getenv("CREDENTIAL_ROLE_ALIAS") +
		"/credentials"
	DEVICE_CERT_PATH                 = os.Getenv("DEVICE_CERT_PATH")
	DEVICE_PRIVATE_KEY_PKCS11_PHRASE = os.Getenv("DEVICE_PRIVATE_KEY_PKCS11_PHRASE")
)

type RoleAliasResponse struct {
	Credentials CredentialsResponse `json:"credentials"`
}

type CredentialsResponse struct {
	AccessKeyId     string `json:"accessKeyId"`
	SecretAccessKey string `json:"secretAccessKey"`
	SessionToken    string `json:"sessionToken"`
	Expiration      string `json:"expiration"`
}

func GetCredentialsForCert() (*RoleAliasResponse, error) {

	easy := curl.EasyInit()
	defer easy.Cleanup()

	easy.Setopt(curl.OPT_URL, CREDENTIAL_ALIAS_ENDPOINT)
	easy.Setopt(curl.OPT_SSL_VERIFYPEER, 0)
	easy.Setopt(curl.OPT_SSL_VERIFYHOST, false)
	easy.Setopt(curl.OPT_VERBOSE, 0)
	easy.Setopt(curl.OPT_SSLENGINE, "pkcs11")
	easy.Setopt(curl.OPT_SSLCERTTYPE, "PEM")
	easy.Setopt(curl.OPT_SSLKEYTYPE, "eng")
	easy.Setopt(curl.OPT_SSLCERT, DEVICE_CERT_PATH)
	easy.Setopt(curl.OPT_SSLKEY, DEVICE_PRIVATE_KEY_PKCS11_PHRASE)

	roleAliasRes := &RoleAliasResponse{}
	execFunc := func(buf []byte, userdata interface{}) bool {
		if err := json.Unmarshal(buf, roleAliasRes); err != nil {
			return false
		}

		return true
	}

	easy.Setopt(curl.OPT_WRITEFUNCTION, execFunc)
	err := easy.Perform()
	if err != nil {
		return nil, err
	}

	return roleAliasRes, nil
}
```

```go:クレデンシャルを定期更新するgoroutine
package domain

import (
	"context"
	"time"

	"github.com/shuntaka9576/edgego/infra/iot"
	"github.com/shuntaka9576/edgego/logger"
)

var (
	LOTATE_BUFFER_TIME_MINUTES int = 5
)

type LotateCredentilasUseCase struct {
	ExpiredAt *time.Time
	UpdateAt  *time.Time
}

func (l *LotateCredentilasUseCase) Run(ctx context.Context, credentialCh chan *Credentials) {
	go func() {
		ticker := time.NewTicker(5 * time.Second)

	LOOP:
		for {
			select {
			case <-ticker.C:

				nowTime := time.Now()

				if l.ExpiredAt == nil && l.UpdateAt == nil || l.UpdateAt.Unix() <= nowTime.Unix() {
					logger.Info().Msgf("start update credential")

					if l.ExpiredAt != nil && l.UpdateAt != nil {
						logger.Info().Msgf("now: %s", nowTime.Format(time.RFC3339))
						logger.Info().Msgf("UpdateAt: %s", l.UpdateAt.Format(time.RFC3339))
						logger.Info().Msgf("ExpiredAt: %s", l.ExpiredAt.Format(time.RFC3339))
					} else {
						logger.Info().Msgf("init credential")
					}

					res, err := iot.GetCredentialsForCert()

					if err != nil {
						logger.Warn().Msgf("get crendential error: %s", err)

						continue
					}

					parsedTime, err := time.Parse(time.RFC3339, res.Credentials.Expiration)
					if err != nil {
						logger.Warn().Msgf("parsed time error: %s", err)
						continue
					}
					updateAt := parsedTime.Add(-time.Minute * time.Duration(LOTATE_BUFFER_TIME_MINUTES))

					l = &LotateCredentilasUseCase{
						ExpiredAt: &parsedTime,
						UpdateAt:  &updateAt,
					}

					logger.Info().Msgf("updated credential success. expired at %s", l.ExpiredAt.Format(time.RFC3339))
					credentialCh <- &Credentials{
						AccessKeyId:     res.Credentials.AccessKeyId,
						SecretAccessKey: res.Credentials.SecretAccessKey,
						SessionToken:    res.Credentials.SessionToken,
						Expiration:      *l.ExpiredAt,
					}
				}
			case <-ctx.Done():
				logger.Info().Msg("Done")
				break LOOP
			}
		}
	}()
}
```
:::


:::details 定期的にSQLite3を更新するコード

以下の内容をSQLite3に書き込みます

|DBのフィールド|書き込む内容|
|---|---|
|id|起動時タイムスタンプを取得、5秒おきに1ずつインクリメントして書き込み(特に意図はないです)|
|create_at|SQLite3の組み込み変数を利用して現在日時を書き込み

```go:定期的にSQLite3を更新するgoroutine
package domain

import (
	"context"
	"database/sql"
	"fmt"
	"os"
	"time"

	"github.com/shuntaka9576/edgego/logger"

	_ "github.com/mattn/go-sqlite3"
)

var (
	SQLITE3_SAMPLE_DB_PATH = os.Getenv("SQLITE3_SAMPLE_DB_PATH")
	TABLE_NAME             = "HealthCheck"
)

type RoutineWriteSqlite3 struct {
}

func (l *RoutineWriteSqlite3) Run(ctx context.Context) {
	con, _ := sql.Open("sqlite3", SQLITE3_SAMPLE_DB_PATH)

	go func() {
		ticker := time.NewTicker(5 * time.Second)
		defer con.Close()
		id := time.Now().Unix()

	LOOP:
		for {
			select {
			case <-ticker.C:
				logger.Info().Msgf("sqlite3 write start")

				cmd := fmt.Sprintf("INSERT INTO %s (id) VALUES (%d);",
					TABLE_NAME,
					id,
				)
				_, err := con.Exec(cmd)
				if err != nil {
					logger.Err().Msgf("sqlite3 write err: %s", err)
					continue
				}
				id += 1
				logger.Info().Msgf("sqlite3 write success")
			case <-ctx.Done():
				logger.Info().Msg("Done")
				break LOOP
			}
		}
	}()
}
```
:::



# 結果確認

## 前提

* クレデンシャルは15分でexpire。expireする5分前にクレデンシャルの更新とLitestreamの再起動が走る(=10分毎にクレデンシャルの更新とLitestreamの再起動が走る)
* SQLiteへ書き込みは5秒に1回
* クレデンシャル更新とSQLiteへの書き込みは別goroutine上で動作
* Goアプリ起動前の状態
  * SQLiteのデータベース、テーブルは作成済み、空の状態
  * S3は空の状態
  * Litestreamは、Goアプリから起動されるため、サービス停止状態

## 確認したいこと
1時間程度実行したらGoアプリとLitestreamを停止し、デバイス側とS3からリストアしたテーブルのレコード数が一致することを確認する^[Litestreamは`journalctl -u litestream`で、Goアプリは自身が吐いたログを確認します。]

## 結果

### 経過概要

|時刻|対象|イベント|
|---|---|--|
|7:45:00|手動|Goアプリ起動|
|7:45:03|Goアプリログ|起動ログ
|7:45:08|Goアプリログ|1回目のSQLite書き込み
|7:45:09|Goアプリログ|Litestreamのrestart成功ログ
|7:45:13|Goアプリログ|2回目のSQLite書き込み
|(中略)|(中略)|(中略)|
|8:50:00|手動|Goアプリ起動停止|
|8:50:00|手動|Litestream停止|


### 各種ログ確認

ログみる限り、再起動にかかる時間(レプリケートされないダウンタイム)は、1秒未満に見えます。

:::details Litestreamサービスログ
```
 5月 14 07:45:09 raspberrypi systemd[1]: Started Litestream. 👈 Goアプリから再起動
 5月 14 07:45:09 raspberrypi litestream[2239]: litestream v0.3.8 
 5月 14 07:45:09 raspberrypi litestream[2239]: initialized db: /home/pi/auto/private-reteminal-app/.db/test.db
 5月 14 07:45:09 raspberrypi litestream[2239]: replicating to: name="s3" type="s3" bucket="YOUR_S3_NAME" path="replica" region="ap-northeast-1" endpoint="" sync-interval=1s
 5月 14 07:45:10 raspberrypi litestream[2239]: /home/pi/auto/private-reteminal-app/.db/test.db: sync: new generation "404fe1ff630ab2f0", no generation exists
 5月 14 07:45:16 raspberrypi litestream[2239]: /home/pi/auto/private-reteminal-app/.db/test.db(s3): snapshot written 404fe1ff630ab2f0/00000001
 5月 14 07:55:14 raspberrypi litestream[2239]: signal received, litestream shutting down
 5月 14 07:55:14 raspberrypi systemd[1]: Stopping Litestream...
 5月 14 07:55:14 raspberrypi litestream[2239]: litestream shut down
 5月 14 07:55:14 raspberrypi systemd[1]: litestream.service: Succeeded.
 5月 14 07:55:14 raspberrypi systemd[1]: Stopped Litestream.
 5月 14 07:55:14 raspberrypi systemd[1]: Started Litestream.
 5月 14 07:55:14 raspberrypi litestream[2443]: litestream v0.3.8
 5月 14 07:55:14 raspberrypi litestream[2443]: initialized db: /home/pi/auto/private-reteminal-app/.db/test.db
 5月 14 07:55:14 raspberrypi litestream[2443]: replicating to: name="s3" type="s3" bucket="YOURDB" path="replica" region="ap-northeast-1" endpoint="" sync-interval=1s
 5月 14 08:05:19 raspberrypi litestream[2443]: signal received, litestream shutting down
 5月 14 08:05:19 raspberrypi systemd[1]: Stopping Litestream...
 5月 14 08:05:19 raspberrypi litestream[2443]: litestream shut down
 5月 14 08:05:19 raspberrypi systemd[1]: litestream.service: Succeeded.
 5月 14 08:05:19 raspberrypi systemd[1]: Stopped Litestream.
 5月 14 08:05:19 raspberrypi systemd[1]: Started Litestream.
 5月 14 08:05:19 raspberrypi litestream[2519]: litestream v0.3.8
 5月 14 08:05:19 raspberrypi litestream[2519]: initialized db: /home/pi/auto/private-reteminal-app/.db/test.db
 5月 14 08:05:19 raspberrypi litestream[2519]: replicating to: name="s3" type="s3" bucket="YOUR_S3_NAME" path="replica" region="ap-northeast-1" endpoint="" sync-interval=1s
 5月 14 08:15:24 raspberrypi litestream[2519]: signal received, litestream shutting down
 5月 14 08:15:24 raspberrypi systemd[1]: Stopping Litestream...
 5月 14 08:15:24 raspberrypi litestream[2519]: litestream shut down
 5月 14 08:15:24 raspberrypi systemd[1]: litestream.service: Succeeded.
 5月 14 08:15:24 raspberrypi systemd[1]: Stopped Litestream.
 5月 14 08:15:24 raspberrypi systemd[1]: Started Litestream.
 5月 14 08:15:24 raspberrypi litestream[2556]: litestream v0.3.8
 5月 14 08:15:24 raspberrypi litestream[2556]: initialized db: /home/pi/auto/private-reteminal-app/.db/test.db
 5月 14 08:15:24 raspberrypi litestream[2556]: replicating to: name="s3" type="s3" bucket="YOUR_S3_NAME" path="replica" region="ap-northeast-1" endpoint="" sync-interval=1s
 5月 14 08:25:29 raspberrypi litestream[2556]: signal received, litestream shutting down
 5月 14 08:25:29 raspberrypi systemd[1]: Stopping Litestream...
 5月 14 08:25:29 raspberrypi litestream[2556]: litestream shut down
 5月 14 08:25:29 raspberrypi systemd[1]: litestream.service: Succeeded.
 5月 14 08:25:29 raspberrypi systemd[1]: Stopped Litestream.
 5月 14 08:25:29 raspberrypi systemd[1]: Started Litestream.
 5月 14 08:25:29 raspberrypi litestream[2603]: litestream v0.3.8
 5月 14 08:25:29 raspberrypi litestream[2603]: initialized db: /home/pi/auto/private-reteminal-app/.db/test.db
 5月 14 08:25:29 raspberrypi litestream[2603]: replicating to: name="s3" type="s3" bucket="YOUR_S3_NAME" path="replica" region="ap-northeast-1" endpoint="" sync-interval=1s
 5月 14 08:35:34 raspberrypi litestream[2603]: signal received, litestream shutting down
 5月 14 08:35:34 raspberrypi systemd[1]: Stopping Litestream...
 5月 14 08:35:34 raspberrypi litestream[2603]: litestream shut down
 5月 14 08:35:34 raspberrypi systemd[1]: litestream.service: Succeeded.
 5月 14 08:35:34 raspberrypi systemd[1]: Stopped Litestream.
 5月 14 08:35:34 raspberrypi systemd[1]: Started Litestream.
 5月 14 08:35:34 raspberrypi litestream[2674]: litestream v0.3.8
 5月 14 08:35:34 raspberrypi litestream[2674]: initialized db: /home/pi/auto/private-reteminal-app/.db/test.db
 5月 14 08:35:34 raspberrypi litestream[2674]: replicating to: name="s3" type="s3" bucket="shuntaka20220512" path="replica" region="ap-northeast-1" endpoint="" sync-interval=1s
 5月 14 08:45:40 raspberrypi litestream[2674]: signal received, litestream shutting down
 5月 14 08:45:40 raspberrypi systemd[1]: Stopping Litestream...
 5月 14 08:45:40 raspberrypi litestream[2674]: litestream shut down
 5月 14 08:45:40 raspberrypi systemd[1]: litestream.service: Succeeded.
 5月 14 08:45:40 raspberrypi systemd[1]: Stopped Litestream.
 5月 14 08:45:40 raspberrypi systemd[1]: Started Litestream.
 5月 14 08:45:40 raspberrypi litestream[2848]: litestream v0.3.8
 5月 14 08:45:40 raspberrypi litestream[2848]: initialized db: /home/pi/auto/private-reteminal-app/.db/test.db
 5月 14 08:45:40 raspberrypi litestream[2848]: replicating to: name="s3" type="s3" bucket="shuntaka20220512" path="replica" region="ap-northeast-1" endpoint="" sync-interval=1s
 5月 14 08:50:03 raspberrypi litestream[2848]: signal received, litestream shutting down
 5月 14 08:50:03 raspberrypi systemd[1]: Stopping Litestream... 👈 手動停止ログ(sudo systemctl stop litestream)
 5月 14 08:50:03 raspberrypi litestream[2848]: litestream shut down
 5月 14 08:50:03 raspberrypi systemd[1]: litestream.service: Succeeded.
 5月 14 08:50:03 raspberrypi systemd[1]: Stopped Litestream.
```
:::

:::details Goアプリログ(一部省略)
```
2022-05-14T07:45:03+09:00 INFO   start app
2022-05-14T07:45:08+09:00 INFO   start update credential
2022-05-14T07:45:08+09:00 INFO   sqlite3 write start
2022-05-14T07:45:08+09:00 INFO   init credential
2022-05-14T07:45:08+09:00 INFO   sqlite3 write success
2022-05-14T07:45:09+09:00 INFO   updated credential success. expired at 2022-05-13T23:00:09Z
2022-05-14T07:45:09+09:00 INFO   get credential main expired at 2022-05-13T23:00:09Z
2022-05-14T07:45:09+09:00 INFO   start restart litestream
2022-05-14T07:45:09+09:00 INFO   restart sucess litestream
(中略)
2022-05-14T07:55:13+09:00 INFO   start update credential
2022-05-14T07:55:13+09:00 INFO   now: 2022-05-14T07:55:13+09:00
2022-05-14T07:55:13+09:00 INFO   sqlite3 write success
2022-05-14T07:55:13+09:00 INFO   UpdateAt: 2022-05-13T22:55:09Z
2022-05-14T07:55:13+09:00 INFO   ExpiredAt: 2022-05-13T23:00:09Z
2022-05-14T07:55:14+09:00 INFO   updated credential success. expired at 2022-05-13T23:10:14Z
2022-05-14T07:55:14+09:00 INFO   get credential main expired at 2022-05-13T23:10:14Z
2022-05-14T07:55:14+09:00 INFO   start restart litestream
2022-05-14T07:55:14+09:00 INFO   restart sucess litestream
(中略)
2022-05-14T08:05:18+09:00 INFO   start update credential
2022-05-14T08:05:18+09:00 INFO   now: 2022-05-14T08:05:18+09:00
2022-05-14T08:05:18+09:00 INFO   sqlite3 write success
2022-05-14T08:05:18+09:00 INFO   UpdateAt: 2022-05-13T23:05:14Z
2022-05-14T08:05:18+09:00 INFO   ExpiredAt: 2022-05-13T23:10:14Z
2022-05-14T08:05:19+09:00 INFO   updated credential success. expired at 2022-05-13T23:20:19Z
2022-05-14T08:05:19+09:00 INFO   get credential main expired at 2022-05-13T23:20:19Z
2022-05-14T08:05:19+09:00 INFO   start restart litestream
2022-05-14T08:05:19+09:00 INFO   restart sucess litestream
(中略)
2022-05-14T08:15:23+09:00 INFO   start update credential
2022-05-14T08:15:23+09:00 INFO   now: 2022-05-14T08:15:23+09:00
2022-05-14T08:15:23+09:00 INFO   sqlite3 write success
2022-05-14T08:15:23+09:00 INFO   UpdateAt: 2022-05-13T23:15:19Z
2022-05-14T08:15:23+09:00 INFO   ExpiredAt: 2022-05-13T23:20:19Z
2022-05-14T08:15:24+09:00 INFO   updated credential success. expired at 2022-05-13T23:30:24Z
2022-05-14T08:15:24+09:00 INFO   get credential main expired at 2022-05-13T23:30:24Z
2022-05-14T08:15:24+09:00 INFO   start restart litestream
2022-05-14T08:15:24+09:00 INFO   restart sucess litestream
(中略)
2022-05-14T08:25:28+09:00 INFO   start update credential
2022-05-14T08:25:28+09:00 INFO   now: 2022-05-14T08:25:28+09:00
2022-05-14T08:25:28+09:00 INFO   sqlite3 write success
2022-05-14T08:25:28+09:00 INFO   UpdateAt: 2022-05-13T23:25:24Z
2022-05-14T08:25:28+09:00 INFO   ExpiredAt: 2022-05-13T23:30:24Z
2022-05-14T08:25:29+09:00 INFO   updated credential success. expired at 2022-05-13T23:40:29Z
2022-05-14T08:25:29+09:00 INFO   get credential main expired at 2022-05-13T23:40:29Z
2022-05-14T08:25:29+09:00 INFO   start restart litestream
2022-05-14T08:25:29+09:00 INFO   restart sucess litestream
(中略)
2022-05-14T08:35:33+09:00 INFO   start update credential
2022-05-14T08:35:33+09:00 INFO   sqlite3 write start
2022-05-14T08:35:33+09:00 INFO   now: 2022-05-14T08:35:33+09:00
2022-05-14T08:35:33+09:00 INFO   UpdateAt: 2022-05-13T23:35:29Z
2022-05-14T08:35:33+09:00 INFO   sqlite3 write success
2022-05-14T08:35:33+09:00 INFO   ExpiredAt: 2022-05-13T23:40:29Z
2022-05-14T08:35:34+09:00 INFO   updated credential success. expired at 2022-05-13T23:50:34Z
2022-05-14T08:35:34+09:00 INFO   get credential main expired at 2022-05-13T23:50:34Z
2022-05-14T08:35:34+09:00 INFO   start restart litestream
2022-05-14T08:35:34+09:00 INFO   restart sucess litestream
(中略)
2022-05-14T08:45:38+09:00 INFO   start update credential
2022-05-14T08:45:38+09:00 INFO   sqlite3 write start
2022-05-14T08:45:38+09:00 INFO   now: 2022-05-14T08:45:38+09:00
2022-05-14T08:45:38+09:00 INFO   UpdateAt: 2022-05-13T23:45:34Z
2022-05-14T08:45:38+09:00 INFO   ExpiredAt: 2022-05-13T23:50:34Z
2022-05-14T08:45:38+09:00 INFO   sqlite3 write success
2022-05-14T08:45:40+09:00 INFO   updated credential success. expired at 2022-05-14T00:00:40Z
2022-05-14T08:45:40+09:00 INFO   get credential main expired at 2022-05-14T00:00:40Z
2022-05-14T08:45:40+09:00 INFO   start restart litestream
2022-05-14T08:45:40+09:00 INFO   restart sucess litestream
(中略)
2022-05-14T08:49:58+09:00 INFO   sqlite3 write start
2022-05-14T08:49:58+09:00 INFO   sqlite3 write success
signal: terminated 👈 手動停止ログ(kill)
make: *** [Makefile:2: start] エラー 1
```
:::


### レコード数確認

デバイス側のSQLiteのレコード数を確認します。

```bash: デバイス側のレコード数確認
$ sqlite3 test.db
SQLite version 3.27.2 2019-02-25 16:06:06
Enter ".help" for usage hints.
sqlite> select count(*) from HealthCheck;
779
```

ログ上の`sqlite3 write success`の回数も779件なので、一致しています

replicateしたSQLiteをMacにrestoreし、レコード数を確認します。
```bash
# AWS環境にassume-role
$ litestream restore -o restored.db s3://YOUR_S3_NAME/replica
$ ls
restored.db
$ sqlite3 restored.db
SQLite version 3.32.3 2020-06-18 14:16:19
Enter ".help" for usage hints.
sqlite> select count(*) from HealthCheck ;
779
```

一致していますね。

# さいごに

当然ではありますが、期限付きのクレデンシャルでも何事もなく、レプリケートできました。Litestream楽しいですね！
