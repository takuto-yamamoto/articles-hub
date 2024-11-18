---
title: 'DynamoDB で部分取得・更新・削除の API を実装する'
emoji: '🐶'
type: 'tech'
topics: ['aws', 'serverless', 'restapi', 'dynamodb', 'apigateway']
published: true
published_at: 2024-11-16 08:00
---

## 概要

大規模なデータセットを CRUD する場合、効率性を保ちながらデータを操作するには部分的な取得・更新・削除操作が重要です。

この記事では、API Gateway, Lambda, DynamoDB を用いた一般的な AWS サーバレス構成における、部分 CRUD に対応した 汎用的な REST API を実装することを目指します。

:::message
本記事で実装する API および IaC のコードの全文は、[こちらのリポジトリ](https://github.com/takuto-yamamoto/aws-dynamodb-selective-cruds)で公開しています。
:::

## 部分取得

以下の `User` エンティティを考えます

```typescript
type User = {
  id: string;
  username: string;
  bio?: string; // typescriptにおけるoptional表現
};
```

ここから `username` 属性だけを取得するにはどうすれば良いでしょうか？

以下のステップに分けて考えてみましょう。

1. 取得したい属性を指定する
2. 指定した属性のみ取得する
3. 特殊な名前を持つ属性に対応する
4. 深さのある属性に対応する

### 1. 取得したい属性を指定する

レスポンスデータの取得属性を絞るための一般的な手段として、クエリパラメータが挙げられます。

例えば、`username` 属性のみを取得する場合は、`GET /users/:userId?field=username` のようになります。

あるいは複数の属性を取得する場合、 `?field=username&field=bio` のようになるでしょう。

:::message
`?fields=username,bio` のような実装も一般的です。
:::

今回のように、API Gateway + Lambda の場合は、Lambda ハンドラ内で以下のように `field` パラメータを受け取ります。

```typescript
// 存在しない場合は[]にフォールバック
const fields = event.multiValueQueryStringParameters?.field ?? [];
```

### 2. 指定した属性のみ取得する

受け取った `field` パラメータは、[ProjectionExpression](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Expressions.ProjectionExpressions.html) として DynamoDB へのクエリに反映させる必要があります。

例えば、`username` 属性 と `bio` 属性を取得したい場合は `ProjectionExpression: 'username, bio'` となります。

任意の `field` パラメータに対しては、

```typescript
const input: GetCommandInput = {
  TableName: tableName
  Key: { id: userId },
};

// fieldが指定されている場合、ProjectionExpressionに反映させる
if (fields.length > 0) {
  input.ProjectionExpression = fields.join(', ');
}
```

と実装することができます。

### 3. 特殊な名前を持つ属性に対応する

`ProjectionExpression` パラメータには、DynamoDB の予約語( `Name` や `Size` 等)を使用できないという制約があります。

この制約に対応するために、[ExpressionAttributeNames](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Expressions.ExpressionAttributeNames.html) によるエイリアスを使用します。

:::message
`ExpressionAttributeNames` や 後述の `ExpressionAttributeValues` を使用することは、DynamoDB の予約語を悪用した NoSQL インジェクションの対策としても機能します。
:::

例えば、 `Name` 属性と `Size` 属性を取得したい場合は

```typescript
// エイリアスされた属性名で構成される
input.ProjectionExpression = '#attr0, #attr1';
input.ExpressionAttributeNames = {
  '#attr0': 'Name',
  '#attr1': 'Size',
};
```

という設定値になっている必要があります。

任意の `field` パラメータに対しては、

```typescript
// エイリアスオブジェクトの初期化
const expressionAttributeNames: Record<string, string> = {};

// プレースホルダーを用いた projectedFields を計算
const projectedFields = fields.map((field, i) => {
  // プレースホルダを生成し、実際の field にエイリアス
  const placeholder = `#attr${i}`;
  expressionAttributeNames[placeholder] = field;
  return placeholder;
});

// コマンドinputにエイリアスを設定
input.ProjectionExpression = projectedFields.join(', ');
input.ExpressionAttributeNames = expressionAttributeNames;
```

と実装することができます。

### 4. 深さのある属性に対応する

既存の `User` エンティティに、以下のような `preferences` 属性を追加し、`preference.language` 属性だけを取得すること考えます。

```typescript
type User = {
  // 他の属性は省略
  preferences: {
    language: 'en' | 'jp';
    notifications: {
      email: boolean;
      sms: boolean;
    };
  };
};
```

この場合、`?field=preference.language` のようなパラメータ指定が考えられます。

しかしながら、このように指定した場合 DynamoDB は 「 `preference` 属性の `language` 属性の値」ではなく、「 `preference.language` という 1 階層の属性の値」を取得しようとする仕様を持ちます。

これを防ぐために、以下のように `ExpressionAttributeNames` と `ProjectionExpression` を設定する必要があります。

```typescript
input.ProjectionExpression = '#attr0_0.#attr0-1'; // 各階層ごとにエイリアスが必要
input.ExpressionAttributeNames = {
  '#attr0_0': 'preference',
  '#attr0_1': 'language',
};
```

任意のケースに対応するためには、

```typescript
const expressionAttributeNames: Record<string, string> = {};
const projectedFields = fields.map((field, i) => {
  // 属性をドットで分割
  const fieldParts = field.split('.');
  // 各階層ごとに ExpressionAttributeNames に反映
  const placeholders = fieldParts.map((fieldPart, k) => {
    const placeholder = `#attr${i}_${k}`;
    expressionAttributeNames[placeholder] = fieldPart;
    return placeholder;
  });
  // エイリアス済みの属性名を結合して ProjectionExpression を構成
  return placeholders.join('.');
});
input.ProjectionExpression = projectedFields.join(', ');
input.ExpressionAttributeNames = expressionAttributeNames;
```

のように実装することができます。

ただしこのままだと、いくらでも深い `field` を指定できてしまうため、 `field` の深さを制限しておきます。

```typescript
// 3階層目以降の属性は個別に取得できない
// preferences.notifications.sms -> ['preferences', 'notifications']
const maxDepth = 2;
const fieldParts = field.split('.').slice(0, maxDepth);
```

以上で、基本的な部分取得の実装は完了です。

## 部分更新

部分更新の考え方は、部分取得よりも少し複雑です。

以下のステップに分けて検討します。

1. 更新したい属性を指定する
2. 指定した属性のみ更新する
3. 特殊な名前を持つ属性に対応する、深さのある属性に対応する
4. 【new】暗黙的に属性を指定する

### 1. 更新したい属性を指定する

部分取得同様、更新したい属性は `field` パラメータを用いて指定します。

例えば、`username` 属性のみを更新する場合は、`PATCH /users/:userId?field=username` のようになります。

:::message
部分的な更新を想定するため、PUT メソッドではなく PATCH メソッドを使用します。
:::

### 2. 指定した属性のみ更新する

更新クエリにおいては、`ProjectionExpression` の代わりに [UpdateExpression](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Expressions.UpdateExpressions.html) を使用します

例えば、`username` 属性 と `bio` 属性を更新したい場合は、

```typescript
const username = 'ユーザ1';
const bio = 'ユーザ1のbio';
input.UpdateExpression = `SET username = ${username}, bio = ${bio}`;
```

のように `UpdateExpression` を定義します。

任意の属性について `UpdateExpression` を定義する場合は、

```typescript
// 更新式は SET から開始
let updateExpression = 'SET ';

// 各属性ごとに式を作成し追加
updateExpression += fields
  .map((field) => `${field} = ${data[field]}`)
  .join(', ');
```

のように実装することができます。

### 3. 特殊な名前を持つ属性に対応する、深さのある属性に対応する

部分取得の場合同様、DynamoDB の予約語や、深い属性の問題に対応する必要があります。

さらに部分更新の場合は、属性名に対応する「値」においても、予約語を回避する必要があります。

例えば、 `preferences.language` 属性を、`jp` に更新したい場合は、

```typescript
const language = 'jp';
input.ExpressionAttributeNames = {
  '#attr0_0': 'preferences',
  '#attr0_1': 'language',
};
input.ExpressionAttributeValues = {
  ':val0': language,
};
input.UpdateExpression = `SET #attr0_0.#attr0_1 = :val0`;
```

というコマンド設定になります。

任意の属性名とその値に対しては、以下のように実装することができます。

```typescript
// エイリアスオブジェクト(属性名、値)と更新式
const expressionAttributeNames: Record<string, string> = {};
const expressionAttributeValues: Record<string, any> = {};
let updateExpression = 'SET ';

// 属性の深さを制限
const targetFields = fields.map((field) =>
  field.split('.').slice(0, maxFieldDepth).join('.')
);

// field ごとに更新式を生成し、カンマで結合
updateExpression += targetFields
  .map((field, i) => {
    // 値のエイリアス
    const valKey = `:val${i}`;
    // 属性名のエイリアス
    const attrKeys = field.split('.').map((fieldPart, k) => {
      const attrKey = `#attr${i}_${k}`;
      expressionAttributeNames[attrKey] = fieldPart;
      return attrKey;
    });
    // 更新データから対象のフィールドの値を取得し、値のエイリアス
    expressionAttributeValues[valKey] = getNestedValue(data, field);
    return `${attrKeys.join('.')} = ${valKey}`;
  })
  .join(', ');
```

なお、上記実装の途中に登場する `getNestedValue(data, field)` については、以下のように実装しています。

```typescript
function getNestedValue<T>(
  data: Record<string, any>,
  field: string
): T | undefined {
  let targetValue: any = data;

  // オブジェクトの場合は指定した子要素を取得
  for (const fieldPart of field.split('.')) {
    if (typeof targetValue !== 'object' || targetValue === null) {
      return undefined;
    }
    targetValue = targetValue[fieldPart];
  }

  return targetValue as T;
}
```

### 4. 暗黙的に属性を指定する

以下のようなリクエストが送信された場合、API はどのような挙動をとるべきでしょうか？

```plaintext
PATCH /users/:userId
{
  "username": "ユーザ1_更新",
  "preferences": {
    "language": "jp"
  }
}
```

他の属性には影響を与えずに、`username` 属性と `preferences.language` 属性だけ更新されて欲しいですよね。

このような挙動を実現するためには、`field` パラメータが指定されない場合に、リクエストボディから動的に `field` を計算する必要があります。

リクエストボディから `field` を計算する関数は、以下のように実装できます。

```typescript
const inferFields = (data: Record<string, any>, maxDepth: number) => {
  const inferredFields: string[] = [];

  // データの各属性について検証
  for (const [field, value] of Object.entries(data)) {
    if (value === undefined) continue;

    if (
      typeof value === 'object' &&
      !Array.isArray(value) &&
      value !== null &&
      maxDepth > 1
    ) {
      // オブジェクトかつnull/配列ではない場合、深さ制限をかけた上で再起的に属性名を連結し、 field を生成する
      const nestedFields = inferFields(value, maxDepth - 1).map(
        (part) => `${field}.${part}`
      );
      inferredFields.push(...nestedFields);
    } else {
      // null/配列/リテラルの場合は、属性名をそのまま field として記載
      inferredFields.push(field);
    }
  }

  return inferredFields;
};
```

以上で、基本的な部分更新の実装は完了です。

## 部分削除

部分削除は、部分更新のリクエストに `NULL` を許容することで実現できます。

例えば、`bio` 属性を削除したい場合、以下のようにリクエストすることで部分削除が可能です。

```plaintext
PATCH /users/:userId
{
  "bio": null,
}
```

:::message
null による部分削除を許容する場合、DB や API のスキーマ側でも null を許容する必要があることに注意してください。
:::

以上で、基本的な部分削除の実装は完了です。

## 残存課題

### 属性の完全な削除

基本的な部分削除においては、属性の値を null に設定することはできますが、属性そのものを消すことはできません。

属性そのものを削除したい場合は、以下のように空のオブジェクトと `field` 属性を用いると良いでしょう。

```plaintext
PATCH /users/:userId?field=preferences.notifications
{
  "preference": {},
}
```

ただし注意点として、DynamoDB において属性そのものを削除する場合は、`UpdateExpression` の `REMOVE` 句を使用する必要があります。

「指定した `field` に対応する値が `undefined` であった場合は `REMOVE` 句を使用する」という条件分岐が追加する必要があるでしょう。

### トレードオフ

以上の内容から読み取れる通り、部分 CRUD の実装は複雑です。またその細部は、使用するインフラ環境に大きく依存します。

さらに注意すべき点として、一つのリソースに対して部分 CRUD を実装する場合は、他の全てのリソースについても同様に部分 CRUD を実装しないと、ユーザーの誤解を招くことになるでしょう。

以上の特性を踏まえた上で、実際の実装時には、実装メリットとコストをしっかりと天秤にかける必要がありそうです。

## 最後に

今回記事投稿に至った理由としては、DynamoDB の部分 CRUD 操作についての既存情報をほとんど見つけられなかったためです。故に本記事の内容についても、私個人が独自に考えた部分が多いです。

まだまだ浅学の身ですので、仕様の考慮漏れや杜撰な実装等多くあるかと思いますが、何か一つでも参考になるエッセンスを提供できていれば幸いです。

最後まで読んでいただきありがとうございます！
ご意見などございましたら、お気軽にコメントお願いします！
