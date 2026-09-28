---
title: "Go 1.28でstring(int)周りの挙動が変わりそう"
emoji: "✨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## はじめに

2026年09月23日のProposal Review Meetingで珍しいProposalが**Proposal-Accepted**になったので今回はそのProposalについてまとめていこうと思います。

## spec: remove string(int) #3939

今回承認されたProposalは[spec: remove string(int)#3939](https://github.com/golang/go/issues/3939)です。このProposalで提案されているのは整数値から文字列への直接変換を変更し`rune`、`byte`を基底型として持つ場合とuntyped rune constantの場合にのみ文字列へと直接変換できるように制限しようというものです。
たとえば以下のコードはGo 1.27で動作します。

```go
str := string(65)
```

このProposalはこの型変換を制限しようというものであり、2026年09月23日のProposal Review Meetingで承認されました。

## なぜ制限されるのか

上記構文が制限される理由はProposal内で「直感的ではない挙動」と「一貫性の無さを生み出している」の2つが挙げられています。

例えば、以下のコードを実行したとき画面に出力されるのは一体何でしょうか？

```go
str := string(65)
println(str)
```

直感的には `65` と出力されるように思えますが、実際に出力されるのは`A`です。
これは、`string(65)`と呼び出した際に`65`をUnicode コードポイントとして解釈するために発生しています。

また、`string(int)`が引数で指定された数値をUnicode コードポイントとして解釈すると知っていたとしても今度は一貫性のない挙動を示します。例えば

```go
str1 := string(0xD800)
str2 := string("\uD800")
```

上記2つはどちらも同じサロゲート領域の値を示しています。しかし、前者の書き方はコンパイルに成功しますが、有効なUnicode コードポイントではないため`"\uFFFD"`に変換されます。一方後者の書き方はそもそもGoのコンパイラによって拒否され、同じ値を扱っているのにもかかわらず異なる挙動を示してしまっています。

## このProposalによってどのように挙動が変化するのか

このProposalによって整数型から文字列型への型変換を行う場合、基底型が`rune`または`byte`の値、もしくはuntyped rune constantであることが求められます。例えば以下のようになります。

```go
string(rune(65)) // OK // 従来通り
string([]byte{'a'}) // OK // 従来通り
string(int32(65)) // OK // rune = int32 で基底型が同一のためOK
string(uint8(65)) // OK // byte = uint8 で基底型が同一のためOK
string('A') // OK // untyped rune literalはOK

type myRune rune
string(myRune(65)) // OK // 基底型がruneのためOK

string(65) // NG // untyped integer literalはNG
```

## 更新方法

もし既存のコードで整数型から文字列型への型変換を利用している場合、何を必要としているのかによって移行先の書き方が変わります。

```go
// case 1: Unicode コードポイントとして評価したい
var n1 int
str1 := string(rune(n1))

// case 2-1: 数値を文字列にしたい
var n2 int
str2 := strconv.Itoa(n2)

// case 2-2: int64 を変換するなら strconv.FormatInt を使う
var n3 int64
str3 := strconv.FormatInt(n3, 10)
```

また、これらの記法に関しては`go vet`の`stringintconv`で以前からチェックされていたものになるため、`go vet`を実行することでこういった書き方をしている箇所が存在しないか機械的に確認することもできます。

## Go 1 and the Future of Go Programs との兼ね合い

このProposalは明らかにGoの後方互換性を損なう可能性のあるProposalであり、Go 1の後方互換性ポリシー（[Go 1 and the Future of Go Programs](https://go.dev/doc/go1compat)）と衝突する可能性があります。しかしこのProposalではこの課題を以下のようにすることで回避しました。

- モジュールが宣言しているGoのバージョンに応じて挙動を変更する
- すべての整数→文字列の直接変換を禁止せず、必要なユースケースが存在する`byte`や`rune`に関してはこれからも直接の型変換を許可する

Go Modulesを利用するプロジェクトでは、`go.mod`の`go`ディレクティブが、そのモジュールに適用するGo言語バージョンを指定します。そのため、利用しているツールチェーンがGo 1.28以降であっても、`go`ディレクティブがGo 1.27以前であれば、Go 1.27以前の書き方が許容されるようです。
また、実用上必要な変換を残すため、`byte` `rune` からの変換に関してはこれからも言語機能として残り続けます。

## まとめ

このProposalはGo 1.28での導入を目指して現在実装が進められています。そのため順調に進めば来年の2月にはこの変更が世に出ることになりそうです。リリースされたとしてもあまり大きな影響はないでしょうが、よりGoが書きやすく間違いにくくなっていくなら嬉しいなと思います。
