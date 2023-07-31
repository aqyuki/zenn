---
title: "gonewを使ってみた"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "tool"]
published: true
---

Go 公式が新しいツールである`gonew`を発表していたので少し触ってみました。

# Gonew とは

gonew とは事前に作成しておいたテンプレートをもとに新しい Go のプラジェクトを作成する事のできるツールです。この記事を書いている時点ではまだ機能等を大幅に縮小したプロトタイプのみです。これからのユーザーの反応をもとに k 異能の追加などを行っていくんだと思います。
[参考](https://go.dev/blog/gonew)

# 使ってみる

## インストール

```bash
go install golang.org/x/tools/cmd/gonew@latest
```

でインストールできます。

## 実行

とりあえず公式の用意しているテンプレートをもとに使ってみる。



```bash
gonew golang.org/x/example/hello github.com/aqyuki/hello
```

`gonew`は`gonew <template URL> <Package name>`という形式で実行できます。`<template URL>`の部分に元となるリポジトリの URL を`<Package name>`のところに自分の利用したいパッケージ名を指定します。

## 生成されたファイルを確認

生成されたファイルは以下のとおりです。

- reverse
  - reverse.go
  - reverse_test.go
- go.mod
- hello.go
- LICENSE

これは、[ここ](https://pkg.go.dev/golang.org/x/example/hello)に書かれているものと同じファイルとディレクトリです。また、go.mod ファイルの中は以下のようになっていました。

```:go.mod
module github.com/aqyuki/hello

go 1.19
```

自動的にモジュール名が自動で変更されるのはいいですね。

# まとめ

1. 今回公開されたのはあるリポジトリをもとに新しい Go プロジェクトを開始することのできる`gonew`というツール
2. まだ、プロトタイプのためこれから仕様が変わっていく可能性がある
3. 使ってみた感じ、かなり便利そう
