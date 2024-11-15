# API Gateway + Lambda + DynamoDB で部分的な CRUD 操作を実装する

大規模なデータセットを CRUD する場合、効率性を保ちながらデータを操作するには部分的な取得/更新/削除操作が重要であり、これは DynamoDB でも然りです。

この記事では、API Gateway, Lambda, DynamoDB を用いた一般的な AWS サーバレス構成における、部分 CRUD に対応した REST API を実装することを目指します。

:::message
本記事で実装する API および IaC のコードの全文は、[こちらのリポジトリ](https://github.com/takuto-yamamoto/aws-dynamodb-selective-cruds)で公開しています。
:::

## 部分取得

例えば、DynamoDB のテーブル上に以下の User エンティティがあるとします

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

リクエスト側で属性を指定するための一般的な方法は、クエリパラメータを用いることです。

例えば、`username` 属性のみを取得する場合は、`GET /users/:userId?field=username` のようになります。

あるいは、`username` と `bio` 属性など、複数の属性を取得する場合、 `?field=username&field=bio` のようになるでしょう。

API Gateway + Lambda の場合は、Lambda ハンドラ内で以下のように `field` パラメータを受け取ります。

```typescript
// 存在しない場合は[]にフォールバック
const fields = event.multiValueQueryStringParameters?.field ?? [];
```

:::message
複数の属性を取得する場合は `?fields=username,bio` のような実装も一般的です。
:::

### 2. 指定した属性のみ取得する

受け取った `field` パラメータは、DynamoDB へのクエリに、[ProjectionExpression](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Expressions.ProjectionExpressions.html) として反映させる必要があります。

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

この制約に対応するためには、[ExpressionAttributeNames](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Expressions.ExpressionAttributeNames.html) によるエイリアスを使用します。

:::message
`ExpressionAttributeNames` や 後述の `ExpressionAttributeValues` を使用することは、DynamoDB の予約語を悪用した NoSQL インジェクションの対策にもなります。
:::

例えば、 `Name` 属性と `Size` 属性を取得したい場合は

```typescript
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

以下のような `preferences` 属性を追加し、`preference.language` 属性だけを更新すること考えます。

```typescript
type User = {
  // 他の属性は省略
  preferences: {
    language: 'en' | 'jp';
    notifications: {
      email: true;
      sms: true;
    };
  };
};
```

この場合、`?field=preference.language` のようなパラメータ指定が考えられますが、現状の実装だと DynamoDB は 「 `preference` 属性の `language` 属性の値」ではなく、「 `preference.language` という 1 階層の属性の値」を取得しようとします。

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
3. 特殊な属性や深い属性の更新に対応する
4. 暗黙的に部分更新する

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

### 3. 特殊な属性や深い属性の更新に対応する

部分取得の場合同様、DynamoDB の予約語や、深い属性の問題に対応する必要があります。

基本的な考え方は部分取得の場合と同じですが、部分更新の場合は、属性名ではなく対応する値においても予約語を回避する必要があります。

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

### 4. 暗黙的に部分更新する

以下のような、`field` パラメータが指定されていないリクエストが送信された場合は、API はどのような挙動をとるべきでしょうか？

```plaintext
PATCH /users/:userId

{
  "username": "ユーザ1_更新",
  "preferences": {
    "language": "jp"
  }
}
```

`username` 属性および、`preferences.language` 属性のみが更新され、他の属性が既存の値のままである場合に、直感的で使いやすいと感じるのではないでしょうか。

このような挙動を実現するためには、`field` パラメータが指定されない場合に、リクエストボディから動的に `field` を計算する機能を実装する必要があります。

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
      const nestedFields = this.inferFields(value, maxDepth - 1).map(
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

このような部分削除を許容する場合、DB スキーマや API の型バリデーションにおいて、null 値を許容する必要がある点に注意してください。
