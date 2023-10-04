---
title: '[JS] Object, Arrayループの実行速度比較'
emoji: '⏱️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [JavaScript]
published: true
---

## 実行環境

CPU: Apple M1
OS: macOS 13.4.1
Node.js: v20.6.1
Memory: 8GB
Storage: 256GB

## Object

**実装方法**

1. for in

```ts
for (const key in obj)
```

2. Object.keys (for of)

```ts
for (const key of Object.keys(obj))
```

3. Object.keys (forEach)

```ts
Object.keys(obj).forEach((key) => {});
```

4. Object.entries (for of)

```ts
for (const [key, value] of Object.entries(obj))
```

5. Object.entries (forEach)

```ts
Object.entries(obj).forEach(([key, value]) => {});
```

**計測方法**
**N**個のkeyを持つオブジェクトをループする操作を100,000回繰り返すのにかかる時間**T**を計測し、各実装方法で**N**と**T**の関係を考察する

:::details ベンチマークのコード

```ts
const iters = 100_000;

const keys = 1;
const obj = {};
for (let i = 0; i < keys; i++) {
  obj[`key${i}`] = i;
}

console.time('for in');
for (let i = 0; i < iters; i++) {
  for (const key in obj) {
    obj[key] += 1;
  }
}
console.timeEnd('for in');

console.time('Object.keys (for of)');
for (let i = 0; i < iters; i++) {
  for (const key of Object.keys(obj)) {
    obj[key] += 1;
  }
}
console.timeEnd('Object.keys (for of)');

console.time('Object.keys (forEach)');
for (let i = 0; i < iters; i++) {
  Object.keys(obj).forEach((key) => {
    obj[key] += 1;
  });
}
console.timeEnd('Object.keys (forEach)');

console.time('Object.entries (for of)');
for (let i = 0; i < iters; i++) {
  for (const [key, value] of Object.entries(obj)) {
    obj[key] += 1;
  }
}
console.timeEnd('Object.entries (for of)');

console.time('Object.entries (forEach)');
for (let i = 0; i < iters; i++) {
  Object.entries(obj).forEach(([key, value]) => {
    obj[key] += 1;
  });
}
console.timeEnd('Object.entries (forEach)');
```

:::

**結果**
![](/images/js-loop-bench/object-bench.png)

:::details 詳細な時間
Keyの数: 1

```
for in: 1.479ms
Object.keys (for of): 2.6ms
Object.keys (forEach): 1.805ms
Object.entries (for of): 4.592ms
Object.entries (forEach): 5.393ms
```

Keyの数: 10

```
for in: 1.923ms
Object.keys (for of): 3.655ms
Object.keys (forEach): 2.985ms
Object.entries (for of): 10.569ms
Object.entries (forEach): 10.512ms
```

Keyの数: 100

```
for in: 220.766ms
Object.keys (for of): 163.254ms
Object.keys (forEach): 160.879ms
Object.entries (for of): 1.073s
Object.entries (forEach): 1.135s
```

Keyの数: 1000

```
for in: 2.716s
Object.keys (for of): 1.819s
Object.keys (forEach): 1.830s
Object.entries (for of): 11.452s
Object.entries (forEach): 11.464s
```

:::

## Array

**実装方法**

1. for let

```ts
for (let j = 0; j < arr.length; j++)
```

2. for of

```ts
for (const item of arr)
```

3. for of .entries

```ts
for (const [index, item] of arr.entries())
```

4. for of .keys

```ts
for (const index of arr.keys())
```

5. forEach

```ts
arr.forEach((item, index) => {}))
```

**計測方法**
**N**個の要素を持つ配列をループする操作を100,000回繰り返すのにかかる時間**T**を計測し、各実装方法で**N**と**T**の関係を考察する

**結果**
![](/images/js-loop-bench/array-bench.png)

:::details 詳細な時間

要素数: 1

```
for let: 2.958ms
for of: 3.672ms
for of .entries: 9.126ms
for of .keys: 3.208ms
forEach: 3.523ms
```

要素数: 10

```
for let: 5.969ms
for of: 9.251ms
for of .entries: 53.897ms
for of .keys: 8.947ms
forEach: 7.245ms
```

要素数: 100

```
for let: 34.301ms
for of: 72.622ms
for of .entries: 513.946ms
for of .keys: 74.407ms
forEach: 35.735ms
```

要素数: 1000

```
for let: 326.696ms
for of: 640.936ms
for of .entries: 4.796s
for of .keys: 644.261ms
forEach: 340.525ms
```

:::
