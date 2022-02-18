---
title: "JavaScriptでObjectのKeyに変数展開する"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: true
---
こんにちは，[@ry_km](https://twitter.com/ry_km_u_u)です。
似たような内容の記事はいくつかあったので，忘れないように自分用のメモとして書きます。タイトルの通り，JavaScriptでObjectのKeyに変数展開する方法についてです。

## JavaScriptのObjectについておさらい
JavaScriptのObjectはJSONと似ていますが，Keyにクオーテーションマークを使わなくてよかったり，Valueに関数を入れたりできるなど汎用性が高くなっています。

```javascript
const person = {
  name: ['Bob', 'Smith'],
  age: 32,
  gender: 'male',
  interests: ['music', 'skiing'],
  bio: function() {
    console.log(`${this.name[0]} ${this.name[1]} is ${this.age} years old. He likes ${this.interests[0]} and ${this.interests[1]}.`);
  },
  greeting: function() {
    console.log(`Hi! I\'m ${this.name[0]}.`);
  }
};

person.greeting() // Hi! I'm Bob.
```

(出典：[MDN Web Docs - JavaScriptオブジェクトの基本](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Objects/Basics) 一部筆者にて加筆修正)

ちなみに，上記コードで`function() {}`をアロー関数式`() => {}`に変えるとうまくいかないです^[アロー関数が使えない場面は初めて見ました！引用した記事を参考に細かい違いを勉強したいと思います。]。アロー関数にしてしまうと，[レキシカルスコープ](https://wemo.tech/904#index_id3)の`this`（この場合はグローバルオブジェクト）になってしまって`person`を参照してくれません。

https://qiita.com/suin/items/a44825d253d023e31e4d#%E9%81%95%E3%81%841-this%E3%81%AE%E6%8C%87%E3%81%99%E3%82%82%E3%81%AE

## Keyに変数展開する

上記例を見ていただければ分かる通り，Valueに変数を展開することは当然可能です。しかし，Key側に変数展開しようとすると，展開せずに変数名がそのまま表示されてしまいます。

```javascript
const newKey = 'hoge'
const obj = {
  newKey: 'test'
};
console.log(obj) // { newKey: 'test' }
```
Keyに`'hoge'`のように変数展開したいときは，`[newKey]`のように`[]`で変数名を囲います。

```javascript
const newKey = 'hoge'
const obj = {
  [newKey]: 'test'
};
console.log(obj) // { hoge: 'test' }
```

ちなみに，この書き方はES2015以降で使える方法です。

## この書き方どこかで見たことあるような...
そう！TypeScriptで型指定するときですね！
```typescript
const someObject: {[id: string]: value} = {};
```

`[id: string]`のように，`[]`で囲っていることで`id`が変数であることを明示しているんですね。これを知るまでは，ObjectにはKey-Valueがいくつも入るから配列にしているのかな，とか思っていました^[自分でも何を言っているのかわかりません。その程度の理解だったということです。]。

ちょっとしたことですが，新しいことを知ると嬉しいスッキリしますね。
以上になります。お読みいただきありがとうございました！