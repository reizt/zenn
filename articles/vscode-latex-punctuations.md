---
title: '[VSCode Latex] 句読点を理系のやつに自動で変換する'
emoji: '🔁'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [Latex, VSCode]
published: true
---

## モチベーション

普段は句読点に「。」と「、」を使っているので、論文やレポートのためにデフォルトの句読点を「．」と「，」にはしたくない！

## 実現方法

VSCodeの拡張機能 [Run on Save](https://marketplace.visualstudio.com/items?itemName=emeraldwalk.RunOnSave) を使い、
TeXファイルの句読点を「．」と「，」に変換するスクリプトをファイル保存時に実行する。

今回はワークスペース内の任意のTeXファイルに対して変換を行うように設定していきます。

1. VSCodeのワークスペースを設定する

```json
{
  "emeraldwalk.runonsave": {
    "commands": [
      {
        "match": ".tex",
        "isAsync": false,
        "cmd": "node transform-punctuations.cjs ${file}"
      }
    ]
  }
}
```

2. `transform-punctuations.cjs`を実装

```js
const { readFileSync, writeFileSync } = require('node:fs');

const tranformPunctuations = (text) => {
  const replacements = {
    '。': '．',
    '、': '，',
  };
  return text.replace(/[。、]/g, (match) => replacements[match]);
};

const inputFile = process.argv[2];

const text = readFileSync(inputFile, 'utf-8');

const transformedText = tranformPunctuations(text);

if (text !== transformedText) {
  writeFileSync(inputFile, transformedText, 'utf-8');
}
```

## 最後に

おそらくもっといい方法があってもおかしくないのですが、
間に合わせ的に作れるシンプルな方法なのでぜひ試してみてください。
