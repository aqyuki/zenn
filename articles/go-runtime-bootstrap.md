---
title: "Go runtime - Bootstrap"
emoji: "✨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

## はじめに

みなさんどうも、しろねです。

Goランタイムについて調べたことをまとめていく「Go runtime」シリーズ。第2回となる今回は「Go runtime - Bootstrap」をテーマに書いていこうと思います。

:::message
本記事の内容は調査時点での最新コミット [0351fe5](https://github.com/golang/go/commit/0351fe5c9795a37afd08c3c7568abd3186c629b4) を元に作成しています。
今後の開発により、これらの処理に変更が加えられる可能性があるためあらかじめご了承ください。
:::

## 環境

```bash
> go version
go version go1.27.4 darwin/arm64

> go env GOOS
darwin

> go env GOARCH
arm64
```

## 登場する用語について軽く紹介

詳しくは自身で調べるか、今後の記事での解説を待っていただきたいのですが、以下の内容を理解していれば、本記事を読み進められると思います。

- G : Goランタイムにおいてgoroutineを表す
- M : GoランタイムにおいてOSのスレッドを表す
- P : Goランタイムにおいて仮想CPU（処理の実行権）を表す。Gの実行キューなどを持つ。
- g0 : Mに紐づく特殊なG

## Bootstrapの流れ

Goのプログラムが起動するまでの流れですが、darwin/arm64環境の場合、`rt0_arm64_darwin.s`の`_rt0_darwin_arm64`から`asm_arm64.s`の`runtime·rt0_go`にかけてこれらのBootstrapが行われます。
実際には`rt0_arm64_darwin.s`の`_rt0_darwin_arm64`では`asm_arm64.s`の`runtime·rt0_go`を呼び出すだけなので実質`asm_arm64.s`の`runtime·rt0_go`が次に示すBootstrapを担っています。
Bootstrapの流れは以下のとおりです。

1. `argc`/`argv`をメモリに退避する
1. 必要に応じてTLS（Thread Local Storage）を初期化する
1. `runtime.g0`を初期化する
1. `runtime.m0`を初期化する
1. `runtime.check`を呼び出す
1. `argc`/`argv`をスタックに復帰させて`runtime.args`を呼び出す
1. `runtime.osinit`を呼び出す
1. `runtime.schedinit`を呼び出す
1. `runtime.main`を実行するgoroutineを作成
1. `runtime.mstart`を呼び出してgoroutineの処理を開始する

まず最初に、レジスタ経由で渡された`argc`と`argv`をメモリに退避します。darwin/arm64の場合、R0レジスタに`argc`が、R1レジスタに`argv`が設定された状態でプログラムが起動します。ただし、R0およびR1は汎用レジスタであり今後の初期化処理で利用されるため、メモリ上に領域を確保して退避しています。

```asm:asm_arm64.s
	SUB	$32, RSP
	MOVW	R0, 8(RSP) // argc
	MOVD	R1, 16(RSP) // argv
```

その後、必要に応じてTLSの初期化を行います。TLSとはThread Local Storageの略でOSスレッドごとに固有のデータを保存するための仕組みです。Goにおいては現在実行中のGを特定する際にも利用されます。

```asm:asm_arm64.s
#ifdef TLS_darwin
	// Initialize TLS.
	MOVD	ZR, g // clear g, make sure it's not junk.
	SUB	$32, RSP
	MRS_TPIDR_R0
	AND	$~7, R0
	MOVD	R0, 16(RSP)             // arg2: TLS base
	MOVD	$runtime·tls_g(SB), R2
	MOVD	R2, 8(RSP)              // arg1: &tlsg
	BL	·tlsinit(SB)
	ADD	$32, RSP
#endif
```

TLSの初期化が完了したら、`runtime.g0`および`runtime.m0`の初期化を行います。
`runtime.g0`と`runtime.m0`はともにruntimeパッケージに定義されたグローバル変数です。これらはGoランタイムのBootstrapに利用され、初期化完了後も`runtime.g0`は`runtime.m0`のg0として、`runtime.m0`は通常のMと同様に動作します。

```asm:asm_arm64.s
MOVD	$runtime·g0(SB), g
MOVD	RSP, R7
MOVD	$(-64*1024)(R7), R0
MOVD	R0, g_stackguard0(g)
MOVD	R0, g_stackguard1(g)
MOVD	R0, (g_stack+stack_lo)(g)
MOVD	R7, (g_stack+stack_hi)(g)

// 中略 : 省略した箇所ではcgoを利用する際に必要となる初期化処理を行っている

BL	runtime·save_g(SB)
// update stackguard after _cgo_init
MOVD	(g_stack+stack_lo)(g), R0
ADD	$const_stackGuard, R0
MOVD	R0, g_stackguard0(g)
MOVD	R0, g_stackguard1(g)

// set the per-goroutine and per-mach "registers"
MOVD	$runtime·m0(SB), R0

// save m->g0 = g0
MOVD	g, m_g0(R0)
// save m0 to g0->m
MOVD	R0, g_m(g)
```

`runtime.g0`と`runtime.m0`の初期化が完了した後は、`runtime.check`を呼び出し、Goランタイムが正常に動作するために必要な前提条件が満たされているかを検証します。

```asm:asm_arm64.s
BL	runtime·check(SB)
```

その後、`runtime.args`、`runtime.osinit`、`runtime.schedinit`を順に実行して必要な初期化処理を行っていきます。それぞれの処理が行っている内容は以下のとおりです。

- `runtime.args` : `argc`と`argv`をプログラムから利用できるように保存する。パース等は行わない
- `runtime.osinit` : OSに依存するGoランタイムの設定を行う
- `runtime.schedinit` : スケジューラーやメモリアロケーター、GCなどGoランタイムの基本的な初期化処理を行う。また、ここで`runtime.m0`とPの関連付けも行われる。

```asm:asm_arm64.s
MOVW	8(RSP), R0	// copy argc
MOVW	R0, -8(RSP)
MOVD	16(RSP), R0		// copy argv
MOVD	R0, 0(RSP)
BL	runtime·args(SB)
BL	runtime·osinit(SB)
BL	runtime·schedinit(SB)
```

必要な初期化を済ませた後、`runtime.main`を実行するgoroutine (G)を作成します。ここで作られたGは`runtime.m0`に紐づくPの`runnext`、`runq`に入ります。通常はrunnextに登録され、Mが開始した際に優先的に実行されます。
その後、`runtime.mstart`を実行してGoの実行に入ります。ただし、この時点ではまだ一部のコンポーネントが動作しておらずGoランタイムの初期化は完了していません。
残りの初期化は`runtime.main`が実行されるタイミングで行われます。

```asm:asm_arm64.s
// create a new goroutine to start program
MOVD	$runtime·mainPC(SB), R0		// entry
SUB	$16, RSP
MOVD	R0, 8(RSP) // arg
MOVD	$0, 0(RSP) // dummy LR
BL	runtime·newproc(SB)
ADD	$16, RSP

// start this M
BL	runtime·mstart(SB)
UNDEF
```

## `runtime.main`の流れ

`runtime.main`はBootstrapでは行われなかったGoランタイムの初期化や関連機能の有効化を行った後に、ユーザーが定義した`main.main`の呼び出し、そしてプログラム終了時の後片付けを行っています。
`runtime.main`の流れは次のとおりです。

1. CPUのビット数に応じてスタックの最大サイズを設定する
1. `sysmon`を利用する環境では`sysmon`を新しいMで実行する
1. Goランタイムの初期化を行っているGとMが切り替わらないように固定
1. `runtime.doInit(runtime_inittasks)`を実行し、runtimeパッケージが依存する`init()`関数を呼び出す
1. GCの有効化
1. GOMAXPROCSの自動更新を有効化
1. cgo関連の初期化
1. 各パッケージの初期化タスクを依存関係に応じた順番で実行する
1. Goランタイムの状態を管理する変数を更新した後、GとMの固定を解除
1. c-archiveもしくはc-sharedとしてビルドされていた場合はここで処理が終了する。
1. ユーザーの定義した`main.main`を呼び出す
1. ASANが有効かつcgoを利用している場合はC側のメモリリークを検査する。
1. 他のgoroutineでpanicが発生している場合は、その処理が進むように実行機会を譲り、終了フックなどを実行してプログラムを終了する。

まず最初にCPUのビット数に応じてスタックの最大サイズを設定します。64-bit環境の場合は1 GB、32-bit環境の場合は250 MBとなっています。

```go:proc.go
// Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
// Using decimal instead of binary GB and MB because
// they look nicer in the stack overflow failure message.
if goarch.PtrSize == 8 {
  maxstacksize = 1000000000
} else {
  maxstacksize = 250000000
}
```

スタックの最大サイズを設定した後、Mの新規開始を有効化し、`sysmon`を利用する環境では`sysmon`を新しいMで実行します。

```go:proc.go
// Allow newproc to start new Ms.
mainStarted = true

if haveSysmon {
  systemstack(func() {
    newm(sysmon, nil, -1)
  })
}
```

また、このタイミングで`lockOSThread`を呼び出して実行中のGとMを固定します。これにより、Gは現在実行されているMに固定されます。ロックが解除されるまで、そのGが別のMで実行されることはなく、固定先のMも原則として別のGを実行しなくなります。

```go:proc.go
// Lock the main goroutine onto this, the main OS thread,
// during initialization. Most programs won't care, but a few
// do require certain calls to be made by the main thread.
// Those can arrange for main.main to run in the main thread
// by calling runtime.LockOSThread during initialization
// to preserve the lock.
lockOSThread()
```

`lockOSThread`を呼び出した後、`doInit`を呼び出してGoランタイムが依存する初期化タスクを実行します。`runtime_inittasks`はビルド時にリンカーによって静的リンクされます。
`runtime_inittasks`を実行した後、GCを有効化し、GOMAXPROCSの自動更新を有効化します。

```go:proc.go
doInit(runtime_inittasks) // Must be before defer.

// Defer unlock so that runtime.Goexit during init does the unlock too.
needUnlock := true
defer func() {
  if needUnlock {
    unlockOSThread()
  }
}()

gcenable()
defaultGOMAXPROCSUpdateEnable() // don't STW before runtime initialized.
```

また、cgoが有効な場合、このタイミングでもcgo関連の初期化が行われます。

```go:proc.go
if iscgo {
  if _cgo_pthread_key_created == nil {
    throw("_cgo_pthread_key_created missing")
  }

  if GOOS != "windows" {
    if _cgo_thread_start == nil && (!goexperiment.CgoBindings || cgoThreadStart == nil) {
      throw("_cgo_thread_start missing")
    }
    if _cgo_setenv == nil && (!goexperiment.CgoBindings || cgoSetenv == nil) {
      throw("_cgo_setenv missing")
    }
    if _cgo_unsetenv == nil && (!goexperiment.CgoBindings || cgoUnsetenv == nil) {
      throw("_cgo_unsetenv missing")
    }
  }
  if _cgo_notify_runtime_init_done == nil && (!goexperiment.CgoBindings || cgoNotifyRuntimeInitDone == nil) {
    throw("_cgo_notify_runtime_init_done missing")
  }

  // Set the x_crosscall2_ptr C function pointer variable point to crosscall2.
  if set_crosscall2 == nil {
    throw("set_crosscall2 missing")
  }
  set_crosscall2()

  // Start the template thread in case we enter Go from
  // a C-created thread and need to create a new thread.
  startTemplateThread()
  if goexperiment.CgoBindings && cgoNotifyRuntimeInitDone != nil {
    cgoNotifyRuntimeInitDone()
  } else {
    cgocall(_cgo_notify_runtime_init_done, nil)
  }
}
```

その後、各パッケージの初期化タスクをモジュール間の依存関係などを考慮した順番で実施していきます。これにはグローバル変数の初期化や`init()`関数の実行などが含まれています。

```go:proc.go
// Run the initializing tasks. Depending on build mode this
// list can arrive a few different ways, but it will always
// contain the init tasks computed by the linker for all the
// packages in the program (excluding those added at runtime
// by package plugin). Run through the modules in dependency
// order (the order they are initialized by the dynamic
// loader, i.e. they are added to the moduledata linked list).
last := lastmoduledatap // grab before loop starts. Any added modules after this point will do their own doInit calls.
for m := &firstmoduledata; true; m = m.next {
  doInit(m.inittasks)
  if m == last {
    break
  }
}
```

c-archiveもしくはc-sharedとしてビルドされていた場合は、ここで`runtime.main`の処理を終了します。これ以降は実行ファイルとしてビルドされている際に実行される処理です。
これまでの処理でGoランタイムの初期化は完了したため、ここでユーザーの定義した`main.main`を呼び出します。

```go:proc.go
fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
fn()
```

`main.main`の終了後、ASANが有効で、実際にcgoの呼び出しを利用した場合、C側のメモリリークを検査します。ただし、`lsandoleakcheck`は呼び出し元に戻ってこない可能性があるため、事前に`runExitHooks`を実行しておきます。

```go:proc.go
// Check for C memory leaks if using ASAN and we've made cgo calls,
// or if we are running as a library in a C program.
// We always make one cgo call, above, to notify_runtime_init_done,
// so we ignore that one.
// No point in leak checking if no cgo calls, since leak checking
// just looks for objects allocated using malloc and friends.
// Just checking iscgo doesn't help because asan implies iscgo.
exitHooksRun := false
if asanenabled && (isarchive || islibrary || NumCgoCall() > 1) {
  runExitHooks(0) // lsandoleakcheck may not return
  exitHooksRun = true
  lsandoleakcheck()
}
```

その後、他のgoroutineでpanicが発生している場合は、スタックトレースの出力などが完了できるよう、必要に応じて処理を譲ります。
最後に、終了時フックなどを呼び出してプログラムを終了します。

```go:proc.go
// Make racy client program work: if panicking on
// another goroutine at the same time as main returns,
// let the other goroutine finish printing the panic trace.
// Once it does, it will exit. See issues 3934 and 20018.
if runningPanicDefers.Load() != 0 {
  // Running deferred functions should not take long.
  for c := 0; c < 1000; c++ {
    if runningPanicDefers.Load() == 0 {
      break
    }
    Gosched()
  }
}
if panicking.Load() != 0 {
  gopark(nil, nil, waitReasonPanicWait, traceBlockForever, 1)
}
if !exitHooksRun {
  runExitHooks(0)
}
if raceenabled {
  racefini() // does not return
}

exit(0)
for {
  var x *int32
  *x = 0
}
```

## まとめ

今回の記事では、GoランタイムのBootstrapをテーマに扱いました。この記事がGoランタイムのライフサイクルを理解する助けになれば幸いです。

次回以降の記事ではGoランタイムのスケジューラーやGC・メモリアロケーターについてまとめていこうと思っています。興味のある方は、Xなどでリアクションをしていただけるとうれしいです。

今回も最後まで読んでいただきありがとうございました。
