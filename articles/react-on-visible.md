---
title: '[React]「ポップアップが表示されたらinputにfocusする」を正しく書く'
emoji: '👍'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [React, TypeScript]
published: false
---

:::message
「モーダル」「ポップアップ」「ダイアログ」の違いに関しては[こちらの記事](https://rilaks.jp/blog/modal-pop-up-dialog/)を参照してください。
本記事では「ポップアップ」で統一しますが、使い方が間違っていたらすみません。
:::

## 結論

- 「input 要素に focus する」はみなさんご存知の通り`ref.current.focus()`を使う
- `useEffect`はコンポーネントがマウントされたときに実行されるため、「コンポーネントが表示されたら」とは本質的には違う。
- なので `useEffect`に`ref.current.focus()`を直接書くのではなく、
- **`IntersectionObserver`を使って「ポップアップが表示されたら」を表現し、`useEffect`でハンドラの登録・解除を行う。**

## はじめに

「ポップアップが表示されたときに input 要素に focus する」という UI (以下「Focus 機能」)を実装するとします。
ポップアップは独立したコンポーネントであり、「ポップアップを開閉する動作」自体は親コンポーネントが制御しているとしましょう。
この場合、この機能はポップアップコンポーネント内で`useEffect`を使って実装するのが一般的だと思います。

![ポップアップが表示されたらinputにfocusするUI](/images/react-on-visible/popup-focus.gif)

本記事では Focus 機能を 2 つのパターンで実装し、筆者の考える Focus 機能のベストな実装方法を提唱します。

`useEffect`の正しい使い方に関しては、色々な方が記事で説明してくださっているので、そちらを参考にするとより理解が深まると思います。
https://qiita.com/al_tarte/items/9f12f91afb74fe8ecb0e
https://zenn.dev/uhyo/articles/useeffect-taught-by-extremist
https://qiita.com/keiya01/items/fc5c725fed1ec53c24c5
https://zenn.dev/fujiyama/articles/c26acc641c4e30

## Focus 機能追加前のコード

`Popup.tsx`にポップアップの UI を記述し、`main.tsx`でポップアップの開閉状態を管理しています。
簡略化のため送信ボタンなどは省略しています。

```tsx:Popup.tsx
- import { useEffect } from 'react';
+ import { useEffect, useRef } from 'react';

type Props = {
  onClose: () => void;
};

export function Popup({ onClose }: Props): React.ReactNode {
  return (
    <div className="fixed inset-0 w-screen h-screen bg-black/50 z-10 flex justify-center items-center">
      <div className="bg-background p-8 flex flex-col gap-y-4 rounded-lg">
        <h2 className="text-xl">Form</h2>
        <input type="text" name="name" placeholder="Enter your name" className="h-12 rounded-md px-4" />
        <button type="button" onClick={onClose}>
          Close
        </button>
      </div>
    </div>
  );
}
```

```tsx:main.tsx
import { useState } from 'react';
import { Popup } from './Popup';

export function Main() {
  const [showPopup, setShowPopup] = useState(false);

  return (
    <div className="flex justify-center items-center w-screen h-screen">
      <button type="button" onClick={() => setShowPopup(true)}>
        Show Popup
      </button>
      {showPopup && <Popup onClose={() => setShowPopup(false)} />}
    </div>
  );
}
```

## 悪い実装例

では、実際に Focus 機能を追加してみましょう。
まずは悪い実装例を見ていきます。
以下のコードは`useEffect`内で直接`inputRef.current.focus()`を実行しています。
Github Copilot などの AI がよくこのパターンの実装をサジェストしてきますが、これは「ポップアップが表示されたら」ではなく「ポップアップがマウントされたら」を表現しているに過ぎず、本質的ではありません。
実際、ポップアップが`display: none`で DOM ツリーに存在しているとき、それが表示されても input にフォーカスは当たらないことになります。

```diff tsx:Popup.tsx
- import { useEffect } from 'react';
+ import { useEffect, useRef } from 'react';

type Props = {
  onClose: () => void;
};

export function Popup({ onClose }: Props): React.ReactNode {
+  const inputRef = useRef<HTMLInputElement>(null);
+
+  useEffect(() => {
+    if (inputRef.current == null) return;
+    inputRef.current.focus();
+  }, []);

  return (
    <div className="fixed inset-0 w-screen h-screen bg-black/50 z-10 flex justify-center items-center">
      <div className="bg-background p-8 flex flex-col gap-y-4 rounded-lg">
        <h2 className="text-xl">Form</h2>
+         <input ref={inputRef} type="text" name="name" placeholder="Enter your name" className="h-12 rounded-md px-4" />
-         <input type="text" name="name" placeholder="Enter your name" className="h-12 rounded-md px-4" />
        <button type="button" onClick={onClose}>
          Close
        </button>
      </div>
    </div>
  );
}
```

## 良い実装例

```diff tsx:Popup.tsx
- import { useEffect } from 'react';
+ import { useEffect, useRef } from 'react';

type Props = {
  onClose: () => void;
};

export function Popup({ onClose }: Props): React.ReactNode {
+  const inputRef = useRef<HTMLInputElement>(null);
+
+  useEffect(() => {
+    if (inputRef.current === null) return;
+    const handleIntersect = ([entry]: IntersectionObserverEntry[]) => {
+      if (entry?.isIntersecting === true) {
+        inputRef.current?.focus();
+      }
+    };
+    const observer = new IntersectionObserver(handleIntersect);
+    observer.observe(inputRef.current);
+    return () => {
+      observer.disconnect();
+    };
+  }, []);
+
  return (
    <div className="fixed inset-0 w-screen h-screen bg-black/50 z-10 flex justify-center items-center">
      <div className="bg-background p-8 flex flex-col gap-y-4 rounded-lg">
        <h2 className="text-xl">Form</h2>
+         <input ref={inputRef} type="text" name="name" placeholder="Enter your name" className="h-12 rounded-md px-4" />
-         <input type="text" name="name" placeholder="Enter your name" className="h-12 rounded-md px-4" />
        <button type="button" onClick={onClose}>
          Close
        </button>
      </div>
    </div>
  );
}
```
