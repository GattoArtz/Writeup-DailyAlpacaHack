# emojify ( 2025/12/3 )
## Description
`:pizza:` -> 🍕
## Solve
与えられるindex.jsは以下の通りです。
``` javascript
import express from "express";
import fs from "node:fs";

const waf = (path) => {
  if (typeof path !== "string") throw new Error("Invalid types");
  if (!path.startsWith("/")) throw new Error("Invalid 1");
  if (!path.includes("emoji")) throw new Error("Invalid 2");
  return path;
};

express()
  .get("/", (req, res) => res.type("html").send(fs.readFileSync("index.html")))
  .get("/api", async (req, res) => {
    try {
      const path = waf(req.query.path);
      const url = new URL(path, "http://backend:3000");
      const emoji = await fetch(url).then((r) => r.text());
      res.send(emoji);
    } catch (err) {
      res.send(err.message);
    }
  })
  .listen(3000);
```
htmlのテキストボックスで任意の文字列を入力してShowを押すと、定義されたwafルールに基づいてエスケープし、backendからファイルの中身を受け取る、といった形です。
今回定義されているパスに対するwafルールは以下の通りです。
- stringであること
- /で始まること
- emojiを含むこと
逆に言えば、これ以外のエスケープが用意されていないため、これに合わせてうまくパスを作ってあげれば、wafルールを通ってフラグを取得できそうです。
ここで、flagがどこにあるか確認します。secretディレクトリに含まれるindex.jsは以下の通りです。
``` javascript
import express from "express";

const FLAG = process.env.FLAG ?? "Alpaca{REDACTED}";

express()
  // http://secret:1337/flag
  .get("/flag", (req, res) => res.send(FLAG))
  .listen(1337);
```
これを見ると、FLAGを環境変数から取得し、secret:1337へのGETリクエストがあった際にflagを返す、といった構造をしています。つまり、backendからファイルを取りに行く際にsecret:1337へリクエストを曲げてあげれば、flagの内容を取得して表示してくれそうです。
jsでのURL()コンストラクターでは、文字列で、url が相対参照の場合に使用するベース URL を表します。 指定されなかった場合、既定値は undefined となります。
(引用:https://developer.mozilla.org/ja/docs/Web/API/URL/URL)
つまりは、渡されたパスが相対参照出なかった場合、BaseURLが破棄されそのまま与えられたurlが使用される、ということです。これを利用します。

まず、secret:1337へリクエストを向けるので、//secret:1337とします(//とすることで、相対参照を回避します)。この時点で、wafルールの上二つはクリアしています。
残りはemojiを含めることですが、パスに影響を与えず含める方法の一つとして、クエリパラメーターに入れることを考えました。これなら、パスがemojiの影響を受けません。
## Solver
```
curl http://34.170.146.252:53998/api?path=//secret:1337/flag?q=emoji
```
