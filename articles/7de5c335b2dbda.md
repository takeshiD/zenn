---
title: "Arch Linuxメモ パーティション MBR"
emoji: "📚"
type: "tech"
topics: ["linux", "arch"]
published: true
---

Arch Linuxを触ったときに調べたことをメモしていきます。

# パーティション
あまり慣れてないので調べながら進めます。
現在のArch Linuxの起動環境はUEFIです。
VirtualBox仮想ストレージは20GBで作成してあります。

## デバイスの確認
`lsblk -l`か`fdisk -l`で確認が可能です。

```sh
lsblk -l
```
```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 653.4M  1 loop /run/archiso/airootfs
sda      8:0    0    20G  0 disk
sr0     11:0    1 807.3M  0 rom  /run/archiso/bootmnt
```
たぶんloop0とsr0はArchLinuxのisoイメージ自体を指しているのかな？と思います。
sdaはVirtualBox仮想ストレージです。

次に`fdisk -l`です。
```sh
fdisk -l
```
```sh
Disk /dev/sda: 20 GiB, 21474836480 byutes, 41943040 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512bytes
I/O size (minimum/optimal): 512 bytes / 512bytes

Disk /dev/loo0: 653.41 MiB, 685150208 byutes, 1338184 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512bytes
I/O size (minimum/optimal): 512 bytes / 512bytes
```
`lsblk`と表示する内容の趣旨が違うように感じます。`lsblk`はブロックデバイスの表示に重きを置いているのでしょうか。

## パーティションの基礎

パーティションとは一つのストレージを分割して利用出来るようにするための仕組みです。  
区切る意味があるのかと思いそうですが、例えば **OS起動用プログラム** を普通のデータと同じようにマウスポチポチで消せたり編集出来たりするとOSが起動しなくなるリスクがあります。  
最低限、OS起動用プログラムのパーティションと、データ格納用のパーティションを作成するようです。

ではパーティションで分割するための仕組みの話です。ストレージのどこからどこがOS起動用で、どこがデータ格納用なのかを管理する必要があります。
大きく次の2種の管理方法が有名です。

1. マスターブートレコード(MBR, 1980年前半～)
2. GUIDパーティション(GPT, 1990年後半～)

MBRが古くから使われているもので、最近ではGPTが主流のようです。
MBRについて簡単に見ていきます。

### マスターブートレコード(MBR)

ストレージのパーティション管理方法をMBRにすると、ストレージの先頭512Byteが次のような構造になります。

| 開始アドレス | 終端アドレス | 内容  | サイズ |
| ----------- | ----------- | ---- | ------ |
| 0x0000  | 0x01bd | ブートストラップローダ | 446Byte |
| 0x01be  | 0x01cd | 第1パーティションテーブル | 16Byte |
| 0x01ce  | 0x01dd | 第2パーティションテーブル | 16Byte |
| 0x01de  | 0x01ed | 第3パーティションテーブル | 16Byte |
| 0x01ee  | 0x01fd | 第4パーティションテーブル | 16Byte |
| 0x01fe  | 0x01ff | ブートシグネチャ(0xAA55) | 2Byte |
||| 合計 | 512Byte | 

ブートストラップローダはブート可能なフラグが立っているパーティションを探して、そのパーティションを読み込み起動させる役割を持っています。


お試しでMBRを作成して中身を見てみましょう。
```sh
$ parted /dev/sda
GNU Parted 3.4
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted)
```
`mklabel msdos` で/dev/sdaがMBRに初期化されます。終えたら`quit`でpartedを抜けます。
```sh
(parted) mklabel msdos
(parted) quit
```
本当に作成されたのか`hexdump`で/dev/sdaを見てみます。ストレージの先頭512Byteだけ読み込みます。

```sh
hexdump /dev/sda -s 0 -n 512 -v
```
:::details hexdumpのオプション
* `-s` : 開始アドレス　未指定の場合は0になる
* `-n` : 読み込み長さ(Byte) 今回は512Byteを読み込み
* `-v` : 表示を省略しない指定。未指定だと同じ値を`*`で省略される。
:::

```:hexdump出力結果
0000000 b8fa 1000 d08e 00bc b8b0 0000 d88e c08e
0000010 befb 7c00 00bf b906 0200 a4f3 21ea 0006
0000020 be00 07be 0438 0b75 c683 8110 fefe 7507
0000030 ebf3 b416 b002 bb01 7c00 80b2 748a 8b01
0000040 024c 13cd 00ea 007c eb00 00fe 0000 0000
0000050 0000 0000 0000 0000 0000 0000 0000 0000
0000060 0000 0000 0000 0000 0000 0000 0000 0000
0000070 0000 0000 0000 0000 0000 0000 0000 0000
0000080 0000 0000 0000 0000 0000 0000 0000 0000
0000090 0000 0000 0000 0000 0000 0000 0000 0000
00000a0 0000 0000 0000 0000 0000 0000 0000 0000
00000b0 0000 0000 0000 0000 0000 0000 0000 0000
00000c0 0000 0000 0000 0000 0000 0000 0000 0000
00000d0 0000 0000 0000 0000 0000 0000 0000 0000
00000e0 0000 0000 0000 0000 0000 0000 0000 0000
00000f0 0000 0000 0000 0000 0000 0000 0000 0000
0000100 0000 0000 0000 0000 0000 0000 0000 0000
0000110 0000 0000 0000 0000 0000 0000 0000 0000
0000120 0000 0000 0000 0000 0000 0000 0000 0000
0000130 0000 0000 0000 0000 0000 0000 0000 0000
0000140 0000 0000 0000 0000 0000 0000 0000 0000
0000150 0000 0000 0000 0000 0000 0000 0000 0000
0000160 0000 0000 0000 0000 0000 0000 0000 0000
0000170 0000 0000 0000 0000 0000 0000 0000 0000
0000180 0000 0000 0000 0000 0000 0000 0000 0000
0000190 0000 0000 0000 0000 0000 0000 0000 0000
00001a0 0000 0000 0000 0000 0000 0000 0000 0000
00001b0 0000 0000 0000 0000 ed4c a356 0000 0000
00001c0 0000 0000 0000 0000 0000 0000 0000 0000
00001d0 0000 0000 0000 0000 0000 0000 0000 0000
00001e0 0000 0000 0000 0000 0000 0000 0000 0000
00001f0 0000 0000 0000 0000 0000 0000 0000 aa55
0000200
```

0000で2byte、1行で16byteになります。

細かく区切って見てみると
```:ブートストラップローダ(446Byte)
0000000 b8fa 1000 d08e 00bc b8b0 0000 d88e c08e
0000010 befb 7c00 00bf b906 0200 a4f3 21ea 0006
0000020 be00 07be 0438 0b75 c683 8110 fefe 7507
0000030 ebf3 b416 b002 bb01 7c00 80b2 748a 8b01
0000040 024c 13cd 00ea 007c eb00 00fe 0000 0000
0000050 0000 0000 0000 0000 0000 0000 0000 0000
0000060 0000 0000 0000 0000 0000 0000 0000 0000
0000070 0000 0000 0000 0000 0000 0000 0000 0000
0000080 0000 0000 0000 0000 0000 0000 0000 0000
0000090 0000 0000 0000 0000 0000 0000 0000 0000
00000a0 0000 0000 0000 0000 0000 0000 0000 0000
00000b0 0000 0000 0000 0000 0000 0000 0000 0000
00000c0 0000 0000 0000 0000 0000 0000 0000 0000
00000d0 0000 0000 0000 0000 0000 0000 0000 0000
00000e0 0000 0000 0000 0000 0000 0000 0000 0000
00000f0 0000 0000 0000 0000 0000 0000 0000 0000
0000100 0000 0000 0000 0000 0000 0000 0000 0000
0000110 0000 0000 0000 0000 0000 0000 0000 0000
0000120 0000 0000 0000 0000 0000 0000 0000 0000
0000130 0000 0000 0000 0000 0000 0000 0000 0000
0000140 0000 0000 0000 0000 0000 0000 0000 0000
0000150 0000 0000 0000 0000 0000 0000 0000 0000
0000160 0000 0000 0000 0000 0000 0000 0000 0000
0000170 0000 0000 0000 0000 0000 0000 0000 0000
0000180 0000 0000 0000 0000 0000 0000 0000 0000
0000190 0000 0000 0000 0000 0000 0000 0000 0000
00001a0 0000 0000 0000 0000 0000 0000 0000 0000
00001b0 0000 0000 0000 0000 ed4c a356 0000
```
ここは機械語になっているようです。00001b0の最後2Byteは第1パーティションなので含みません。

```:第1パーティションテーブル
00001b0                                    0000
00001c0 0000 0000 0000 0000 0000 0000 0000
```
```:第2パーティションテーブル
00001c0                                    0000
00001d0 0000 0000 0000 0000 0000 0000 0000
```
```:第3パーティションテーブル
00001d0                                    0000
00001e0 0000 0000 0000 0000 0000 0000 0000
```
```:第4パーティションテーブル
00001e0                                    0000
00001f0 0000 0000 0000 0000 0000 0000 0000
```
```:ブートシグネチャ
00001f0                                    aa55
```

パーティションテーブルは全て0ですね。まだ作成されていないからでしょうか。

#### パーティションテーブルを作成する 
試しに第1パーティションだけ作成してみます。1MiB始まりの100MiB終わりでテキトーな大きさにしておきます。
partedに入ったら`mkpart primary ext4 1MiB 100MiB`と入力して実行終えたらquitで抜けます。

```sh
$ parted /dev/sda
GNU Parted 3.4
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mkpart primary ext4 1MiB 100MiB
(parted) quit
```

同じように`hexdump`で確認してみます。
全部見る必要は無いので第1パーティションテーブルの部分だけ見てみます。
```
hexdump /dev/sda -s 0 -n 512
```
```:第1パーティション
00001b0                                    0400
00001c0 0401 9083 9142 0800 0000 1800 0003
```
おお、変わりましたね。この表示はリトルエンディアンになっているので注意してください。

リトルエンディアンに注意しながら読んでみると、パーティションテーブルは次のような構造になっています。

| アドレスオフセット | 内容 | サイズ | 今回の値(hex) | 解釈 |
|---|---|---|---|---|
| 0x00 | ブートフラグ | 1Byte | 00 | ブート不可 |
| 0x01 | パーティション開始セクタ(CHS方式) | 3Byte | 04 01 04 |  | 
| 0x04 | パーティション識別子 | 1Byte | 83 | etx4 |
| 0x05 | パーティション終了セクタ(CHS方式) | 3Byte | 90 42 91 | |
| 0x08 | パーティション開始セクタ(LBA方式) | 4Byte | 00 00 08 00 | 2048セクタ目 |
| 0x0c | パーティション全セクタ数 | 4Byte | 00 03 18 00 | 202752セクタ分 |

パーティション全セクタ数をMiBに計算し直すと、MBRの場合セクタサイズは512Byteで固定なので次のようになります。

202752 Sector * 512Byte = 103,809,024 Byte
103,809,024 Byte / 1024^2 = 99 MiB

1MiBから100MiBでサイズ指定したので計算は合っていそうですね。

これをパーティションテーブルといいストレージ上のセクタ番号を保存して管理しています。
第1～第4も全て同じです。

#### ブートシグネチャ
次の値が最後にありました。

```:ブートシグネチャ
00001f0                                    aa55
```

これはブートシグネチャと呼ばれ、このストレージがMBRで構成されていることを示すマジックナンバーです。



続いてGPTについて中身を見ていきたいと思いますが記事が長くなってしまったので次回にします。