---
layout: default
title:  "xv6: ブートブロックはどう作られロードされるのか"
lang: jp
image:
    path: /assets/images/dax86-xv6.png
---

# xv6: ブートブロックはどう作られロードされるのか

x86のエミュレーターであるdax86を自作し、xv6を動かす際にもちろんOSの理解が必要となる訳ですが、一番最初のチャレンジはやはりブーティングでした。xv6がいかにブートブロックをビルドし、OSイメージに書き込み、それをマシンがどうメモリーに載せて実行するのか始めから終わりまでを書きました。

## Makefile

xv6はMakefileでブートブロックやカーネルをコンパイル、リンクしOSイメージを作成します。以下に見られる様に`xv6.img`は`bootblock`と`kernel`を依存リストに持ち、`dd`コマンドでファイルにそれらをOSイメージファイルに書き込んでいます。このブロックは複数のファイルを指定した位置に書いているのですが、その詳細はこの記事の後の部分で説明します。

```sh
xv6.img: bootblock kernel
	dd if=/dev/zero of=xv6.img count=10000
	dd if=bootblock of=xv6.img conv=notrunc
	dd if=kernel of=xv6.img seek=1 conv=notrunc
```

## ブートブロックの作成

下記に見られる様に、`bootblock`はプロテクテッドモードに入る為の小さなアセンブリである`bootasm.S`とカーネルのコードをディスクからメモリーにコピーし、そこにジャンプする`bootmain.c`を依存リストに持ちます。

```sh
bootblock: bootasm.S bootmain.c
	$(CC) $(CFLAGS) -fno-pic -O -nostdinc -I. -c bootmain.c
	$(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c bootasm.S
	$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o
	$(OBJDUMP) -S bootblock.o > bootblock.asm
	$(OBJCOPY) -S -O binary -j .text bootblock.o bootblock
	./sign.pl bootblock
```

ここで一歩下がってマシンに電源が入ってどの様に振る舞うのかを考えたいと思います。UEFIではなくBIOSを持った伝統的なコンピューターは電源が入るとPOST(power-on セルフテスト)を行い、オプションROMのコードが走りマルチプロセッサの設定等が行われます。それが完了するとブートディスクの最初のセクタの後部2バイトに、`0xAA55`というデータがあるかをチェックします。

下記は`xxd`コマンドをOSイメージに実行した結果です。セクタ(512バイト)の最後から2バイト目であるアドレス、`0x1FE`にこの該当のデータがあるのがわかります。

```sh
$ xxd xv6.img
...
000001a0: 1518 0001 008d 65f4 5b5e 5f5d c300 0000  ......e.[^_]....
000001b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001f0: 0000 0000 0000 0000 0000 0000 0000 55aa  ..............U.
...
```

実はこのシグネチャーは上記のMakefileの`bootblock`下に見られる様に、`./sign.pl bootblock`というPerlスクリプトで以下の様に書き込まれます。

```perl
$buf .= "\0" x (510-$n);
$buf .= "\x55\xAA";

open(SIG, ">$ARGV[0]") || die "open >$ARGV[0]: $!";
print SIG $buf;
close SIG;
```

さてここまででBIOSがブートできるシグネチャー(MBR: マスターブートレコード)を見つける所まで追って来た訳ですが、この後BIOSは該当セクタをメモリの(多くの場合)`0x7C00`番地にコピーし、そこから実行を開始します。それが上記のMakefileで`$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o`とリンカーにコマンドを指定する理由です。実行部分である`.text`セクションが`0x7C00`とBIOSの開始位置を一致する様に指定されてる訳です。

## ブートローディング

1セクタのスペースは勿論カーネルコードに十分ではなく、上記でメモリーにロードされた1セクタ分のデータは実はカーネルではなくカーネルをディスクからメモリーにコピーする為のコードになっており、それはこのコピー機能を512バイト内で実装することがOS開発者の仕事であるということを意味します。xv6ではカーネルのコードはOSイメージの2セクター目以降に書き込まれ、またそこからロードされます。この部分をディスク作成からロードまで読み切るのは少し手こずりました。

下記の`bootmain.c`でセクタを読む`readseg`のコードでは`offset`の変数がセクタ番号のオフセットであり、`1`がデフォルトで足されています。

```c
void
readseg(uchar* pa, uint count, uint offset)
{
  uchar* epa;

  epa = pa + count;

  // Round down to sector boundary.
  pa -= offset % SECTSIZE;

  // Translate from bytes to sectors; kernel starts at sector 1.
  offset = (offset / SECTSIZE) + 1;

  // If this is too slow, we could read lots of sectors at a time.
  // We'd write more to memory than asked, but it doesn't matter --
  // we load in increasing order.
  for(; pa < epa; pa += SECTSIZE, offset++)
    readsect(pa, offset);
}
```

OSイメージを作成するMakefileではカーネルの書き込みは以下の様に指定されます。

```sh
dd if=kernel of=xv6.img seek=1 conv=notrun
```

ここでは`seek=1`がキーポイントになります。`seek=n`というのは`dd`コマンドのオプションで、何ブロック飛ばして書き込みを始めるかを指定するのに使われます。さらに`dd`コマンドのブロックのデフォルトサイズは512バイトなので、上記のコードは最初の512バイトを飛ばしてカーネルを書き込む様に指示しているということになります。そしてこの飛ばした最初の512バイトというのが先述したカーネルをコピーするブートローディングのコード(`bootblock`)であり、`offset`に足されている`1`はそれを飛ばしてカーネルをコピーする為の数字になっているという訳です。