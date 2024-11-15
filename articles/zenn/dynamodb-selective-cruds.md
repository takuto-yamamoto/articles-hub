# dynamodb で部分取得・部分更新・部分削除を実装する

DynamoDB は、AWS が提供するスケーラブルな NoSQL データベースで、柔軟性とパフォーマンスを両立したデータ管理を可能にします。

しかし、大規模なデータセットを扱う場合、効率性を保ちながらデータを操作するには、部分的な取得、更新、削除の技術を活用することが重要です。

この記事では、API Gateway, Lambda, DynamoDB を用いた一般的な AWS サーバレス構成における、部分 CRUD に対応した REST API の実装に関する Tips を提供します。

なお本記事で実装する API および IaC のコードは、[こちらのリポジトリ](https://github.com/takuto-yamamoto/aws-dynamodb-selective-cruds)で公開しています（一部変更を加えてあります）。

## 部分取得

例えば、DynamoDB のテーブル上に以下の User エンティティがあるとします

```typescript
type User = {
  id: string;
  username: string;
  bio?: string; // typescriptにおけるoptional表現
};
```

ここから`username`属性だけを取得するにはどうすれば良いでしょうか？

以下のステップに分けて考えてみましょう。

1. 取得したい属性をを指定する
2. 指定した属性のみ取得する
3. 特殊な名前を持つ属性に対応する
4. 深さのある属性に対応する

### 1. 取得したい属性を指定する

リクエスト側で属性を指定するための一般的な方法は、クエリパラメータを用いることです。

`username`属性のみを取得する場合は、`GET /users/:userId?field=username` のようになります。

`username`と`bio`属性など、複数の属性を取得する場合、`?field=username&field=bio` のようになるでしょう。

API Gateway + Lambda の場合は、Lambda ハンドラ内で以下のように`field`パラメータを受け取ります。

```typescript
const fields = event.multiValueQueryStringParameters?.field ?? []; // 存在しない場合は[]にフォールバック
```

なお補足として、複数属性の場合は`?fields=username,bio`のような実装も一般的です。ただし今回は、APIGateway がカンマ区切りのクエリパラメータを自動でパースしてくれないため、不採用としています。

### 2. 指定した属性のみ取得する

受け取った`field`パラメータは、DynamoDB へのクエリに、[ProjectionExpression](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Expressions.ProjectionExpressions.html) として反映させる必要があります。

例えば、`username`と`bio`属性を取得したい場合は `ProjectionExpression: 'username, bio'` となります。

任意の`field`パラメータに対しては、

```typescript
const input: GetCommandInput = {
  TableName: this.tableName
  Key: { id: userId },
};

// fieldが指定されている場合、ProjectionExpressionに反映させる
if (fields.length > 0) {
  input.ProjectionExpression = fields.join(', ');
}
```

と実装することができます。

### 3. 特殊な名前を持つ属性に対応する

`ProjectionExpression` パラメータには、DynamoDB の予約語(`Name` や `Size` 等)を使用できないという制約があります。

この制約に対応するためには、[ExpressionAttributeNames](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Expressions.ExpressionAttributeNames.html) によるエイリアスを使用します。

例えば、`Name`と`Size`属性を取得したい場合は

```typescript
input.ProjectionExpression = '#attr0, #attr1';
input.ExpressionAttributeNames = {
  '#attr0': 'Name',
  '#attr1': 'Size',
};
```

という設定値になっている必要があります。

任意の`field`パラメータに対しては、

```typescript
const expressionAttributeNames: Record<string, string> = {};
const projectedFields = fields.map((field, i) => {
  const placeholder = `#attr${i}`;
  expressionAttributeNames[placeholder] = field;
  return placeholder;
});
input.ProjectionExpression = projectedFields.join(', ');
input.ExpressionAttributeNames = expressionAttributeNames;
```

と実装することができます。

### 4. 深さのある属性に対応する

以下のような `preferences` 属性が追加された場合に、`preference.language`だけを更新したい場合、どうすればいいでしょうか？

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

この場合、`?field=preference.language`のようなパラメータ指定が考えられますが、これを DynamoDB へのクエリに正確に反映させるためには、以下のように`ExpressionAttributeNames`と`ProjectionExpression`を設定する必要があります。

```typescript
input.ProjectionExpression = '#attr0_0.#attr0-1'; // 各階層ごとにエイリアスが必要
input.ExpressionAttributeNames = {
  '#attr0_0': 'preference',
  '#attr0_1': 'language',
};
```

深さのある任意の`field`に対応するためには、

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

ただし、このままだといくらでも深い`field`を指定できてしまい、計算のためのリソース/時間を無駄に消費するため、`field`の深さを制限しておきます。

```typescript
const maxDepth = 2;
const fieldParts = field.split('.').slice(0, maxDepth);
```

以上で、基本的な部分取得の実装は完了です。

## 部分更新

## 部分削除
