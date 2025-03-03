---
title: 'oxlint と eslint を共存させるしかねえ'
emoji: '🐶'
type: 'tech'
topics: ['typescript', 'nodejs', 'react', 'eslint', 'oxlint']
published: false
---

rust 製爆速 linter こと oxlint を eslint と共存させることで、爆速 lint 環境を作る

https://oxc.rs/docs/guide/usage/linter.html

## oxlint の特徴

https://oxc.rs/docs/guide/usage/linter.html#features

- 爆速([ESLint の 50~100 倍速い](https://github.com/oxc-project/bench-javascript-linter?tab=readme-ov-file#oxlint-vs-eslint-v9))
- eslint や プラグインに基づく 480 以上のルールをデフォルトで搭載
- eslint のエコシステムを継承(.eslintignore, .eslintrc.json, lint 無効化コメント)

## eslint との共存

例えば、 自分がよく使う以下の import 系の eslint プラグインは、oxlint では 2025 年 3 月時点で未対応（※最新の対応状況は [GitHub Issue](https://github.com/oxc-project/oxc/issues/481) へ）

- [eslint-plugin-import](https://github.com/import-js/eslint-plugin-import)
- [eslint-plugin-unused-imports](https://github.com/sweepline/eslint-plugin-unused-imports)

こんな時に oxlint では

- oxlint では未実装のルール/プラグインを eslint で実行
- oxlint と eslint で重複する項目を oxlint 側でのみ実施

するように設定できる

## 設定ファイルの共存

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

eslint 側で import 系の プラグインを整備する

この時に oxlint プラグインの `buildFromOxlintConfigFile` を使用することで、oxlint 側で実行するルールを eslint 側で off にできる（=重複排除）

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
  }
);
```

## 実行設定

eslint と oxlint を共存させる場合は、制約として oxlint -> eslint の順番で実行する必要がある（参考: [Run it before eslint](https://github.com/oxc-project/eslint-plugin-oxlint?tab=readme-ov-file#run-it-before-eslint)）

```json
{
  "scripts": {
    "lint": "npx oxlint && npx eslint"
  }
}
```

修正まで実施したい場合は、eslint 同様 `--fix` オプションを使用する（参考: [Oxlint - Automatic Fixes](https://oxc.rs/docs/guide/usage/linter/automatic-fixes.html)）

pre-commit に組み込むならこんな感じ（lint-staged を使用した pre-commit については[こちら](https://zenn.dev/risu729/articles/latest-husky-lint-staged)を参考）

```json
"lint-staged": {
  "**/*.{js,mjs,cjs,jsx,ts,mts,cts,tsx,vue,astro,svelte}": [
    "npx oxlint --fix",
    "npx eslint --fix"
  ]
}
```
