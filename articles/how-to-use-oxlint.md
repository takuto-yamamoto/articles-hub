---
title: 'oxlint で eslint を高速化させる'
emoji: '🐂'
type: 'tech'
topics: ['typescript', 'nodejs', 'react', 'eslint', 'oxlint']
published: true
---

rust 製の爆速 Linter こと oxlint で eslint を高速化したいんじゃ！！！

https://oxc.rs/docs/guide/usage/linter.html#features

というわけでレッツゴー！

## eslint + typescript-eslint は遅い

JavaScript におけるリンターのデファクトとなっている [ESLint](https://eslint.org/)
また ESLint に TypeScript 対応を拡張する [typescript-eslint](https://typescript-eslint.io/) も広く使われている

typescript-eslint がやってくれることは以下の 2 つ

- TypeScript 構文を解析して AST を作成する
- 型情報 Lint ルールの追加

しかしながら、いかんせん**この 2 つを組み合わせると遅い**のである

## oxlint の特徴

- 爆速([ESLint の 50~100 倍速い](https://github.com/oxc-project/bench-javascript-linter?tab=readme-ov-file#oxlint-vs-eslint-v9))
- eslint や eslint プラグインに基づく 480 以上のルールをデフォルトで搭載
- eslint のエコシステムを継承(`.eslintignore`, `.eslintrc.json`, 無効化コメント等)
- ただし型情報 Lint ルールは存在しない
  - [開発中ではあるっぽい？](https://github.com/oxc-project/oxc/issues/2180)
  - oxc ツールチェーンの兄弟分として[ts パーサ](https://oxc.rs/docs/guide/usage/parser.html)も開発されている

## oxlint は実用に耐えうるのか？

例えば oxlint では 、 自分がよく使う以下のプラグインには未対応
（[最新の対応状況](https://github.com/oxc-project/oxc/issues/481)）

- [eslint-plugin-import](https://github.com/import-js/eslint-plugin-import)
- [eslint-plugin-unused-imports](https://github.com/sweepline/eslint-plugin-unused-imports)
- [typescript-eslint](https://typescript-eslint.io/)

こんな時に oxlint では

- oxlint では未実装のルール/プラグインを eslint で実行
- oxlint と eslint で重複する項目を oxlint 側でのみ実施

するように設定することで、(限定的にはなるものの) eslint を高速化できる！

### 設定方法

oxlint および [eslint-plugin-oxlint](https://github.com/oxc-project/eslint-plugin-oxlint) を既存の eslint 環境に追加する

```sh
npm install --save-dev oxlint eslint-plugin-oxlint
```

oxlint の設定ファイルを作成（設定ファイルについては [こちら](https://oxc.rs/docs/guide/usage/linter/config.html)）

```json
// .oxlintrc.json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["import", "typescript", "unicorn"],
  "env": {
    "browser": true
  },
  "settings": {},
  "rules": {
    "eqeqeq": "warn",
    "import/no-cycle": "error"
  },
  "overrides": [
    {
      "files": ["*.test.ts", "*.spec.ts"],
      "rules": {
        "@typescript-eslint/no-explicit-any": "off"
      }
    }
  ]
}
```

eslint 側で import 系の プラグインを整備する。
この時に oxlint プラグインの `buildFromOxlintConfigFile` を使用することで、oxlint 側で実行するルールを eslint 側で off にできる。

```ts
// eslint.config.ts
import * as importPlugin from 'eslint-plugin-import';
import unusedImportsPlugin from 'eslint-plugin-unused-imports';
import tsParser from '@typescript-eslint/parser';
import globals from 'globals';
import tseslint from 'typescript-eslint';
import oxlint from 'eslint-plugin-oxlint';

export default tseslint.config(
  {
    ignores: ['**/build/**', '**/node_modules/**'],
  },
  {
    languageOptions: {
      globals: globals.browser,
      parser: tsParser,
    },
  },
  // oxlint側で使用しているルールをoffにする
  ...oxlint.buildFromOxlintConfigFile('.oxlintrc.json'),
  {
    plugins: { import: importPlugin },
    rules: {
      'import/order': [
        'warn',
        {
          groups: [
            'builtin',
            'external',
            'internal',
            ['parent', 'sibling'],
            'object',
            'type',
            'index',
          ],
          'newlines-between': 'always',
          pathGroupsExcludedImportTypes: ['builtin'],
          alphabetize: { order: 'asc', caseInsensitive: true },
          pathGroups: [
            {
              pattern: '@remix-run/**',
              group: 'external',
              position: 'before',
            },
          ],
        },
      ],
    },
  },
  {
    plugins: { 'unused-imports': unusedImportsPlugin },
    rules: {
      '@typescript-eslint/no-unused-vars': 'off',
      'unused-imports/no-unused-imports': 'error',
      'unused-imports/no-unused-vars': [
        'error',
        {
          vars: 'all',
          varsIgnorePattern: '^_',
          args: 'after-used',
          argsIgnorePattern: '^_',
        },
      ],
    },
  },
);
```

### 実行順序とオート修正

eslint と oxlint を共存させる場合は、制約として oxlint → eslint の順番で実行する必要がある
（参考: [Run it before eslint](https://github.com/oxc-project/eslint-plugin-oxlint?tab=readme-ov-file#run-it-before-eslint)）

```json
{
  "scripts": {
    "lint": "npx oxlint && npx eslint"
  }
}
```

修正まで実施したい場合は、eslint 同様 `--fix` オプションを使用する
pre-commit に組み込むならこんな感じ

```json
"lint-staged": {
  "**/*.{js,mjs,cjs,jsx,ts,mts,cts,tsx,vue,astro,svelte}": [
    "npx oxlint --fix",
    "npx eslint --fix"
  ]
}
```

## 最後に

linter はシフトレフトの要であり、CI や pre-commit など実行する機会は非常に多いと思います。
読んでいただいた方の lint ライフ高速化に少しでも貢献できることを願っています！
