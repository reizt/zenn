---
title: '[ESLint] ディレクトリを跨いだimportを検出する'
emoji: '🚨'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [ESLint]
published: true
---

## はじめに

プロジェクト内でディレクトリ構造が複雑になると、特定のディレクトリを跨いだ `import` が発生しがちです。特に大規模なプロジェクトでは、依存関係が散乱し、管理が難しくなる可能性があります。今回は、プロジェクト内でディレクトリを跨いだ `import` を ESLint で検出するプラグインを作成する方法を紹介します。

## 目的

ディレクトリ間の `import` を制御し、特定のスコープ（例: `@shared`）のみ許可することで、依存関係の適切な管理を行います。

```js
// NG
import { bar } from '../someOtherFile';
import { bar } from '../../someOtherDir/foo';
// OK
import { bar } from '../../@shared/foo';
import { bar } from './foo';
import { z } from 'zod';
```

## 実装

まず、ESLint プラグインをプロジェクト内で設定します。このプラグインは、親ディレクトリからの `import` を検出し、それが `@shared` ディレクトリからのものでない場合に警告を出すようにします。

```js
const isValidImport = (value) => {
  const isParent = /^\.\.\//.test(value);
  const isSharedParent = /^(\.\.\/){1,}@shared/.test(value);
  return !isParent || isSharedParent;
};

const noRestrictedImport = {
  meta: {
    type: 'problem',
    docs: {
      description: 'Any modules referenced from subdirectories must be in the @shared scope',
      recommended: 'error',
    },
    messages: {
      restrictedImport: 'Importing from restricted module is not allowed',
    },
    schema: [],
  },
  create(ctx) {
    return {
      ImportDeclaration(node) {
        if (!isValidImport(node.source.value)) {
          ctx.report({
            node,
            messageId: 'restrictedImport',
          });
        }
      },
    };
  },
};

export const localPlugin = {
  meta: {
    name: 'local',
    version: '0.0.0',
  },
  rules: {
    'no-restricted-import': noRestrictedImport,
  },
};
```

## 設定

このプラグインは npm パッケージとして公開するのではなく、プロジェクト内に直接配置し、 `eslint.config.js` にインポートして使用します。

```js
import { localPlugin } from './path/to/plugin';

export default [
  {
    files: ['**/*.ts', '**/*.tsx'],
    plugins: {
      local: localPlugin,
    },
    rules: {
      'local/no-restricted-import': 'error',
    },
  },
];
```

## 動作説明

isValidImport 関数は、 ../ から始まる親ディレクトリへの `import` を検出し、それが `@shared` ディレクトリからであれば許可します。
それ以外の `import` はルールに基づきエラーを報告します。
使用例:
例えば、以下のようなコードで `import` しようとした場合、

```js
import { bar } from '../../someOtherDir/foo';
```

`@shared` ディレクトリからの `import` でないため、ESLint がエラーを報告します。

```
error  Importing from restricted module is not allowed
```

# 終わりに

この ESLint プラグインを使うことで、依存関係の管理が容易になり、プロジェクトのスケーラビリティが向上します。必要に応じて、他のルールや条件を追加することも可能です。
