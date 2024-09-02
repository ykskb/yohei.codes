---
layout: default
title: "スレッド、clone、futexとCMPXCHG"
lang: jp
image:
    path: /assets/images/dp-regex.png
---

# スレッド、clone、futexとCMPXCHG

これは、スレッド（もしくはpthread）に関連するトピックについて、自分が過去に断続的に取ったメモのページです。理解を整理するためにまとめました。

## pthreadの内部

[pthreads](https://man7.org/linux/man-pages/man7/pthreads.7.html)は[`clone`](https://man7.org/linux/man-pages/man2/clone.2.html)を呼び出す。これにより`task_struct`が作成される。

> メモ:
>
> * pthread（POSIXスレッド）はlibcの一部。ほとんどのLinuxではglibc（GNU Cライブラリ）によって実装されている。
>
> * ユーザー空間で実装され、`clone()`や[`futex`](https://man7.org/linux/man-pages/man2/futex.2.html)などのシステムコールを通じてカーネルとやり取りする。
>
> * [`fork()`](https://man7.org/linux/man-pages/man2/fork.2.html)も`clone()`を呼び出すが、異なるパラメータでの呼び出しになる。

* `clone()`には次のような異なるフラグが存在する：

    * `CLONE_VM`: メモリ空間（`fork()`はメモリ空間を共有しない）

    * `CLONE_FS`: ファイルシステム

    * `CLONE_FILES`: ファイルディスクリプタ

    * `CLONE_SIGHAND`: シグナルハンドラ

    * `CLONE_THREAD`: スレッド

    * `CLONE_PARENT_SETTID`: 

    * `CLONE_CHILD_CLEARTID`: 

> メモ: libc vs カーネルのシステムコール
>
> カーネルのシステムコールは移植性と安全性のためにlibcが標準インターフェイスとなる。`man 2 [名前]`はカーネルのシステムコールを説明し、`man 3 [名前]`はlibcの関数を説明するが、`man 2`でもライブラリセクションにlibcが表示されることが多い。
>
> libc関数が単なるラッパーか、システムコールの上にさらにロジックを持っているかを調べるためには、その関数の内部や知識が必要。しかし、関数が`man 3`にしか存在しない場合、それはlibcの中でシステムコールをラップしているカスタム関数であることが分かる。

## カーネルの観点

プロセスとスレッドの両方は、カーネル内で`task_struct`として扱われる。

* スケジューリング: 統一タスクスケジューリング
    
    カーネルはプロセスとスレッドを異なる方法でスケジュールしない。

    * プリエンプティブマルチタスキング: OSはスレッド間で中断し、切り替えを行うことができる。（OSはハードウェアのタイマー割り込みを使用。）

    * マルチタスキングの基準:
        
        * タイムスライス（クォンタム）

        * I/O操作

        * 優先順位

    * 完全公平スケジューラ（CFS）がデフォルト。リアルタイムのための他のポリシーとして、`SCHED_FIFO`と`SCHED_RR`なども存在。

* スレッドの状態（レジスタ、プログラムカウンタ、スタックポインタなど）は`スレッド制御ブロック（TCB）`として保存され、プロセスの状態は`プロセス制御ブロック（PCB）`として保存される。

* スレッドとプロセスの違い:

    * スレッドの`task_struct`はプロセスのスレッドグループの一部となる。

    * スレッドの構造体は共有リソースを指す。

## カーネルスケジュールの制御

* CPUアフィニティ: スレッド/プロセスが実行できるCPUコアを決定します。

    スレッドを特定のコアにバインドし、コンテキストスイッチを減少させ、キャッシュ利用効率を向上させる為に使用。

    * [`sched_setaffinity(pid, cpusetsize, mask)`](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html)

    * [`sched_getaffinity(pid, cpusetsize, mask)`](https://linux.die.net/man/2/sched_getaffinity)

* ポリシー: 異なるポリシーを設定できる。

    [`sched_setscheduler(pid, policy, param)`](https://man7.org/linux/man-pages/man2/sched_setscheduler.2.html) と`sched_getscheduler(pid)`

    * `SCHED_OTHER`: デフォルトのLinuxタイムシェアリングスケジューラ

    * `SCHED_FIFO`: 先入れ先出しのリアルタイムスケジューリング

    * `SCHED_RR`: ラウンドロビンのリアルタイムスケジューリング

    * `SCHED_IDLE`: バックグラウンドタスク用の非常に低い優先度

    * `SCHED_BATCH`: バッチ処理ジョブに適した設定

* 優先順位: 優先順位も設定できる。

    [`setpriority(which, who, priority)`](https://linux.die.net/man/2/setpriority) と`getpriority(which, who)`

    * `type`: `PRIO_PROCESS`など

    * `who`: 識別子

    * `priority`: 新しい優先度の値

* コマンド:

    `nice`と`renice`は優先度を変更する。

    * 値は`-20`（最も高い優先度）から`19`（最も低い優先度）の範囲。

* pthread:

    * [`pthread_setschedparam`](https://man7.org/linux/man-pages/man3/pthread_setschedparam.3.html) と`pthread_getschedparam`: スケジューリングポリシーとパラメタ

    * [`pthread_attr_setschedpolicy`](https://man7.org/linux/man-pages/man3/pthread_attr_setschedpolicy.3.html) と`pthread_attr_setschedparam`: スレッドの属性

## ユーザーレベルスレッド

* コルーチン: 実行を一時停止し、後で再開できる関数。Python（async/await）、Kotlin、Luaで一般的。

* ファイバー: コルーチンに似ているが、一般的により汎用的。RubyやC++にライブラリあり。

* `m:n`スレッド: `m`個のユーザーレベルスレッドがそれより少数の`n`個のカーネルスレッドにマッピングされます。

    * 利点: 柔軟性 / `n`を調整可能 / コンテキストスイッチなし / ブロッキング操作を避ける処理可能

    * 欠点: 複雑 / デバッグ / 限定的なOSサポート（LinuxとWindowsは`1:1`スレッドモデルを使用）

    * OSレベル: 単純化そして効率化のためほとんど`1:1`（古いSolarisとGNU Portable Threadsは`m:n`マッピングを提供）

    * 例:

        * Go: `goroutines`は少数のOSスレッドにスケジュールされる。

        * Erlang: BEAM仮想マシンを通じて、軽量プロセスは少数のOSスレッドにスケジュールされる。

        * NodeJS: モデルは違うが、イベントループ（イベント駆動アーキテクチャ）を通じて少数のスレッドで複数のタスクが並行して処理される。

> `1:1`モデルのみの言語
>
> JavaやRustは`m:n`やグリーンスレッドのモデルを言語として提供せず、`1:1`のOSネイティブなスレッドの呼び出しのみを言語としてサポートする。
> 
> それに伴いこれらの言語では`m:n`、グリーンスレッドやファイバーの機能は言語サポート外のライブラリとして実装、提供される。
        
## 同期機構

複数のスレッドがある場合、競合を処理する必要あり。pthreadは`ミューテックス`、`条件変数`、`セマフォ`などの同期プリミティブを提供。

* __Mutex__ (`pthread_mutex_t`): 一度に1つのスレッドのみが特定のコード（メモリ）にアクセスできるようにする。

    * `futex`を使用。
    
    * 所有権がある: ロックの所有者スレッドだけが更新できる。

    * 実装: 内部ステート（カーネルまたはOSがスレッドのblockとwakeupのメカニズムを提供）

* __Condition variable__ (`pthread_cond_t`): 特定の条件が満たされるまでスレッドをブロックする。通常はミューテックスと組み合わせて使用される。

    * 関連するミューテックスを解放し、`futex`を使用して他のスレッドからのシグナルやブロードキャストを待つ。

* __Semaphore__ (`sem_t`): 共有リソースへのアクセスを制御。

    * ミューテックスに似ていて、セマフォカウントがゼロであり、スレッドがそれを減少させようとすると、スレッドは`futex`を使用して待機する。カウントが増加すると、`futex`を使用して待機しているスレッドの1つを起こす。

    * 所有権なし：どのスレッドからでも変更可能。
    
    * 使用例: リソースプール（例: DB接続プール）/ producer/consumerの可用性管理

    - 実装: カウンタ + キュー

## Futex

"Fast userspace mutex"の略。競合が発生した場合のみ、futexはカーネルとやり取りする。

> メモ:
> 
> `futex`はカーネルの関与を最小限に抑えるように設計されているため、一見カーネルの一部ではないように見えるが、実際にはカーネルの構成要素。

* 高速パス: ミューテックスが解放されている→ロックが取得される（ユーザー空間のみ）

* 遅延パス: ミューテックスが解放されていない→スレッドがシステムコールを行い、カーネルはミューテックスが解除されるまでスレッドをスリープさせる。

* `futex`は、スレッドを効率的にスリープさせ、ウェイクアップするメカニズムを提供し、カーネルが待機キューを管理する。

    * `futex_wait`: スレッドがfutex変数（通常は特定のメモリ位置）を待っていることを示すために使用される。条件が満たされていない場合（例: ロックが既に保持されている場合）、カーネルはスレッドをスリープさせる。

    * `futex_wake`: futex変数を待機している1つまたは複数のスレッドを起こすために使用される。ロックが解除されたり条件が変わったりしたとき、futex_wakeを使用して待機中のスレッドに通知し、それによってスレッドが実行を続ける。

* コンペア・アンド・スワップ（CAS）: ロック取得に使用されるアトミック操作。

    * X86/64: `EAX`, `EBX`, `ECX`（汎用レジスタ）または`RAX`, `RBX`, `RCX`（64ビット汎用レジスタ）などを使い、`CMPXCHG destination, source`の命令。

    * Arm: 汎用レジスタ（例: `r0`, `r1`, etc.）上で`LDREX`（ロード専用）および`STREX`（ストア専用）の命令。

        ```asm
        LDREX r0, [address]      ; 'address'の値を'r0'にロード
        CMP r0, old_value        ; ロードした値を'old_value'と比較
        STREXEQ r1, new_value, [address] ; 等しい場合、'new_value'を'address'にアトミックに格納
        ```