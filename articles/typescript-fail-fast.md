---
title: 'TypeScript で「早く失敗」するための Tips 集'
emoji: '🐶'
type: 'tech'
topics: ['typescript', 'nodejs', 'react', 'tips']
published: false
---

## 概要

「早く失敗する(=Fail Fast)」とは、「問題を早めに見つけて直すことが出来れば、手戻りが減り全体的なリスクやコストが抑えられる」という意味の標語です。

この言葉は、一見するとネガティブに聞こえるかもしれません。
しかし実際は、商用一発勝負よりも E2E テストでバグが見つかる方がいいでしょうし、E2E テストよりも単体テストでバグが見つかる方がいいでしょう。
単体テスト実行エラーよりもコンパイルエラーの方が嬉しいですし、もっと言えば、コードの編集中にリアルタイムで問題が分かると最高でしょう。

この記事では、TypeScript コーディングにおいて、そんな「早く失敗する」ための 基本的 Tips を 紹介できればと思います。

## 早く失敗するための Tips

### Tip1: エディタを賢くする

TypeScript の直接的なテクニックという訳ではないですが、エディタが最適化されていればされているほど、コードの編集中にリアルタイムで問題が分かるようになります。

標準の TS 言語サーバだけでも十分に賢いのですが、加えて以下を拡張機能として入れておくことをお勧めします。

- リンタ([eslint](https://eslint.org/) など)
- フォーマッタ([prettier](https://prettier.io/) など)
- スペルチェッカ([cspell](https://cspell.org/) など)

この辺りの機能は、エディタだけでなく pre-commit などにも入れておくと良いと思います。

### Tip2: string 型を使うときは、一旦立ち止まる

「文字列だから」という理由で、勢いで string 型使っていませんか？

実際は数個の候補文字列に絞れるのにも関わらず、 string 型が使用されているケースが結構あると思います。

候補文字列リテラルのユニオン型で定義することで、typo を含む想定されていない候補値に対し、早期に問題を検出することができます。

```typescript
type BadUser = {
  username: string;
  role: string;
};
type GoodUser = {
  username: string;
  role: 'admin' | 'member';
};

const user = {
  username: 'badUser',
  role: 'admim', // typo
};
const badUser: BadUser = user; // 実行するまで問題に気づかない
const goodUser: GoodUser = user; // ERROR: Type 'string' is not assignable to type '"admin" | "member"'.
```

### Tip3: 型文脈にも DRY 原則を適用する

DRY(Don't Repeat Yourself = 繰り返しを避けろ)原則は、型定義においても同様に適用されるべきです。定義元の多重管理を避けることで、変更漏れによる問題を早期に検出できるようになります。

例えば既に以下のような型定義と変数宣言があるとしましょう。

```typescript
type SystemSupportLanguage = 'en' | 'jp';
type BadDictionary = {
  en: string;
  jp: string;
};
type GoodDictionary = {
  [language in SystemSupportLanguage]: string;
};

const badButterflyDict: BadDictionary = { en: 'Butterfly', jp: '蝶' };
const goodButterflyDict: GoodDictionary = { en: 'Butterfly', jp: '蝶' };
```

SystemSupportLanguage 型と BadDictionary 型で `en` や `jp` が二重管理されています。

ある時要件が変更になり、対応する言語にイタリア語を追加することになりました。

```typescript
type SystemSupportLanguage = 'en' | 'jp' | 'it';
```

この時、Good な型定義をしていれば、変更すべき箇所がコンパイルエラーとして早期検出されます。

```typescript
const badButterflyDict: BadDictionary = { en: 'Butterfly', jp: '蝶' }; // 実行までエラーは出ない
const goodButterflyDict: GoodDictionary = { en: 'Butterfly', jp: '蝶' }; // ERROR: Property 'it' is missing in type '{ en: string; jp: string; }'
```

ちなみに、BadDictionary 型から SystemSupportLanguage 型を逆に作ることもできます。

```typescript
type BadDictionary = {
  en: string;
  jp: string;
};
type SystemSupportLanguage = keyof BadDictionary;
```

重要なのは「**定義元が一元化されていること**」です。

:::message
他にも、extends 句やジェネリクス型、組み込みのユーティリティ型なども、DRY 解消に役立ちます。
また API の型をフロントとバックエンドで二重定義しないために、OpenAPI のような言語共通規格を使うことも考えられます。
:::

### Tip4: 常に 1 つのステートに対応する型を設計する

常に 1 つのステートに対応する型を設計しましょう。例えば、こんな型があったとします。

```typescript
type BadFetchInfo<T> = {
  status: 'success' | 'loading' | 'error';
  data: T | null;
  error: string | null;
};
```

リテラルのユニオン型で値が絞られており、ジェネリクス型で汎用性も高そうなので、一見問題はなさそうです。

ですがこの型は、「型定義には則っているのに存在しえない状態」を生み出してしまいます。

```typescript
const result: BadFetchInfo = {
  status: 'success',
  data: null,
  error: 'エラーが発生しました', // fetchのステータスはsuccessなのに実際はエラー
};
```

こうなると開発者は、定義したステートが「存在しうる状態か」を常に意識しないといけません。これは開発者にとって大きな負担です。

**取りうる状態と 1 対 1 で対応する型**を定義することで、開発者はこの苦しみから解放されます。

```typescript
type FetchSuccess<T> = { status: 'success'; data: T; error: null };
type FetchLoading = { status: 'loading'; data: null; error: null };
type FetchError = { status: 'error'; data: null; error: string };

type GoodFetchInfo<T> = FetchSuccess<T> | FetchLoading | FetchError; // 取りうる状態のユニオンで定義する
```

こうすることで、存在し得ない状態に対しては人間ではなく TypeScript のコンパイラや言語サーバがエラーを出してくれるようになり、問題が早期に検出できるようになります。

ちなみにこの考え方は [タグ付きユニオン(判別可能なユニオン)](https://typescriptbook.jp/reference/values-types-variables/discriminated-union) という型定義の手法と非常に相性がいいです。こちらもかなり汎用的な型定義手法なので、知らない方は是非調べてみてください。

:::message
型を厳密に定義することで、コードの堅牢性だけでなく、エディタ開発時のオートコンプリートの充実度にも影響します。
地味ですが、積もり積もって開発体験に大きな貢献をします。
:::

### Tip5: Exhaustive Check(徹底的なチェック)を行う

リテラルのユニオン型に対し条件分岐している場合、元の型定義が更新された時に、条件分岐側でその変更を検知できるでしょうか？

答えは No です。

```typescript
type SystemSupportLanguage = 'en' | 'jp';

if (language === 'jp') {
  funcJp();
} else if (language === 'en') {
  funcEn();
}

// 定義元に'fr'を追加
type SystemSupportLanguage = 'en' | 'jp' | 'fr';

if (language === 'jp') {
  funcJp();
} else if (language === 'en') {
  funcEn();
} // 'fr'の場合の処理は定義されていないが、コンパイルエラーにはならない
```

こんな時に使えるのが Exhaustive Check(徹底的なチェック)です。
通過しないはずの分岐に never 型の変数をおいておくことで、想定外の通過が発生しうる場合はコンパイルエラーを返すことができます。

```typescript
// 定義元に'fr'を追加
type SystemSupportLanguage = 'en' | 'jp' | 'fr';

// 'fr'の条件分岐をしていないため、コンパイルエラーになる
if (language === 'jp') {
  funcJp();
} else if (language === 'en') {
  funcEn();
} else {
  const _exhaustiveCheck: never = language; // ERROR: Type 'fr' is not assignable to type 'never'.
}
```

注意点として、原則通過しない分岐を追加するため、テストカバレッジを取得している場合は影響を与える可能性があります。

また`_exhaustiveCheck` 変数自体は、定義のみで使用はしない変数になるため、リンタなどが`unused-var`としてエラーを出す場合があります。
「`_`から始まる変数は未使用でも無視する」などといったパターンを予め指定しておくといいでしょう。

### Tip6: 型バリデーションライブラリを使う

[zod](https://zod.dev/) や [valibot](https://valibot.dev/) といった型バリデーションライブラリは、型定義において非常に強力です。

例えば、定数から型を定義しその型チェック関数を作りたい場合、純粋な TypeScript だと以下のような実装になるでしょう。

```typescript
const LOG_LEVELS = ['fatal', 'error', 'warn', 'info', 'debug'] as const;

type LogOption = {
  loggerName: string;
  logLevel: (typeof LOG_LEVELS)[number];
};

const isLogOption = (obj: unknown): obj is LogOption => {
  return (
    typeof obj === 'object' &&
    obj !== null &&
    typeof (obj as LogOption).loggerName === 'string' &&
    LOG_LEVELS.includes((obj as LogOption).logLevel)
  );
};

const logOption: unknown = ...

if (!isLogOption(logOption)) {
  throw Error('invalid log option');
}

const { loggerName, logLevel } = logOption;
```

型チェック部分が何をやっているかパッと分かりづらいですし、型チェック部分に実装ミスがあったとしても、なかなか気づくことができません。

型バリデーションライブラリはそんな課題を解決してくれます。以下は zod の例です。

```typescript
import { z } from 'zod';

const LOG_LEVELS = ['fatal', 'error', 'warn', 'info', 'debug'] as const;

const logOptionSchema = z.object({
  loggerName: z.string(),
  logLevel: z.enum(LOG_LEVELS),
});

type LogOption = z.infer<typeof logOptionSchema>;

const logOption: unknown = ...
const { loggerName, logLevel } = logOptionSchema.parse(logOption); // 型が異なる場合はエラー
```

スキーマを定義するだけで、型と型チェック関数兼パーサーが勝手についてきます。

本来型と言うものは TS の機能なので、JS コンパイル時には消えてしまいます。しかしながら関数や値は JS の機能ですので、コンパイル後も存在し続けます。
ゆえに、型チェック関数や、定数に基づく型の生成といった、型(TS)の文脈と値(JS)の文脈を行き来するような実装は、ややこしくなりがちです。
そんなややこしい部分を、型バリデーションライブラリはいい感じに隠蔽してくれます。

他にも型バリデーションライブラリには色々便利な機能が揃っています。使ってみたいと思った方は、是非調べてみてください。

## 終わりに

TypeScript で「早く失敗」するための 6 つの Tips を記載しましたが、いかがだったでしょうか？

開発サプライチェーンにおいて、エディタやコンパイラと言う一番身近な場所で「早く失敗」することには、デプロイやテストを整備するだけでは得られない、非常に大きな価値があると思います。

今後の皆さんの「早い失敗」に向けて、上記の Tips が一つでも参考になれば幸いです。

それでは、これからも楽しい TypeScript ライフを！
