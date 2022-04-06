---
title: "Amplify GraphQL APIをLambda Functionから操作する"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "amplify", "graphql"]
published: false
---
こんにちは，[@ry_km](https://twitter.com/ry_km_u_u)です。

前回の記事に引き続き，[AWS Amplify](https://aws.amazon.com/jp/amplify/)に関する記事です。
今回は，Amplify上で用意したGraphQL APIをLambda Function上で操作する方法についてまとめます。

自分でやっていて，かなり手こずったので同じ境遇の方々の参考になれば嬉しいです！

# GraphQL APIをLambdaから操作するとは？
AWS AmplifyのGraphQL APIについて紹介は，詳しい記事がたくさんあるのでそちらにお任せすることにして，どうしてこのような利用方法をするのかを軽く紹介します。

以下は，今回僕が使ったGraphQLスキーマの一部です。
```graphql
type RequestRecord @model @auth(rules: [
  { allow: owner, ownerField: "User" },
  { allow: private, provider: iam }
]){
  RequestId: String! @primaryKey
  Status: Int!
  User: String
}
```

ユーザーがデータをアップロードし，それをクラウド上で編集してダウンロードできるようにする，といった流れです。そのため，`Status`ではファイルの処理状況を管理していて，随時ユーザー側の画面に反映しています。したがって処理の進行状況に応じて`Status`を書き換える必要があり，これはフロント側からはできないので，Lambda関数から操作しなければなりません。

このように，Lambda上での変化をユーザー側に反映させる場合に必要となるため，使う頻度は高いと思います。

以下では，このスキーマを使って説明していきます。また，次の設定は完了している前提で進めます。
- Amplifyをプロジェクト全体で有効化済み：`amplify init`
  - GraphQL APIの設定済み：`amplify add api`
  - Functionを追加済み：`amplify add function`

# Graph QLをLambdaで操作するのに必要な権限
権限の設定をGraphQL側とFunction側の両方から見ていきます。以下では，Lambda関数をAmplifyのリソース名であるFunctionと呼びます。
## GraphQL側
AmplifyのGraphQLでは認証方法がいくつか用意されています。
- Cognito User Pool (userPools)
- IAM Policy (iam)
- OpenID Connect (oidc)
- API Key (apiKey)
- Lambda Authorizer (lambda)

https://docs.amplify.aws/cli/graphql/authorization-rules/#authorization-strategies

Amplify上でAuthが設定されている場合，デフォルトでCognito User Poolを使って認証しようとしますが，Functionからは使うことができません。Functionで有効な認証方法は次の2種類です。
- API Key
- IAM Policy

:::message alert
Lambda Authorizerは自分で認証方法を用意するという意味で，Function用の認証ではありません！
:::

このうち，API Keyは有効期限が最長で365日と制限があるため，長期運用には向きません。したがって，IAM Policyを使う方法が便利です。
GraphQLは複数の認証方法を設定することができるため，Cognito User PoolとIAM Policyの両方を一つのモデルに追加することが可能です！

以下，その方法です。
1. GraphQLスキーマを編集する
  IAM認証が必要なモデルの`@auth`にIAM認証`{ allow: private, provider: iam }`を追加します。

https://docs.amplify.aws/cli/graphql/authorization-rules/#configure-multiple-authorization-rules

2. GraphQLの設定を更新する
  以下のように，IAM認証をGraphQLの設定に追加します。
  ```bash
  $ amplify update api
  
  ? Select a setting to edit.
  > Authorization modes
  ? Configure additional auth types?
  > yes
  ? Choose the additional authorization types you want to configure for the API.
  > IAM

  ✅ Successfully updated resource
  ```

3. リソースを更新する：`amplify push` 

## Function側
次に，Function側を設定します。以下，Function名は`gql-function`，API名は`gql-api`とします。

CLIでAPIへのアクセスを許可します。
```bash
$ amplify update function

? Select the Lambda function you want to update.
> gql-function
? Which setting do you want to update?
> Resource access permissions
? Select the categories you want this function to have access to.
> API
? Select the operations you want to permit on gql-api.
> Query, Mutate

You can access the following resource attributes as environment variables from your Lambda function
  API_GQL-API_GRAPHQLAPIENDPOINTOUTPUT
  API_GQL-API_GRAPHQLAPIIDOUTPUT
```

これで，FunctionにGraphQLへのアクセス許可が付与されました。最後に表示されるのは，GraphQLのエンドポイントが格納される環境変数名です。

# Functionの実装方法
次に，IAM認証を使ってLambdaからGraphQLへアクセスするコードを準備します。今回は例として，既に存在するレコードの内容を上書きする関数を用意します。
以下では，プロジェクトのルートを`~`と表します。

まず，実装に必要なパッケージをインストールします。
```json:package.json
{
  "name": "gql-function",
  "dependencies": {
    "aws-appsync": "^4.0.0",
    "aws-sdk": "^2.1084.0",
    "graphql": "^15.7.0",
    "graphql-tag": "^2.12.6",
    "isomorphic-fetch": "^3.0.0",
  }
}
```

後ほど出てきますが，パッケージのバージョンが非常に重要です。

次に，GraphQLを操作する関数を用意します。今回のスキーマでは，`RequestId`がプライマリーキーのため，Mutationでは必ず指定する必要があります。
```typescript:graphqlOperations.ts
import * as AWS from 'aws-sdk';
import { gql } from 'graphql-tag';
import { print } from 'graphql';
import AWSAppSyncClient from 'aws-appsync';
import 'isomorphic-fetch';

// デフォルトで~/src/graphql/mutation.ts に作成されるクエリを
// そのまま持ってきています。
// gqlにテンプレートリテラルとして読み込ませます。
const updateRecord = gql`
  mutation UpdateRequestRecord(
    $input: UpdateRequestRecordInput!
    $condition: ModelRequestRecordConditionInput
  ) {
    updateRequestRecord(input: $input, condition: $condition) {
      RequestId
      Status
      User
      createdAt
      updatedAt
    }
  }
`;

const gqlUpdate = async (
  url: string,
  region: string,
  RequestId: string,
  Record: object
) => {
  const client = new AWSAppSyncClient({
    url,
    region,
    auth: {
      type: 'AWS_IAM',
      credentials: (AWS.config.credentials ?? null)
    },
    disableOffline: true
  });
  try {
    await client.mutate({
      mutation: updateRecord,
      variables: {
        input: {
          RequestId,
          ...Record
        }
      }
    });
  } catch (error) {
    console.error(error);
  }
};

export default gqlUpdate;
```

最後に，上記の`update`関数を呼び出す関数を作ります。
```typescript:index.ts
import gqlUpdate from './graphqlOperations';

export const handler = async () => {
  // Any statements here...
  const requestId = 'hogehoge';
  await gqlUpdate(
    process.env['API_GQL-API_GRAPHQLAPIENDPOINTOUTPUT'],
    process.env['REGION'],
    requestId,
    { Status: 0 }
  );
};
```

以上で完成です！！

と，実はここまでの内容はどの記事にでも書いてあることです。

# ここからが大変だった...
すっっっっごく大変でした。以下，つまづいた点を上げていきます。

## `TypeError: Cannot convert undefined or null to object`
以上のコードを全て書いて動かしたのに，レコードが一向に更新されません。ログを確認すると，見たこともないエラーを吐いていました。
```json
{
    "errorType": "TypeError",
    "errorMessage": "Cannot convert undefined or null to object",
    "stack": [
        "TypeError: Cannot convert undefined or null to object",
        "    at Function.keys (<anonymous>)",
        "    at /var/task/node_modules/apollo-cache-inmemory/lib/bundle.umd.js:331:12",
        "    at /var/task/node_modules/apollo-cache-inmemory/lib/bundle.umd.js:2:68",
        "    at Object.<anonymous> (/var/task/node_modules/apollo-cache-inmemory/lib/bundle.umd.js:5:2)",
        "    at Module._compile (internal/modules/cjs/loader.js:1085:14)",
        "    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1114:10)",
        "    at Module.load (internal/modules/cjs/loader.js:950:32)",
        "    at Function.Module._load (internal/modules/cjs/loader.js:790:12)",
        "    at Module.require (internal/modules/cjs/loader.js:974:19)",
        "    at require (internal/modules/cjs/helpers.js:101:18)"
    ]
}
```

「undefinedまたはnull値をobjectに変換することはできません」...？？真っ当なことを言っていますが，undefined/nullになるような値はそもそも[TSくん](https://typescript-jp.gitbook.io/deep-dive/type-system/exceptions#typeerror)が注意してくれているので，自分のコードに問題はないはずです。

エラーのトレースを見る限りでは`apollo-cache-inmemory`に問題がありそうですが，これはGraphQLのパッケージに使われている依存モジュールなので，そもそもいじれないところです。

これは次のIssueで解決していました。
https://github.com/awslabs/aws-mobile-appsync-sdk-js/issues/691

つまり，GraphQLのバージョンが新しいとAWS AppSync Clientに対応しておらずエラーがでるとのことでした！
上記にも書きましたが，`GraphQL@15.7.0`をインストールすることでこのエラーは回避できます^[執筆時点での最新バージョンはGraphQL@16.3です。]。

## `fetch is not found globally and no fetcher passed`
次に遭遇したエラーはこちらです。Fetchモジュールがなくてエラーが出ています。
こちらは，丁寧に解決方法がエラーメッセージ中に書かれています。
```
Error: fetch is not found globally and no fetcher passed,
to fix pass a fetch for your environment like https://www.npmjs.com/package/node-fetch."

For example:
import fetch from 'node-fetch';
import { createHttpLink } from 'apollo-link-http';
const link = createHttpLink({ uri: '/graphql', fetch: fetch });
```

このエラーは前述のコードに示した通り，`isomorphic-fetch`をimportすることで回避できます。また，エラーメッセージの通りに`node-fetch`を入れることでも回避できるようです。

## `Not Authorized to access ** on type Mutation`
これは，複数のFunctionからGraphQL APIにアクセスを許可したときに吐かれたエラーです。
```json
ERROR	ApolloError: GraphQL error: Not Authorized to access ** on type Mutation
  graphQLErrors: [
    {
      path: [Array],
      data: null,
      errorType: 'Unauthorized',
      errorInfo: null,
      locations: [Array],
      message: 'Not Authorized to access ** on type Mutation'
    }
  ],
  networkError: null,
  extraInfo: undefined
}
```

要は権限がないというエラーです。しかし，上述のFunction側の権限設定は全てのFunctionに対して行っていました。Lambdaコンソール上でも確かにAppSyncへのアクセスは許可されていました。なぜだ。。。

この問題は，以下のIssueで解決しました。
https://github.com/aws-amplify/amplify-cli/issues/9866#issuecomment-1074698348

次のファイルを`~/amplify/backend/api/(API名)/`内に作ることでエラーが出なくなりました！
```json:custom-roles.json
{
  "adminRoleNames": ["arn:aws:sts::(アカウント番号):assumed-role"]
}
```
しかし，これは一体どういうことでしょう？問題の根源はIAMの仕組みの奥深くに根付いていました。

GraphQLのIAM認証はAmplify上で作成したIAM Roleに対してのみ有効ですが，Amplify外で作成されたIAM Roleに対しても権限を与える方法があります。以下の記事にその方法が書かれています。
https://docs.amplify.aws/cli/graphql/authorization-rules/#use-iam-authorization-within-the-appsync-console

その権限を与えるIAM RoleにAssumed-Roleを加える，というのが上記のIssueに書かれた方法です。すなわちどういうことかというと，「指定したアカウント内でAppSyncへの権限を持つPolicy」を含むRoleは全てアクセスを許可する，という意味です^[この辺の解釈は僕も理解が怪しいので，間違っていることがあれば教えていただけると幸いです！]。

もう少し分かりやすく噛み砕きます。
Lambda関数へは一般にLambdaExecutionRoleを設定しますが，Role内にPolicyを追加して具体的にアクセス可能なサービスを選びます。LambdaExecutionRoleは実行時に一時クレデンシャルが必要なため，ARN`arn:aws:sts::(アカウント番号):assumed-role/LambdaExecutionRole/(セッション名)`を持つユーザーを与えられます。
このARNは毎回変わるので，先ほどの`custom-roles.json`に指定することはできません。

そこで，`assumed-role`までを指定することで，アカウント内のAssumeRoleによって作成されたユーザーにアクセスを許可することができます。
しかしながら，元のPolicyにAppSyncの許可がないと当然アクセスはできないので，全てのユーザーに許可されたわけではないことに注意が必要です。

Role・Policy・AssumeRoleの関係は以下の記事がとても分かりやすいです！
https://dev.classmethod.jp/articles/iam-role-passrole-assumerole/

# まとめ
最終的にはすべてのエラーが解決し，運用できるようになりました！

AmplifyのGraphQL APIをLambda関数から操作する実例は，ネット記事でもあまり載っておらず，かなり途方に暮れていました。
同じようなエラーが出た方の参考になれば嬉しいです！