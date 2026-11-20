---
title: "Go runtime - Entry point"
emoji: "✨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## はじめに

みなさんどうも、しろねです。

最近、時間に余裕があったこともあり、Goのソースコードを読んでいました。そのことを知り合いに話したところ、「技術記事としてまとめてみたら？」と言われました。自分でも、せっかく調べたことをしまっておくのはもったいないと感じたため、今回からシリーズ形式で調査内容をまとめていこうと思います。

第1回となる本記事では「Go runtime - Entry point」というテーマで書いていこうと思います。

## 環境

```bash
go version
# go version go1.27.4 darwin/arm64

go env GOOS
# darwin

go env GOARCH
# arm64
```

## 注意事項

本記事で扱う内容には、Goのコンパイラやリンカーが出力するバイナリに依存する部分が多くあります。

[Go 1 and the Future of Go Programs](https://go.dev/doc/go1compat)でも、互換性が保証されるのは主にソースコードレベルであり、バイナリの同一性までは保証されないことが説明されています。そのため、今後のGoのバージョンアップによって、本記事に記載したコマンドの出力や調査結果が変わる可能性があります。あらかじめご了承ください。

## Goのエントリーポイント

私たちが普段Goでプログラムを書く際、プログラムの実行開始地点として意識するのは `main.main` です。しかし、実際のGoプログラムは `main.main` よりも前の段階から様々な処理（schedulerの初期化やGCの起動など）を行っています。そういった初期化処理を含めたエントリーポイントはどこなのでしょうか？

コンパイラやリンカーのコードを読んで特定してもよいのですが、今回は実際のバイナリを解析することで特定することにしました。

まず初めに簡単なGoのプログラムを用意します。今回はHello Worldを出力するだけのプログラムにしました。

```go:main.go
package main

import "fmt"

func main() {
    fmt.Println("Hello World")
}
```

このプログラムをビルドして実行ファイルを生成します。

```bash
mkdir -p dist
CGO_ENABLED=0 go build -o dist/app.out ./
```

次に、作成されたバイナリを解析していきます。macOSにおける実行ファイルは[Mach-O](https://en.wikipedia.org/wiki/Mach-O)というフォーマットになっています。

Mach-Oの詳細な仕様についてはリンク先のサイトや公式ドキュメントを参照してもらいたいのですが、今回のバイナリでエントリーポイントを特定するために必要なのは次の2つです。

- `LC_MAIN` の `entryoff`
- `__TEXT` セグメントの `vmaddr` および `fileoff`

まずは `LC_MAIN` の情報を確認します。`otool -l ./dist/app.out | rg -A 4 'LC_MAIN'` を実行することで、`LC_MAIN` の情報を取得できます。

```bash
> otool -l ./dist/app.out | rg -A 4 'LC_MAIN'
       cmd LC_MAIN
   cmdsize 24
  entryoff 486576
 stacksize 0
Load command 8
```

次に、`__TEXT` セグメントの `vmaddr` と `fileoff` を確認します。`otool -l ./dist/app.out | rg -A 8 'segname __TEXT'` を実行することで、`__TEXT` セグメントの情報を取得できます。

```bash
> otool -l ./dist/app.out | rg -A 8 'segname __TEXT'
  segname __TEXT
   vmaddr 0x0000000100000000
   vmsize 0x0000000000144000
  fileoff 0
 filesize 1327104
  maxprot 0x00000007
 initprot 0x00000005
   nsects 4
    flags 0x0
```

この結果より、`entryoff` が `486576`、`vmaddr` が `0x0000000100000000`、`fileoff` が `0` であることが分かります。

これらの情報から、次にエントリーポイントの仮想アドレス（Entry VA）を計算します。計算式は次のとおりです。

$$
Entry\ VA = vmaddr + (entryoff - fileoff)
$$

この式に当てはめて計算すると、Entry VAは `0x100076cb0` となります。

あとは、`otool` を使って実行ファイルを逆アセンブルし、出力結果からEntry VAを検索することで、アプリケーションのエントリーポイントを特定できます。

```bash
> otool -arch arm64 -tvV ./dist/app.out | rg -B 1 -A 3 '100076cb0'
__rt0_arm64_darwin:
0000000100076cb0        b       _runtime.rt0_go.abi0
0000000100076cb4        udf     #0x0
0000000100076cb8        udf     #0x0
0000000100076cbc        udf     #0x0
```

今回のバイナリにはシンボル情報が残っているため、エントリーポイントに対応するシンボルが `_rt0_arm64_darwin` であることが分かります。また、先頭の命令を確認すると、`runtime.rt0_go` に分岐していることも分かります。

あとは、GoのGitHubリポジトリで `_rt0_arm64_darwin` を検索すると、`src/runtime/rt0_darwin_arm64.s` に定義されていることが分かりました。

実際にソースコードを確認すると、次のようになっています。

```asm
TEXT _rt0_arm64_darwin(SB),NOSPLIT,$0
    // Darwin puts argc and argv in R0 and R1,
    // so there is no need to go through _rt0_arm64.
    JMP runtime·rt0_go(SB)
```

逆アセンブル結果と同様に、`runtime.rt0_go` へ分岐していることが確認できます。

以上のことから、今回ビルドしたdarwin/arm64向けGoプログラムでは、実行ファイルのエントリーポイントが `_rt0_arm64_darwin` であることが分かりました。

## 余談：リンカー側のソースコードからシンボル名を特定する

ちなみに、リンカーのソースコードを読むと、実行ファイル向けのビルドモード（`BuildModeExe`、`BuildModePIE`）では、以下のようにエントリーポイントとなるシンボル名を組み立てている処理があります。

```go
fmt.Sprintf("_rt0_%s_%s", buildcfg.GOARCH, buildcfg.GOOS)
```

そのため、実際のバイナリを解析しなくても、根気強くリンカー側のコードを読んでいればエントリーポイントを特定できます。（めんどくさいですが）

https://github.com/golang/go/blob/648864b1d8ae9b706f6367cf481fcbd887c19598/src/cmd/link/internal/ld/lib.go#L419-L430

## まとめ

今回の記事では、Go runtime のエントリーポイントについて調査しました。

実際にMach-O形式のバイナリを解析することで、今回ビルドしたdarwin/arm64向けGoプログラムのエントリーポイントが `_rt0_arm64_darwin` であることを確認できました。

今回特定した `_rt0_arm64_darwin` から、どのような処理が行われ、どのようにランタイムが初期化されていくのかについては、次回以降の記事で触れていこうと思います。

興味のある方は、Xなどでリアクションしていただけると幸いです。

今回はここまでです。最後まで読んでいただきありがとうございました。
