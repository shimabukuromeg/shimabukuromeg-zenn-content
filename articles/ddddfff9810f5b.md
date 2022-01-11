---
title: "はじめてのCognito"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "cognito"]
published: false
---

最近になってはじめてCognitoを使う機会があったので、調べたことをまとめました。


# Cognitoとは

Cognitoは認証やユーザー管理の仕組みを提供するAWSのサービスです。詳しくはドキュメントです。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/what-is-amazon-cognito.html

# 調べていてつまずいたこと

振り返るとそんなに難しいことではない気もしますが、初見だと以下のようなことを疑問を持ちました。

:::details Cognitoを調べるとAmplifyとセットになって紹介してる記事がたくさんあり、Amplifyとは？AmplifyとCognitoってどういう関係？

- Amplifyは、アプリケーションを作るために必要なサービス群（たとえばホスティングや認証やストレージ、バックエンドのAPIなど）をいい感じにまとめて提供してくれるAWSの仕組みです。その仕組みの中で、認証だとCognitoが使われているので、Cognitoを調べるとAmplifyの情報もたくさん出てきた感じでした。Amplifyと合わせてCognitoを使った場合も、Cognito単独で使った場合も、Cognito自体の認証の仕組みは変わらないです。
:::

:::details ユーザープールとは？Cognitoの知識だったり、対応してるユースケースで今回はどのケース？

- ユーザープールは、ユーザーを管理する仕組みです。ユーザー登録したり、ログインするときに使います。
- 概要を理解するにあたって、図が多くてこの資料がわかりやすかったです。
  - https://d1.awsstatic.com/webinars/jp/pdf/services/20200630_AWS_BlackBelt_Amazon%20Cognito.pdf

:::

:::details フロントからCognitoのリソースを操作したいときに、使うライブラリ選定どんな感じがいいんだろう？next-auth？javascript用のsdk？

- 最初よくわかってなくて、next-auth使えばいい感じにcognitoと連携できる？みたいに思っていましたが、今回のケースだとnext-authは使えませんでした。（Next.jsの構成が、バックエンドにnodeの環境はなく、SSGして配信するためパターンで、Next.jsのapi routeの機能が使えなくて利用できなそうでした）
- next-authのドキュメント
  - https://next-auth.js.org/providers/cognito
- JavaScriptのライブラリで、aws-amplifyというのがあり、これを使ってCognitoへのリソースの操作をするのが良さそうだった。（名前がamplifyとなっていたので、amplify使うときに使うライブラリと勘違いしましたが、Cognito単独で使う場合も利用可能です）

:::

:::details 独自のUIを作るパターンと、UIライブラリを使ったパターンがある？

- 今回は使いませんでしたが、提供されているUIライブラリを使う場合は、@aws-amplify/ui-reactを導入することでいい感じにやってくれるそうでした。
- UIライブラリを使うパターンの記事と独自UIを使うパターンの記事があり、初見だとどちらを参考にすればいいか迷いました。今回は独自UIだったので、UIライブラリを使っているパターンはスルーしました。

:::

# やりたかったこと

ドキュメントに、Cognitoを使用する際のシナリオが記載されています。このシナリオの中の、

- ユーザープールを使用して認証する
- ユーザープールを使用してサーバー側のリソースにアクセスする

が今回やりたかったことでした。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-scenarios.html

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/amazon-cognito-user-pools-authentication-flow.html

今回の具体的な構成については、バックエンドにLaravel、フロントエンドにNext.jsを使っています。Next.jsはSSGでHTMLを生成し、Amplifyで配信する構成です。Next.jsにaws-amplify（CognitoなどのAWSのリソースを扱えるライブラリ）を導入し、フロントからはこのライブラリを使ってCognitoのAPIを操作します。Cognitoで認証が済んだ後、Cognitoから受け取ったトークンを使ってLaravelのAPIにアクセスします。

aws-amplifyにはAuthのオブジェクトがあり、`signUp`、`signIn`、`currentSession` などの関数が提供されているので、これらを使って認証の処理を行います。LaravelのAPIにリクエストを投げる際は、Cognitoから取得したトークンをheaderのAuthorizationに値を入れてリクエストを投げる、という流れです。UIに関しては、提供されているUIライブラリではなく、独自のUIを作成しました。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/scenario-backend.html

aws-amplify。Cognitoなどのリソースを操作するJavaScriptのライブラリ。Cognito使うだけでも、このライブラリが使えます。

https://github.com/aws-amplify/amplify-js

# AWS CLIを使ってのCognitoのリソース操作

フロントを構築する前に、さくっとCognitoのAPIを試してみたかったので、CLIを使って挙動の確認をしました。以下のようなコマンドが使えます。


```bash
// 例. サインアップ
$ aws cognito-idp sign-up --client-id [クライアントID] --username "[ユーザー名]" --password "[パスワード]" --validation-data '[バリデーションデータ]'

// 例. サインアップ後の確認
aws cognito-idp admin-confirm-sign-up --user-pool-id [ユーザープールID] --username [ユーザー名]

// 例. 認証フロー開始
aws cognito-idp admin-initiate-auth --user-pool-id [ユーザープールID] --client-id [クライアントID] --auth-flow ADMIN_USER_PASSWORD_AUTH --auth-parameters "USERNAME=[ユーザー名],PASSWORD=[パスワード]"
```

詳しくは、CLIのリファレンスです。

https://docs.aws.amazon.com/cli/latest/reference/cognito-idp/index.html#cli-aws-cognito-idp

# aws-amplifyを使ってのCognitoのリソース操作

aws-amplifyの使い方は、ドキュメントが参考になったのと、Authオブジェクトが持っているメソッドをながめて、どういうAPIが用意されてるのか確認しました。

詳しくは、aws-amplify のドキュメントです。

https://docs.amplify.aws/lib/auth/emailpassword/q/platform/js/

https://docs.amplify.aws/lib/auth/manageusers/q/platform/js/

こちらが、Authのコードです。

https://github.com/aws-amplify/amplify-js/blob/main/packages/auth/src/Auth.ts

他にもGitHubでCognitoとNext.jsが使われているコードを探して参考にしました。こちらが参考になりました。

https://github.com/DaviBrancol/nextjs-cognito-auth

# ユーザー作成後の承認処理はLambdaで対応

Cognito上にユーザーを作成できたら、Cognitoに登録したユーザーのステータスを確認済みにする必要がありますが、今回のケースではSignUp（新規登録）のUIの都合上、Email／TELによる承認を行う流れがありませんでした。そのため、ユーザー作成後の承認処理をEmail／TELによる承認以外で対応しなければいけませんでした。確認済みのステータスに変更する他の方法としては、管理者による承認とLambdaでPreSignupイベントをトリガーとして承認するパターンがあり、今回はLambdaで対応しました。（SignUpした際に、確認コードによる承認を不要にできないかと調べてみたのですが、できなそうでした。）

以下の記事が大変参考になりました。

https://qiita.com/aws-warrior/items/58e317960cede9a23086

ユーザーアカウント確認の流れはこちらのドキュメントに記載されています。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/signing-up-users-in-your-app.html

Amplify Authメソッドで、承認コードなしでSingUpできないか探しているときに見つけたIssueです。管理系のメソッドをAuthでも使えるようにしてほしい、てきなIssueでしたが、管理系のメソッドは、ユースケース的にユーザー利用側のメソッドには必要ないっていうissueになっていました。

https://github.com/aws-amplify/amplify-js/issues/2200

ステータスについて、具体的にはコンソールから見える確認ステータスのことです。

![](https://storage.googleapis.com/zenn-user-upload/e331edd795b2-20220109.png)

# aws-amplify のAuthでリフレッシュトークンしてる処理を追ってみた

フロントからaws-amplifyを使ってCognitoへの認証を行うときに、トークンをリフレッシュする処理を書く必要があるのか、もしくはライブラリ側で実装してくれてるのか気になったので、処理を追ってみました。


constructorでconfigureが呼ばれてる

https://github.com/aws-amplify/amplify-js/blob/main/packages/auth/src/Auth.ts#L117

configureの中で CognitoUserPool がnewするときに、wrapRefreshSessionCallbackがわたされてる

https://github.com/aws-amplify/amplify-js/blob/main/packages/auth/src/Auth.ts#L196

CognitoUserPool はamazon-cognito-identity-jsから呼ばれてる

https://github.com/aws-amplify/amplify-js/blob/main/packages/auth/src/Auth.ts#L50

CognitoUserPoolはCognitoUserがnewするときに呼ばれてる

https://github.com/aws-amplify/amplify-js/blob/main/packages/amazon-cognito-identity-js/src/CognitoUser.js#L82

this.pool.wrapRefreshSessionCallbackは、refreshSessionの中で呼ばれてる

https://github.com/aws-amplify/amplify-js/blob/main/packages/amazon-cognito-identity-js/src/CognitoUser.js#L1458

refreshSessionは、refreshSessionIfPossibleの中で呼ばれてる

https://github.com/aws-amplify/amplify-js/blob/main/packages/amazon-cognito-identity-js/src/CognitoUser.js#L1229

refreshSessionIfPossibleは、getUserDataの中で呼ばれてる

https://github.com/aws-amplify/amplify-js/blob/6882c5e6e8f1bff2206ff0de74cebbcf87efd622/packages/amazon-cognito-identity-js/src/CognitoUser.js#L1253

getUserData 自体は、Auth.tsの中でいろんなんところで使われてそう。たとえば、currentUserPoolUserの中とか

https://github.com/aws-amplify/amplify-js/blob/6882c5e6e8f1bff2206ff0de74cebbcf87efd622/packages/auth/src/Auth.ts#L1305

currentUserPoolUserは、currentSessionの中とかで呼ばれてる。そのため、currentSessionなどgetUserDataを使ってる処理を実行したら、ライブラリ側で必要に応じて自動でトークンがリフレッシュされてそうでした。

https://github.com/aws-amplify/amplify-js/blob/6882c5e6e8f1bff2206ff0de74cebbcf87efd622/packages/auth/src/Auth.ts#L1432

上記、コード追ってみましたが、この辺りについてドキュメントにも記載があって、以下のように書いていました。`Auth.currentSession()` を呼び出したら自動的にtokenもリフレッシュされるようです。

> This method will automatically refresh the `accessToken` and `idToken` if tokens are expired and a valid `refreshToken` presented. So you can use this method to refresh the session if needed.**`12345`**

https://docs.amplify.aws/lib/auth/manageusers/q/platform/js/#retrieve-current-session

# 参考情報

Cognitoの概要的な記事です。

https://dev.classmethod.jp/articles/re-introduction-2020-amazon-cognito/

もともとは、amazon-cognito-identity-js だったけど、amplify-jsに統合されたってのを知った記事です。

https://medium.com/@noid11/%E3%81%84%E3%81%BE%E3%81%95%E3%82%89-amazon-cognito-identity-sdk-for-javascript-amazon-cognito-identity-js-%E3%82%92%E4%BD%BF%E3%81%86%E6%96%B9%E6%B3%95-801957f40572

つまづきポイント多すぎのところで同じようなことを思ったので、すごく参考になった。今回調べてた中で一番助かった記事でした。

https://zenn.dev/dove/articles/63494de652511c

Cognito試してみた後の消し方チュートリアルです。

https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/tutorial-cleanup-tutorial.html

Cognitoの概要を理解するために調べてた見つけた資料で、図がたくさんあってわかりやすかったです。

https://d1.awsstatic.com/webinars/jp/pdf/services/20200630_AWS_BlackBelt_Amazon%20Cognito.pdf

今回Laravel側は実装しませんでしたが、バックエンドはこの記事がどうなってるかはこの記事が参考になりそうでした。

https://qiita.com/ggg-mzkr/items/25abba8d490b054fb00f

今回の記事とあんまり関係ありませんが、amplifyの推しポイントが書かれたこの記事も勉強になりました。

https://qiita.com/toto_inu/items/77e31e92f908a1fda8f7