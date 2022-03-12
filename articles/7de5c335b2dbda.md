---
title: "Arch Linuxメモ3"
emoji: "📚"
type: "idea"
topics: ["linux", "arch"]
published: false
---

Arch Linuxを触ったときに調べたことをメモしていきます。

# パーティション
あまり慣れてないので調べながら進めます。
現在のArch Linuxの起動環境はUEFIです(とはいいつつUEFIとBIOSの違いもしっかり分かってないんですよね、、、)
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
色々複雑なので少し整理します。


## パーティションの作成
今回はUEFIで構築するため
* EFIパーティション
* rootパーティション
* swapパーティション
が必須になります。今回は`fdisk`ではなく`parted`で作成します。

```sh
parted /dev/sda
```
```sh
GNU Parted 3.4
Using /dev/sda
Welcom to GNU Parted! Type 'help' to view a list of commands.
(parted)
```

まずは`/dev/sda`のパーティションラベル(パーティションテーブル)を作成します。
パーティションラベルとは`/dev/sda`などのストレージ自体に付加する情報です。このあと説明するいくつかのパーティションの先頭位置を記録している領域であり、OSがブートするときの情報を保存しています。

知っておく必要があるのは2種類のパーティションラベルで
__MBR__(Master Boot Record)と
__GPT__(GUID Partition Table)です。
MBRが古い形式で、GPTが現代風と捉えればよいと思います。

