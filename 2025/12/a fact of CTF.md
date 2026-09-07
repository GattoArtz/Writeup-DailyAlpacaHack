# a fact of CTF ( 2025/12/2 )
## Description
AlpacaHack で一番最初に完成したものの難易度調整により出題されなかった幻の問題 （運営コメント）
```
▸ 初心者向けヒント
- この問題は Crypto カテゴリー、すなわち暗号（Cryptography）に関する問題です。
- AlpacaHack では現在 Crypto, Pwn, Rev, Web の 4 つのカテゴリーをメインに出題し、その他の問題は Misc カテゴリーに分類しています。
- まずは、添付ファイル a-fact-of-CTF.tar.gz をダウンロードして解凍してみましょう。
- chall.py では環境変数 FLAG を読み込んでいます。これがフラグです。
- そして、そのフラグを素数を用いて変換しています。
- この chall.py の出力が output.txt です。
- output.txt の値からフラグを逆算するのがこの問題のゴールです。
```
## Solve
与えられるコードは以下の通りです。
``` python
import os


flag = os.environ.get("FLAG", "not_a_flag")

# all prime numbers less than 300
primes = [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97, 101, 103, 107, 109, 113, 127, 131, 137, 139, 149, 151, 157, 163, 167, 173, 179, 181, 191, 193, 197, 199, 211, 223, 227, 229, 233, 239, 241, 251, 257, 263, 269, 271, 277, 281, 283, 293]
assert len(flag) <= len(primes)

ct = 1
for i, c in enumerate(flag):
    ct *= primes[i] ** (ord(c))

print(hex(ct))
```
流れとしては、
flagを環境変数から取得
⇒300以下の素数primesを用意
⇒flagの文字列長がprimesのインデックス長よりも短いことを確認
⇒(各flag[i]のUnicodeコードポイント) ** primes[i]をctに掛ける
⇒上の処理を各flagの文字について行う
⇒hex値でprint
といった形です。このことから、出来上がるctはprimesの掛け合わせでできており、primesは素数なので因数分解が容易に可能です。
