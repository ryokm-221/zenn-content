---
title: "CloudFrontでURLにインデックスを自動で付ける"
emoji: "📲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "cloudfront"]
published: true
---
こんにちは，[@ry_km](https://twitter.com/ry_km_u_u)です。Zenn2記事目です。

みなさんAWSのS3+CloudFront使ってますか？
簡単に低コストでホームページの運用ができるサービスで，[SPA](https://ja.wikipedia.org/wiki/%E3%82%B7%E3%83%B3%E3%82%B0%E3%83%AB%E3%83%9A%E3%83%BC%E3%82%B8%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3)(Single-Page Application)をホスティングするのに大変重宝しています。

ただ初期設定こそ簡単ですが，運用中につまづくポイントが多々あります。
今回は，その中でもあまり知られていないインデックスの設定方法を紹介します。

# S3+CloudFrontのWebホスティングとは？
S3はAWSの代表的なストレージサービス，CloudFrontは同じくAWS謹製のCDNです。S3には[静的Webホスティング](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/WebsiteHosting.html)という機能があり，サーバーサイドプログラムを必要としないWebページ（PHPやASP.NETなどを使わないページ）のホストが簡単にできます。
S3のバケットはアクセス権（ポリシー）を自由に設定することができるので，「作成者は変更・読み取り許可，その他は読み取り専用」とポリシーを設定することで静的ホスティングが可能です。

## メリット
- ApacheやNginxなどの細かい設定をする必要がない
- サーバーレスでの運用が可能（マネージドサービスなので利用料が安い）
- データの耐久性が非常に高い

## CloudFrontとの組み合わせ
また，CloudFrontと組み合わせることで以下のことができるようになります。
- HTTPS化（ACLで証明書を発行する必要あり）
- キャッシュを取って読み込みを高速化
- S3バケットに直接アクセスさせずにホスティングが可能（CloudFrontを使って間接的にバケットにアクセスする）なため，セキュリティー的に安心

最近はhttpsでないと[SEOのランキングが下がる](https://developers.google.com/search/blog/2014/08/https-as-ranking-signal)らしいので，ホームページのHTTPS化は必須ですね。したがって，基本的にS3とCloudFrontは組み合わせて使う必要があります。

# インデックスの設定について
一方で大抵のレンタルサーバーサービスにあって，S3+CloudFrontにはない機能があります。その一つが「インデックスの指定」です。
例えば，お名前.comのレンタルサーバーには次のような記載があります。

![お名前.comレンタルサーバーのインデックス優先順位](/images/cloudfront-function-rewrite-uri/onamae-server-index.png)
*お名前.comレンタルサーバーのインデックス優先順位。出典：[https://help.onamae.com/answer/20292](https://help.onamae.com/answer/20292)*

このように，レンタルサーバーではデフォルトでindex.htmlを参照するように設定されているわけですね（Apacheの内部設定で可能）。つまり，`example.com`にアクセスすると自動的に`example.com/index.html`にリダイレクトされるようになっています。
しかしながら，マニュアル運転のAWS^[AWSは自由が利く分マニュアル運転に近いと感じています。例えば，[サブネットのプライべート・パブリックの設定](https://blog.serverworks.co.jp/tech/2013/05/23/vpc_beginner-2/)一つとってもそうです。ちなみに筆者は若者には珍しく(?)マニュアルで免許を取りました。]ではそうもいきません。S3+CloudFrontでホスティングしている場合，インデックスの設定に関して以下の制限があります。
- バケットのルートにアクセスしている場合，末尾の`index.html`は省略可能（[設定方法](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/DefaultRootObject.html)）
- バケットのルート以外にアクセスしている場合，末尾の`index.html`は省略不可

したがって，以下のようなアクセスでは403エラーが出ます。
- アクセス先：`s3://example-bucket/hoge/index.html`
- `example.com`に紐づけたバケット：`s3://example-bucket/`
- アクセスしたリンク：`example.com/hoge/`

`wget`コマンドを使うとこのようになります。
![デフォルトの設定でインデックスを省略した場合](/images/cloudfront-function-rewrite-uri/403-forbidden.png)
*デフォルトの設定でインデックスを省略した場合。`/api`にはindex.htmlが入れてあります。*

しかしこれが使いにくい...いちいちindex.htmlまで打たないといけないので，かなりメンドクサイです！
これを解決するために，CloudFront Functionsを使ってリクエストのパスを書き換えるということをします。

# CloudFront Functionsとは？
[CloudFront Functions](https://aws.amazon.com/jp/blogs/news/introducing-cloudfront-functions-run-your-code-at-the-edge-with-low-latency-at-any-scale/)（以下CF2と書きます）は，2021/5/6にローンチされた比較的新しいサービスです。

>カスタマイズされたエクスペリエンスを可能な限り最小のレイテンシーで提供するために、今日の多くのアプリケーションはエッジで何らかの形式のロジックを実行します。
>
>この(...)ユースケースを支援するために、218 以上の CloudFront エッジロケーションで軽量の JavaScript コードを Lambda@Edge の 1/6 のコストで実行できる新しいサーバーレススクリプトプラットフォームである CloudFront Functions の提供が開始されました。

（引用：[CloudFront Functions の導入 – 任意の規模において低レイテンシーでコードをエッジで実行](https://aws.amazon.com/jp/blogs/news/introducing-cloudfront-functions-run-your-code-at-the-edge-with-low-latency-at-any-scale/)）

似たようなサービスにLambda@Edgeがありますが，CF2には以下の特徴があります。
- ランタイム：JavaScriptのみ（ECMAScript5.1準拠）
- 実行場所：218のCloudFrontエッジロケーション^[CloudFrontには計13のエッジリージョンと計300以上のエッジロケーションがあります。[凄まじいですね](https://aws.amazon.com/jp/cloudfront/features/?whats-new-cloudfront.sort-by=item.additionalFields.postDateTime&whats-new-cloudfront.sort-order=desc)。]
- 実行時間：1msまで
- 使用可能メモリ：2MBまで
- パッケージサイズ：10KBまで
- ビューアリクエスト・レスポンスのみペイロードとして与えられる
- VPC・ファイルシステムへのアクセスは不可

エッジロケーションで実行できるというのが激アツです！1ms以内という制限があるので，使用用途はパスの書き換えやリダイレクトなどかなり絞られると思います。
注意が必要なのは，JSとはいってもECMAScript5.1準拠という点ですね。`let`や`const`はECMAScript6 (2015)で導入されたので，`var`しか使用できません^[最初それを知らないで`const`を使って書いたら怒られました。]。アロー関数（`const hoge = () => {}`）も使えません。

# 実際にやってみる
使うコードは以下の通りです。
```js
function handler(event) {
    var request = event.request
    var uri = request.uri;
    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    } else if (!uri.includes('.')) {
        request.uri += '/index.html';
    }
    return request;
}
```

今回は[AWS公式のコード](https://github.com/aws-samples/amazon-cloudfront-functions/tree/main/url-rewrite-single-page-apps)をお借りしました。これをCF2に設定していきます。

## 1. CloudFront Functionsを開く
CloudFrontのサイドバーにある関数を開きます
![CloudFrontのサイドバー](/images/cloudfront-function-rewrite-uri/cf2-menu.png)

## 2. 関数を作成
関数を作成します。名前と説明を入力します。説明はいつも通り省略可です。
![CloudFront Functionの作成](/images/cloudfront-function-rewrite-uri/cf2-create-function.png)

上のコードをコピー・ペーストします。
![CloudFront Functionコードの入力](/images/cloudfront-function-rewrite-uri/cf2-create-function-package.png)

## 3. 発行・関連付け
関数を発行します。発行タブに移動して，関数を発行をクリックします。
![CloudFront Functionの発行](/images/cloudfront-function-rewrite-uri/cf2-publish-function.png)

その後，ディストリビューションと関連付けを設定します。
![CloudFront Functionの関連付け1](/images/cloudfront-function-rewrite-uri/cf2-publish-function-distribution1.png)
![CloudFront Functionの関連付け2](/images/cloudfront-function-rewrite-uri/cf2-publish-function-distribution2.png)

これで完成です！操作自体はとても簡単ですね。

# まとめ
更新したディストリビューションはすぐに反映されます。最後に`wget`コマンドで確認してみましょう。
![wgetコマンドの結果](/images/cloudfront-function-rewrite-uri/200-1.png)

ディレクトリ名でリクエストしてもきちんと`index.html`にアクセスすることができました。
S3+CloudFrontでホスティングするときにぜひ使ってみてください！
