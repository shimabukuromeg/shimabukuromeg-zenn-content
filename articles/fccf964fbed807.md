---
title: "CDK for TerraformでSpotifyのPlaylistを作ってみるぞ"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "cdktf", "spotify"]
published: true
---

# はじめに

TerraformでSpotifyのPlaylistを作成する記事が面白かったので、同じことをCDK for Terraformで試してみました。

https://learn.hashicorp.com/tutorials/terraform/spotify-playlist

https://github.com/conradludgate/terraform-provider-spotify


# CDK for Terraformとは

- Terraformの定義をHCLではなくプラグラミング言語を使って定義できる（C#、Python、TypeScript、Java、または Go）
- HashiCorp 社と AWS CDK チームが共同開発
- 2022年8月12日からGA

https://aws.amazon.com/jp/blogs/news/cdk-for-terraform-on-aws-jp/

- 公式ドキュメント

https://www.terraform.io/cdktf

# 準備

元ネタの記事にある認証用サーバーを起動する `Run authorization server`のセクションまでは同じように進めます。

https://learn.hashicorp.com/tutorials/terraform/spotify-playlist

- [Prerequisites](https://learn.hashicorp.com/tutorials/terraform/spotify-playlist#prerequisites)
- [Create Spotify developer app](https://learn.hashicorp.com/tutorials/terraform/spotify-playlist#create-spotify-developer-app)
- [Run authorization server](https://learn.hashicorp.com/tutorials/terraform/spotify-playlist#run-authorization-server)

:::details Run authorization server 起動時の補足情報

```bash
$ docker run --rm -it -p 27228:27228 --env-file ./.env ghcr.io/conradludgate/spotify-auth-proxy
APIKey:   fugafugafugafugafugafugafugafugafugafugafugafugafugafugafugafugafugafuga
Auth URL: http://localhost:27228/authorize?token=hogehogehoge
```

- spotify-auth-proxy を起動した後コンソールに表示される`Auth URL`にアクセスします。
- アクセスするとSpotifyへの認可リクエストの処理などが動き、完了したら同じくコンソールに表示されている`APIKey`が使えるようになります。
- TerraformからSpotify API叩く時に必要になるアクセストークンは、この`APIKey`を使って取得できます。
- そのためこのAPIKeyは後ほどCDK for Terraformを動かす時に使います。
- Spotifyのclient IDとシークレットキーは spotify-auth-proxy コンテナを起動する際にenvファイルを指定しますが、そこで使います。

	```
	SPOTIFY_CLIENT_ID=
	SPOTIFY_CLIENT_SECRET=
	```
:::

# CDK for Terraformを動かす

### パッケージのインストールや設定

- CDKTFをインストールする

```bash
$ npm install --global cdktf-cli@latest
```

- 作業するディレクトリ作成

```bash
$ mkdir learn-cdktf-spotify
$ cd learn-cdktf-spotify
```

- 言語のテンプレートをして指定して、CDKTFを初期化（今回はTypeScriptを指定）

```bash
$ cdktf init --template=typescript --local
Note: By supplying '--local' option you have chosen local storage mode for storing the state of your stack.
This means that your Terraform state file will be stored locally on disk in a file 'terraform.<STACK NAME>.tfstate' in the root of your project.
? Project Name learn-cdktf-spotify
? Project Description A simple getting started project for cdktf.
? Do you want to start from an existing Terraform project? No
? Do you want to send crash reports to the CDKTF team? See
https://www.terraform.io/cdktf/create-and-deploy/configuration-file#enable-crash-reporting-for-the-cli for more information Yes
(省略)
```

- Spotifyのプロバイダーを追加する

```bash
$ cdktf provider add conradludgate/spotify
```
- `$ cdktf provider add` を実行すると`cdktf.json` にプロバイダーが追加され、`.gen/providers/{プロバイダ名}/` にSpotifyのリソースを操作するためのTypeScriptのコードが生成されてる

```json
  "terraformProviders": [
    "conradludgate/spotify@~> 0.2"
  ],
```

- awsやgoogleなどよく利用されるプロバイダーは事前にビルドされたパッケージが公開されてるのでそれが利用される


	> You can also use the provider add command to add providers to your CDKTF application. It will automatically try to install a pre-built provider if available and fall back to generating bindings locally if none was found.

https://www.terraform.io/cdktf/concepts/providers


- 事前にビルドされて公開されてるパッケージ一覧

https://github.com/orgs/hashicorp/repositories?q=cdktf-provider-

- `.env` ファイルを作って、`SPOTIFY_API_KEY` を追加する。`SPOTIFY_API_KEY`には、spotify-auth-proxyコンテナ起動した時に出力される `APIKey` の値を指定する

```
$ npm install dotenv
```

```env
SPOTIFY_API_KEY=
```

### リソース定義

- `main.ts` にSpotifyのリソースの定義を書いていきます。

```typescript
import { Construct } from "constructs";
import { App, TerraformStack, TerraformOutput, Token } from "cdktf";
import { SpotifyProvider, Playlist, DataSpotifySearchTrack } from "./.gen/providers/spotify";
import * as dotenv from 'dotenv'

dotenv.config()
class MyStack extends TerraformStack {
  constructor(scope: Construct, name: string) {
    super(scope, name);

    // アーティスト名
    const artists = [
      "Awich",
      "柊人",
      "BASI",
      "SALU",
      "BAD HOP",
      "PUNPEE",
      "VaVa",
      "BIM",
      "imase",
      "KEIJU",
      "舐達麻",
      "SPARTA",
      "604",
      "Daichi Yamamoto",
      "KID FRESINO",
      "OMSB",
      "kZm"
    ]

    // データソース
    // https://www.terraform.io/cdktf/concepts/data-sources#define-data-sources
    const tracks = artists.map((artist, index) => {
      const dataSpotifySearchTrack = new DataSpotifySearchTrack(this, `searchTrack_${index}`, {
        artist: artist,
      })
      return [...Array(4).keys()].map(n => dataSpotifySearchTrack.tracks.get(n).id)
    }).flat()

    // プロバイダー
    // https://www.terraform.io/cdktf/concepts/providers
    new SpotifyProvider(this, "spotify", {
      apiKey: Token.asString(process.env.SPOTIFY_API_KEY),
    });

    // リソース作成（プレイリスト）
    const spotify_playlist = new Playlist(this, "playlist", {
      name: "My Playlist CDKTF",
      description: "This playlist was created by CDKTF",
      public: true,
      tracks,
    });

    // 出力
    // https://www.terraform.io/cdktf/concepts/variables-and-outputs
    new TerraformOutput(this, "playlist url", {
      value: `https://open.spotify.com/playlist/${spotify_playlist.id}`,
    });
  }
}

const app = new App();
new MyStack(app, "learn-cdktf-spotify");
app.synth();
```

https://www.terraform.io/cdktf/concepts/data-sources#define-data-sources

https://www.terraform.io/cdktf/concepts/variables-and-outputs#when-to-use-output-values

### デプロイする

- デプロイするコマンド実行する

```bash
$ cdktf deploy

（省略）

  learn-cdktf-spotify
  playlist url = https://open.spotify.com/playlist/3MMsv6oBkim3jYdEe69WXz
```

- プレイリストのURLにアクセスする

![](https://storage.googleapis.com/zenn-user-upload/5ab7d743168a-20220918.png)

https://open.spotify.com/playlist/3MMsv6oBkim3jYdEe69WXz

ちゃんと作られていた 🎉

ここまで作ったコード

https://github.com/shimabukuromeg/learn-cdktf-spotify

# SpotifyのリソースをTerraformで管理する仕組み

TerraformはProviderと呼ばれるプラグインにより、AWS、Azure、GCP など様々なリソースを管理することができます。
クラウドコンピューティングのリソース以外でも、カスタムプロバイダーを作成し、プロバイダー経由でAPIを呼び出すようにすることで各サービスのリソースを管理できます。
Spotifyも提供されてるAPIを[プロバイダー](https://github.com/conradludgate/terraform-provider-spotify
)経由でアクセスすることで管理することができます。

- ドキュメントからわかりやすい図

![](https://mktg-content-api-hashicorp.vercel.app/api/assets?product=tutorials&version=main&asset=public%2Fimg%2Fterraform%2Fproviders%2Fcore-plugins-api.png)

- Providerのドキュメント

https://www.terraform.io/language/providers

- Providerのドキュメント（CDKTF）

https://www.terraform.io/cdktf/concepts/providers

- カスタムプロバイダーを作るハンズオン

https://learn.hashicorp.com/collections/terraform/providers

https://tatsuo48.me/lets-create-terraform-provider/

https://budougumi0617.github.io/2020/11/17/unittest_for_terraform_custom_provider/


# おわりに

- CDK for TerraformでSpotifyのプレイリストを作成することができた
- HCLより馴染みのプログラミング言語でリソース定義できるのわかりやすい
- カスタムプロバイダーを作ると外部のAPIのリソースをTerraformで管理できること知らなかったので勉強になった
- カスタムプロバイダーを作る[ハンズオン](https://learn.hashicorp.com/collections/terraform/providers)の資料が面白そうだったので今度試してみたい


# 参考

https://learn.hashicorp.com/tutorials/terraform/cdktf-install?in=terraform/cdktf

https://zenn.dev/kou_pg_0131/articles/cdk-for-terraform

https://zenn.dev/mayforblue/articles/09574f95fdbf69