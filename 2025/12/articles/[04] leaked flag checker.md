# leaked flag checker ( 2025/12/4 )
## Description
みんなだいすきフラグチェッカー
## Solve
与えられるコードは以下の通りです。
``` c
// gcc -o challenge challenge.c
#include <stdio.h>
#include <string.h>

int main(void) {
    char input[32];
    const char xor_flag[] = "REDACTED";
    size_t flag_len = strlen(xor_flag);

    printf("Enter flag: ");
    fflush(stdout);
    scanf("%31s", input);

    if(strlen(input) != flag_len) {
        printf("Wrong length\n");
        return 1;
    }
    for(size_t i = 0; i < flag_len; i++) {
        if((input[i] ^ 7) != xor_flag[i]) {
            printf("Wrong at index %zu\n", i);
            return 1;
        }
    }
    printf("Correct\n");
    return 0;
}
```
flagの中身はわかりませんが、flagの各文字について7でXORし、その文字列がinputと等しければcorrectを返します。
一見して、flagがわからないため解くことが不可能なように見えますが、本当にflagの中身はわからないのでしょうか？
ここで、ghidraを用いてELFの中身を見てみます。main関数の逆コンパイルされたコードは以下の通りです。
``` c

undefined8 main(void)

{
  size_t sVar1;
  undefined8 uVar2;
  long in_FS_OFFSET;
  ulong local_58;
  byte local_46 [54];
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  local_46[0] = 0x46;
  local_46[1] = 0x6b;
  local_46[2] = 0x77;
  local_46[3] = 0x66;
  local_46[4] = 100;
  local_46[5] = 0x66;
  local_46[6] = 0x7c;
  local_46[7] = 0x6b;
  local_46[8] = 0x72;
  local_46[9] = 100;
  local_46[10] = 0x6c;
  local_46[0xb] = 0x7e;
  local_46[0xc] = 0x7a;
  local_46[0xd] = 0;
  printf("Enter flag: ");
  fflush(stdout);
  __isoc99_scanf(&DAT_00102011,local_46 + 0xe);
  sVar1 = strlen((char *)(local_46 + 0xe));
  if (sVar1 == 0xd) {
    for (local_58 = 0; local_58 < 0xd; local_58 = local_58 + 1) {
      if ((local_46[local_58 + 0xe] ^ 7) != local_46[local_58]) {
        printf("Wrong at index %zu\n",local_58);
        uVar2 = 1;
        goto LAB_00101302;
      }
    }
    puts("Correct");
    uVar2 = 0;
  }
  else {
    puts("Wrong length");
    uVar2 = 1;
  }
LAB_00101302:
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return uVar2;
}
```
ばっちりflagが見えてしまっていますね。そもそもここになければ比較のしようがないので、当たり前といえば当たり前ですが。
よって、この各Unicodeコードポイントを7でXORしてフラグを復元します。※XORは可逆なため。
## Solver
``` python
ords = [0x46,0x6b,0x77,0x66,100,0x66,0x7c,0x6b,0x72,100,0x6c,0x7e,0x7a,0]

for o in ords:
    print(chr(o ^ 7))
```