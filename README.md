# cva の型チェックとエラー解消

## 背景

- `cva` 関数は、スタイル（クラス名）を条件に応じて切り替えるための関数

## 問題点

- 設定オブジェクトの型が適切に推論されず、 `Config<unknown>` として扱われる

## 解決策

- `cva` に対してジェネリック型引数として `tyoeof variants` を明示的に渡すことで、設定オブジェクトの正確な型情報を伝え、型チェックを厳密に行うようにした
- 具体例:

```ts
const buttonVariants = cva<typeof variants>(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50 [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
  {
    variants,
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
);
```

## 学び
- ジェネリック型引数を明示することで、`cva` に渡す設定オブジェクトの型安全性が向上し、意図しない値やキーのミスをコンパイル時に防ぐことができる