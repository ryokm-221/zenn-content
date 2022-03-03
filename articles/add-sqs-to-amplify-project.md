---
title: "AmplifyプロジェクトにSQSキューを追加する"
emoji: "☕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "amplify", "lambda", "sqs"]
published: true
---
こんにちは，[@ry_km](https://twitter.com/ry_km_u_u)です。

遅ればせながら，ここ3ヶ月くらいで[AWS Amplify](https://aws.amazon.com/jp/amplify/)の便利さに気づき，新しいプロジェクトの開発はほぼすべてAmplifyで行っている状況です。
先日いつも通り開発していたところ，Amplifyで構築したバックエンドにSQSキューを使う必要があることに気づき，どうしようかと悩みました。この記事では，Amplify + SQS^[この記事の絵文字はコーヒーですが，その理由はQueueに注文を受けて作る人に渡すカフェのイメージがあるからです。深い意味はありません。] (+ Lambda)で構築を行う方法をまとめます。

# そもそもAmplifyとは？（復習）
AWSの公式サイトには以下のように紹介されています。
> AWS Amplify は、フロントエンドのウェブ/モバイルデベロッパーが AWS でフルスタックアプリケーションをすばやく簡単に構築できるようにする専用のツールと一連の機能であり、(...)幅広い AWS サービスを活用できる柔軟性を備えています。(...)
> クラウドの専門知識がなくても、より速く、簡単に拡張できます。

（引用：[AWS Amplify](https://aws.amazon.com/jp/amplify/)，一部省略）

つまり，多岐にわたるAWSのリソースを簡単につなぎ合わせて，フロントエンジニアでも簡単にバックエンドを制御できるようにした構築環境，といったようなところでしょうか。以下のようなリソースが超簡単に追加できます。
- 認証（= Cognito）
- API (GraphQL = APIGateway + DynamoDB, REST = APIGateway)
- ストレージ (= S3, DynamoDB)
- 関数 (= Lambda)

おもしろいところだと，[Sumerian](https://aws.amazon.com/jp/sumerian/)で作ったXRシーンも組み込めるようです。全く用途はわかりませんが，個人的に使ってみたいです。他にも種類があるので，詳しくは[Amplify公式ドキュメント](https://docs.amplify.aws/lib/q/platform/js/)をご確認ください。

以上のドキュメントに乗っているリソースは，Amplify CLIを使って追加できます。
```bash
$ amplify add <リソースの種類>
```

例えばリソースの種類として認証`auth`とすると，認証方法やSAML認証を使うかなどをCLI上で聞いてきます。その後，CloudFormation（以下CFn）が動いて，Cognitoのユーザープールなどを作ってくれます。Amplifyを使わない場合，全部自分で設定する必要があるのでかなり面倒ですが，Amplify CLIを使うとほんの1分ほどでできてしまいます。

さらに，Amplifyには以下の言語・環境に対応した[Client Library](https://docs.amplify.aws/lib/q/platform/js/)があり，追加したリソースの制御も簡単です。
- JavaScript^[残念ながら，TypeScriptへの対応はいまいちです。戻り値の型が`any`で設定されているものが多く，型の設定を自分でやる必要があります。この辺解決策を知っている方いらっしゃいましたらコメントで教えてください...]
- iOS
- Android
- Flutter

またまた認証を例にとると，ログインは次の4行で済みます。
```js
import { Auth } from 'aws-amplify';
const signIn = async () => {
  await Auth.signIn(username, password);
};
```

ここでは書きませんが，AWS SDKを使って書くのと比べると比にならないくらい楽です。フェデレーションIDなどの機密情報も裏で上手く保持してくれているので，作る側は気にする必要がありません！
また，他のリソース同士を関連付けられるので，「ログインユーザーのみAPIの呼び出しができるようにしたい」，「Lambda関数からS3を参照したい」といった要件にもIAMの設定なしで対応します。

## でも，対応しているリソースこれしかないの？
という疑問が湧くと思います。実際，AWSには200を超えるサービスがあり^[出典：[AWSのクラウドが選ばれる10の理由](https://aws.amazon.com/jp/aws-ten-reasons/)]日々増えています。しかしながら，前出のドキュメントには14個^[1つのリソースの種類で2つ以上のAWSサービスを使う場合があるので，単純に14個しか選択肢がないというわけではありません。]ほどの種類しかありません。これでは，いくらWeb・モバイル開発に特化しているとはいえ少なすぎます。

そこで，Amplifyは前述の「よく使われるリソース」以外を使うためのカスタムリソースという選択肢があります。

# Amplifyのカスタムリソース
前述のリソースでは，設定項目をCLI上で聞いて，それに基づいてCFnを動かすという設定方法でした。カスタムリソースでは，CFnのテンプレートを自分で用意することで同様のリソース管理を実現しています。詳しくは[公式ドキュメント](https://docs.amplify.aws/cli/custom/cloudformation/)をご確認ください。

とはいっても，当然ながら前述のClient Libraryに便利なコードは含まれていません。実際の使い方として，Lambda関数を用意してそこから管理するやり方があります^[当然ほかの使い方もあると思います。コメントでぜひ教えてください！]。フロントからはREST API経由でLambdaを呼び出してカスタムリソースを使うことができます。

以下では，このカスタムリソースを使ってSQS + Lambdaの王道パターンをAmplify上で構成してみます！

# 実際にやってみる
前置きがかなり長くなりましたが，とにかくやってみましょう。前提条件として，以下は設定済みであるとします。
- Amplify CLIはインストール済み（`npm -g amplify`）
- Amplifyは初期設定済み（`amplify init`）

また，以下の記事を参考にしています。
https://medium.com/@navvabian/how-to-add-an-sqs-queue-to-your-amplify-cli-bootstrapped-project-cb7781c636ed

## カスタムリソースを作る
それでは，まずはカスタムリソースを作ります。
```bash
$ amplify add custom
? How do you want to define this custom resource? …  (Use arrow keys or type to filter)
  AWS CDK
❯ AWS CloudFormation
```

「CloudFormationとCDKのどちらを使って構成するか」と聞かれるので，個人的に慣れているCFnを選びます。テンプレートファイルは，`amplify/backend/custom/<リソース名>/<リソース名>-cloudformation-template.json`に用意されます^[テンプレートはYAML形式ではなくJSON形式で用意されるので少し書きにくいです...]。`aws sqs cloudformation`などとGoogleで検索して，公式のリファレンスを見ながら設定しましょう。

最終的に，以下のテンプレートができました。
```json:<SQSリソース名>-cloudformation-template.json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Parameters": {
    "env": {
      "Type": "String"
    }
  },
  "Resources": {
    "myQueue": {
      "Type": "AWS::SQS::Queue",
      "Properties": {
        "QueueName": {
          "Fn::Join": ["", ["my-queue-", { "Ref": "env" }]]
        }
      }
    }
  },
  "Outputs": {
    "QueueURL": {
      "Description": "Queue URL",
      "Value": {
        "Ref": "myQueue"
      }
    },
    "QueueArn": {
      "Description": "Queue ARN",
      "Value": {
        "Fn::GetAtt": ["myQueue", "Arn"]
      }
    }
  }
}
```

`Outputs`セクションにQueue URLとARNを出しておくのが大切です。後ほど使います！
この時点で一旦`amplify push`をして，リソースが正しく作成されるかを確認します。SQSのページに行って，キューができていれば完了です。

## Lambda関数と連携させる
ここが一番重要でかつ一番悩むところでした。Amplifyはステージという考え方でリソースを管理していて，開発用ステージ・本番用ステージなど段階を踏むことができ，ステージごとに異なるリソースを作成します。したがって，QueueURLがステージによって変化するため，ハードコードしたり環境変数にぶち込むことはできません。

ここで，例えばストレージを追加してLambda関数と結びつけると，バケット名を環境変数に入れることができますが，ステージ毎にバケット名を自動的に変えて環境変数に入れてくれます。同じことをSQSにも設定してみます。

肝心のLambda関数を設定します。
```bash
$ amplify add function
? Provide a friendly name for your resource to be used as a label for this category in the project: lambdafunction
? Provide the AWS Lambda function name: lambdafunction
? Choose the function template that you want to use: (Use arrow keys)
❯ Hello world function
  CRUD function for Amazon DynamoDB table (Integration with Amazon API Gateway and Amazon DynamoDB)
  Serverless express function (Integration with Amazon API Gateway)
```

Lambda関数の設定ファイルは`amplify/backend/function/<リソース名>/`以下にあります。nodejsランタイムで設定した場合，エントリーポイントは`src/index.js`です^[src内ではnpmを使えるので，必要なパッケージをインストールすることができます。]。
SQSを使う場合，例えば以下のようなコードを書くことが多いと思います。
```js:index.js
const AWS = require('aws-sdk');
const sqs = new AWS.SQS({ region: 'ap-northeast-1' });

exports.handler = async (event) => {
  // ...
  try {
    await sqs.sendMessage({
      MessageBody: 'text',
      QueueUrl: process.env['Queue']
    }).promise();
  } catch (e) {
    console.error(e);
  }
};
```

QueueURLは`Queue`という環境変数に入れることにしています^[このコードはもともとTypeScriptで書いていたものをCommonJSに書き直しています。TSでは，Queueという環境変数が存在しない場合にundefinedになるのが許せない，と怒られるので`QueueURL`の後ろに`??''`を書いて黙らせましょう。]。ですが，現状環境変数をなにも設定していないので，このままでは当然動きません。このLambda関数とSQSを関連付けます。

まず，`amplify/backend/backend-config.json`を開きます。ここには，Amplify CLIで作った全リソースの情報が書かれていて，基本的には自動生成されますが手動で編集することも可能です^[手動で変更した箇所はOverwriteされないという意味です。]。以下の記載があると思いますが，それが先ほど作ったSQSの情報です。
```json:backend-config.json
{
  "custom": {
    "queueResourceName": {
      "service": "customCloudformation",
      "providerPlugin": "awscloudformation",
      "dependsOn": []
    }
  }
}
```

`queueResourceName`は`amplify add custom`でSQSを作った際に入力したResource Nameです。また，このファイルには他にもLambda関数の情報も載っています。
```json
{
  "function": {
    "functionResourceName": {
      "build": true,
      "providerPlugin": "awscloudformation",
      "service": "Lambda"
    }
  }
}
```

この2つを関連付けます。Lambda関数に`dependsOn`というプロパティを付け加え，SQSの情報を渡します。
```json
{
  "function": {
    "functionResourceName": {
      "build": true,
      "providerPlugin": "awscloudformation",
      "service": "Lambda",
      "dependsOn": [
        {
          "category": "custom",
          "resourceName": "queueResourceName",
          "attributes": [
            "QueueURL",
            "QueueArn"
          ]
        }
      ]
    }
  }
}
```

`Attributes`には，先ほどのCFnテンプレートの`Outputs`で指定した名前を入れます。編集後に，一度ステージをチェックアウトして変更があったことを認識させます。
```bash
$ amplify env checkout <ステージ名>
✔ Initialized provider successfully.
Initialized your environment successfully.
```

これで，Lambda関数のCFnテンプレートがSQSのURLとArnを認識できるようになりました！次にLambda関数のCFnテンプレートを編集します。
まずは`Parameter`にQueueURLとQueueArnを入れます。
```json:<Lambdaリソース名>-cloudformation-template.json
{
  "Parameters": {
    "customqueueResourceNameQueueURL": {
      "Type": "String",
      "Default": "customqueueResourceNameQueueURL"
    },
    "customqueueResourceNameQueueArn": {
      "Type": "String",
      "Default": "customqueueResourceNameQueueArn"
    }
  }
}
```

ここの命名規則は<リソースカテゴリ><リソース名><Attribute名>（スペースなし）です^[かなり見にくい...]。最後に，これをLambda関数の環境変数に入れて，SQSへのアクセス権限を与えます^[以下のテンプレートは必要箇所のみを抜き出して書いています。実際のテンプレートファイルと見比べながら使ってください！]。
```json
{
  "Resources": {
    "LambdaFunction": {
      "Type": "AWS::Lambda::Function",
      "Properties": {
        "Environment": {
          "Variables": {
            "Queue": {
              "Ref": "customqueueResourceNameQueueURL"
            }
          }
        }
      }
    },
    "AmplifyResourcePolicy": {
      "Type": "AWS::IAM::Policy",
      "Properties": {
        "PolicyDocument": {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Action": [
                "sqs:SendMessage",
                "sqs:SetQueueAttributes"
              ],
              "Resource": [
                "*"
              ]
            }
          ]
        }
      }
    }
  }
}
```

SQSを許可するポリシーですが，Resourceは一旦`"*"`で設定します^[一度に2個のAttributionを取得できない可能性があり，エラーを避けるためです。]。この状態で一度`amplify push`をして変更を適用します。

問題なく適用できたら，次にResourceを`{ "Ref": "customqueueResourceNameQueueArn" }`としてもう一度pushします。

実際にLambda関数を見に行って，QueueURLやポリシーが設定されているか確認しましょう。
![Lambda関数の環境変数](/images/add-sqs-to-amplify-project/lambda-env.png)
![Lambda関数の実行ロール](/images/add-sqs-to-amplify-project/lambda-role.png)

きちんとSQSのQueueURLとArnが設定されているのが確認できました！

# まとめ
カスタムリソースを使うことで，どんなAWSサービスでも関連付けられるようになるので，Amplifyがますます最強になっていく気がしています。今までAmplifyを使わずに手動で設定していた方でも，Lambda+SQSのようなよくあるサーバーレスパターンをAmplifyで構築できるようになっているので，導入コストはかなり低くなっていると感じました。ぜひカスタムリソースを使ってみてください！
