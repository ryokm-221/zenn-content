---
title: "Stripeで0円決済を実装する時の注意点"
emoji: "🐹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["stripe"]
published: false
---
この記事は、[Stripe / JP_Stripes Advent Calendar 2024](https://qiita.com/advent-calendar/2024/stripe)シリーズ2・22日目に掲載しています。

こんにちは、[@ry_km](https://twitter.com/ry_km_u_u)です。今回、決済インフラ[Stripe](https://stripe.com/jp)を初めて使ってサービスをリリースしましたが、その中でリリース直前でハマりかけて大急ぎで改修することにポイントについて紹介します。なお、本記事ではリリースしたサービスそのものについては触れません。

# Stripeにおける0円決済
今回リリースしたサービスはいわゆる「サブスクリプション型のSaaS」で提供されるものですが、無料で全機能を試せるベータテスト期間を半年ほど経て、その後に有料販売開始となる流れでした。また、サービス側ではサブスクリプションが申し込まれるごとにシリアルキーを発行し、利用者はアプリケーションにそのシリアルキーを入力することでサービスが利用できるようになります。全体の流れは以下のようになっています。

![Stripeとサービス側のDBの連携](/images/stripe-free-transaction-currency/Architecture-Stripe.png)

サービス側のデータベースとStripeの整合性を確実に取るため、Stripeがサブスクリプションを作成したイベントをフックにして、サービス側のデータベースにシリアルキーが記録される仕組みになっています。この構成を無料期間の時点で組んでおくことで、後々有料期間になっても全体の構成を大きく変えることなくリリースできる、と考えていました。ありがたいことに、Stripeでは[0円決済を行うことが可能](https://docs.stripe.com/payments/checkout/no-cost-orders)です^[リンク先では単発決済の場合を示していますが、同様の方法でサブスクリプションでも0円決済が可能です。また、0円決済を行なっている限りStripeの手数料はかかりません。]。

まとめると、無料期間では
- 0円/月のサブスクリプションを、サービスの会員登録時に契約する

有料期間では
- 0円/月のサブスクリプションは機能制限を加えた上で継続（全ユーザーに与える）
- 有料プランを購入する場合でも、同じフローでシリアルキーが発行される

となります。実際、有料販売開始時のバックエンド改修は最小限で済んだため、この選択は正解だっと感じています。

# だがしかし...
有料販売開始の前日18時ごろ、異変に気づいてしまいました。既存顧客向けの割引クーポンをバッチ処理で割り当てようとした時、次のエラーが出てしまいました。

```json
{
  "error": {
    "message": "The customer currency usd is not in the promotion code's supported currencies: jpy.",
    "param": "currency",
    "type": "invalid_request_error"
  }
}
```

Stripeのテスト環境では問題なくバッチ処理できていたため、「？？？」の状態です。エラーが出た顧客の登録情報を見てみると...

![デフォルト通貨がUSDになっているユーザー](/images/stripe-free-transaction-currency/before_usd.png)
*デフォルト通貨がUSDになっているユーザー。currencyがusdになっている。*

ユーザーのデフォルト通貨が「USD」になっていました。クーポンは日本円で定義された定額割引のため、これではユーザーが使えないためエラーになった、ということでした。しかも、デフォルト通貨はユーザーが商品を購入したあとで変更することはできない^[これはよく考えてみると当然の仕様で、複数通貨での決済を可能にすると為替によって顧客の残高が変わってしまうことになりとても煩雑なシステムになってしまうと思います。]、となっていて詰んだ...と思いました。

> 現在の通貨建ての有効なサブスクリプションや Billing オブジェクトがない場合は、購入者のデフォルトの通貨を変更できます。
> 購入者の通貨が設定された後、その購入者用のサブスクリプションやクーポンを別の通貨で作成することはできません。

（引用：[顧客のデフォルト通貨を設定](https://support.stripe.com/questions/setting-a-customers-default-currency)、一部省略）

なぜデフォルト通貨が「USD」になっていたかという点ですが、これはサービスの販売方法を検討していた時期まで遡ります。当初は、世界進出を見据えて日本国内でもドルで展開しようと考えていました。これと同時期にStripeを使うことが決まったため、「$0/月」の商品を作って「0ドル決済」をしようということが決まったのでした。その後に取得したStripeアカウントでは「$0/月」の商品を実際に作りました。

しかし、無料展開を開始した後に検討を重ねた結果「一旦は日本円での展開にしよう」と方針が変更になり、テスト環境やサンドボックスでは「0円/月」に変更しました。一方、本番環境ではすでに会員登録が行われていたため、商品の価格を変更することができず、そのままにしてありました^[「0円決済なんだから通貨なんか関係ないやろ！」と思っていました。]。

![無料プランに登録された2種類の通貨](/images/stripe-free-transaction-currency/Free-Plan.png)
*無料プランに登録された2種類の通貨*

# 対応
いつもお世話になっているStripeの担当者に泣きながらSlackを送りました。並行して対応を調べてみましたが、方法は「全ユーザーのUSDが紐づくサブスクリプションをキャンセルし、日本円が紐づくサブスクリプションを付け直す」しかありませんでした^[Stripe側でも技術のご担当の方に確認いただいたようですが、同じ返答でした。遅い時間にも関わらずオンラインMTGでご対応いただいて本当に感謝しております！]。今回、ダウンタイムを伴うサービス改修の時間を6時間確保していたため、この処理を改修の手順に追加することにしました。

そこで、上記のUSDからJPYへの付け替えを行うスクリプトを急遽用意しました^[改修実施後に気づきましたが、この修正はStripeのダッシュボードからだと行えませんでした。USDのサブスクリプションを最初に購入してキャンセルし、新しいサブスクリプションを作ろうとしても、日本円の価格は選択肢にでてきませんでした。]。

```typescript
import { Stripe } from 'stripe';
const stripe = new Stripe("api_key");

const main = async () => {
  const limit = Number(process.argv[3]); // 一度に処理する件数を決める
  const initialPointer = process.argv[4]; // 続きのユーザーを指定

  const { customers, pointer } = await getCustomers(limit, initialPointer);

  let i = 0;
  for (const customer of customers) {
    const currentSubscription = await deleteCurrentSubscription(customer.id);
    await createNewSubscription(customer.id);
    i++;
  }

  console.log(`Done ${i}/${limit}`);
  console.log('Next pointer:', pointer);
};

// Stripe utils
const getCustomers = async (limit: number, initialPointer?: string) => {
  const customers: Stripe.Customer[] = [];
  let pointer = initialPointer;

  try {
    const res = await stripe.customers.list({
      limit,
      ...(pointer && { starting_after: pointer })
    });
    pointer = res.data[limit - 1].id;
    customers.push(...res.data);
  } catch (error) {
    console.error('ERROR::getCustomers');
    throw error;
  }
  return {
    customers,
    pointer
  };
};

const deleteCurrentSubscription = async (customer: string) => {
  try {
    const subscription = await stripe.subscriptions.list({
      customer
    });
    await stripe.subscriptions.cancel(subscription.data[0].id);
    return subscription.data[0];
  } catch (error) {
    console.error('ERROR::deleteCurrentSubscription');
    throw error;
  }
};

const createNewSubscription = async (customer: string) => {
  try {
    const res = await stripe.subscriptions.create({
      customer,
      items: [
        {
          price: 'price_id_in_jpy',
          quantity: 1
        }
      ]
    });
    return res;
  } catch (error) {
    console.error('ERROR::createNewSubscription');
    throw error;
  }
};


main()
  .then(() => {
    console.log('Done!');
    process.exit(0);
  })
  .catch(() => {
    console.log('Fail');
    process.exit(1);
  });
```

注意点として、（当然ながら）`subscription_id`は変更になるため、その影響範囲を考える必要がありました^[このサービスの一部にはStripeのsubscription_idをキーにしているデータベースがあります。スクリプトでは省略していますが、データベースのキーを書き換える処理も同時に行いました。]。また、この作業を行うと`customer.subscription.deleted`と`customer.subscription.created`のイベントが発火するため、サービス側のデータベース更新が行われてしまいます。このためStripeのWebHookを一時的に止めて作業を行いました。

この処理を行った結果、全ユーザーのデフォルト通貨を日本円に変更できました！

![デフォルト通貨がJPYになったユーザー](/images/stripe-free-transaction-currency/before_jpy.png)
*デフォルト通貨がJPYになったユーザー。currencyがjpyになっており、日本円の商品を購入できる状態になっていることがわかる。*

# まとめ
つくづくですが、この時点で気づいてよかったと思っています。もしクーポン付与をせずにこの問題に気づけなかった場合、サイトを公開してから「全く決済できない！」という状況に陥るところでした。また、無料のサブスクリプションしか存在していなかったため、ユーザーには全く影響なく作業を終えることができました^[0円決済の場合領収書はユーザーには自動で送られません。また、（当然ですが）日割りの金額調整なども考える必要がなかったので助かりました。]。

有料販売のサービスでは、決済周りの正確な挙動はユーザーの信頼に関わる部分なので、ギリギリの段階でしたが影響が広がる前に気づけて良かったと思っています。今回の開発でStripeのサブスクリプション周りの挙動はかなり理解できました。また単発決済であればもっと簡単に設計できることもわかったので、今後のサービス開発に活かしていきたいと考えています！