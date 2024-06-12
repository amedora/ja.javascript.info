
# Promise チェーン

<info:callbacks> の章で言及した問題に戻りましょう。私たちは順次実行される一連の非同期タスクを持っています。例えば、スクリプトの読み込みです。上手くコード化するにはどうすればよいでしょう？

Promise はそのためのいくつかの方法を提供します。

この章では promise チェーンを説明します。

次のようになります:

```js run
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000); // (*)

}).then(function(result) { // (**)

  alert(result); // 1
  return result * 2;

}).then(function(result) { // (***)

  alert(result); // 2
  return result * 2;

}).then(function(result) {

  alert(result); // 4
  return result * 2;

});
```

この考え方は、結果が `.then` ハンドラの連鎖(チェーン)を通じて渡されるということです。

ここでの流れは次の通りです:
1. 最初の promise は1秒で解決されます `(*)`,
2. その後、`.then` ハンドラが呼ばれます `(**)`,
3. 返却された値は次の `.then` ハンドラへ渡されます `(***)`,
4. ...同様に続きます。

結果がハンドラのチェーンに沿って渡されるので、一連の `alert` 呼び出しは `1` -> `2` -> `4` の順番で表示されます。

![](promise-then-chain.svg)

`promise.then` の呼び出しは promise を返すので、続けて次の `.then` を呼び出すことができます。そのためすべてのコードが機能します。

ハンドラが値を返すとき、それは promise の結果になります。なので、次の `.then` はその結果と一緒に呼ばれます。

**よくある初心者が行う誤り: 技術的には単一の Promise に複数の `.then` を追加することもできます。これはチェーンではありません**

例:
```js run
let promise = new Promise(function(resolve, reject) {
  setTimeout(() => resolve(1), 1000);
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});
```

ここで行ったことは、1つの promise に対して複数のハンドラの設定です。これらは結果を相互に渡しません。代わりにそれぞれが独立して処理をします。

ここに図があります(上記のチェーンと比較してください):

![](promise-then-many.svg)

同一の promise 上のすべての `.then` は同じ結果を得ます -- その promise の結果です。従って、上のコードでは、すべての `alert` は同じ `1` を表示します。

実際には、単一の promise に対し複数のハンドラが必要なケースはほとんどありません。チェーンの方がはるかに多く利用されます。

## promise の返却　

`.then(handler)` で使用されるハンドラは promise を作成し返却する可能性があります。

この場合、さらなるハンドラはその promise が完了するまで待ち、結果を得ます。

例:

```js run
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000);

}).then(function(result) {

  alert(result); // 1

*!*
  return new Promise((resolve, reject) => { // (*)
    setTimeout(() => resolve(result * 2), 1000);
  });
*/!*

}).then(function(result) { // (**)

  alert(result); // 2

  return new Promise((resolve, reject) => {
    setTimeout(() => resolve(result * 2), 1000);
  });

}).then(function(result) {

  alert(result); // 4

});
```

ここで最初の `.then` は `1` を表示し、行 `(*)` で `new Promise(…)` を返します。1秒後、それは解決され、結果(`resolve` の引数, ここでは `result*2`) は行 `(**)` にある2番目の `.then` のハンドラに渡されます。それは `2` を表示し、同じことをします。

したがって、出力は前の例と同様 1 -> 2 > 4 ですが、今は `alert` 呼び出しの間に 1秒の遅延があります。

promise を返却することで、非同期アクションのチェーンを組み立てることができます。

## 例: loadScript 

[以前の章](info:promise-basics#loadscript)にあった `loadScript` で対し、この機能を使ってスクリプトを1つずつ順番にロードしてみましょう。:

```js run
loadScript("/article/promise-chaining/one.js")
  .then(function(script) {
    return loadScript("/article/promise-chaining/two.js");
  })
  .then(function(script) {
    return loadScript("/article/promise-chaining/three.js");
  })
  .then(function(script) {
    // それらがロードされていることを表示するために、
    // スクリプトで宣言されている関数を使用
    one();
    two();
    three();
  });
```

アロー関数を使用すると多少短くなります:

```js run
loadScript("/article/promise-chaining/one.js")
  .then(script => loadScript("/article/promise-chaining/two.js"))
  .then(script => loadScript("/article/promise-chaining/three.js"))
  .then(script => {
    // scripts are loaded, we can use functions declared there
    one();
    two();
    three();
  });
```


ここで、各 `loadScript` 呼び出しは promise を返し、次の `.then` はそれが解決されたときに実行されます。その後、次のスクリプトの読み込みを開始します。そのため、スクリプトは次々にロードされます。

このチェーンにより多くの非同期アクションが追加できます。ここで、このコードは依然として "フラット" であり、右にではなく、下に成長していることに注目してください。"破滅のピラミッド" の兆候はありません。

技術的にはそれぞれを `loadScript` に直接 `.then` を書くことも可能です:

```js run
loadScript("/article/promise-chaining/one.js").then(function(script1) {
  loadScript("/article/promise-chaining/two.js").then(function(script2) {
    loadScript("/article/promise-chaining/three.js").then(function(script3) {
      // この関数は変数 script1, script2 と script3 へアクセスすることができます
      one();
      two();
      three();
    });
  });
});
```

このコードは同じことをします: 順番に3つのスクリプトをロードします。しかし、"右に大きくなります"。そのため、コールバックと同じ問題があります。

promise を使い始めた人の中には、チェーンを知らない人もいます。そのような人はこの書き方をしますが、一般的にはチェーンの方が好ましいです。

ネストされた関数が外側のスコープをもつため、`.then` を直接書いても問題ない場合があります。上記の例では、最もネストされたコールバックは`script1`, `script2`, `script3` すべての変数にアクセスできます。ですが、これはルールというよりは例外です。


````smart header="Thenable"
正確には、ハンドラは promise ではなく、"thenable" と呼ばれるオブジェクトを返す可能性があります。"thenable" オブジェクトとは、メソッド `.then` を持つオブジェクトです。これは promise と同じように扱われます。

この思想は、サードパーティライブラリが彼ら自身の "promise 互換な" オブジェクトを実装できるというものです。それらは拡張されたメソッドのセットを持つことができますが、`.then` を実装しているため、ネイティブの promise とも互換があります。

これは thenable オブジェクトの例です:

```js run
class Thenable {
  constructor(num) {
    this.num = num;
  }
  then(resolve, reject) {
    alert(resolve); // function() { native code }
    // 1秒後に this.num*2 で resolve する
    setTimeout(() => resolve(this.num * 2), 1000); // (**)
  }
}

new Promise(resolve => resolve(1))
  .then(result => {
*!*
    return new Thenable(result); // (*)
*/!*
  })
  .then(alert); // 1000ms 後に 2 を表示
```

JavaScript は行 `(*)` で `.then` ハンドラによって返却されたオブジェクトをチェックします: もし `then` という名前のメソッドが呼び出し可能であれば、ネイティブ関数 `resolve`, `reject` を引数として(executor に似ています)それを呼び出し、それらのいずれかが呼び出されるまで待ちます。上の例では、`resolve(2)` が1秒後に `(**)` で呼ばれます。その後、結果はチェーンのさらに下に渡されます。

この特徴により、カスタムオブジェクトを `Promise` から継承することなく、promise チェーンで統合することができます。
````


## より大きな例: fetch 

フロントエンドのプログラミングでは、promise はネットワークリクエストで頻繁に使われます。その例も見てみましょう。

リモートサーバからユーザに関する情報をロードするために [fetch](mdn:api/WindowOrWorkerGlobalScope/fetch) メソッドを使います。メソッドは[別の章](info:fetch)で説明するように多くの任意パラメータがありますが、基本の使い方はとてもシンプルです:

```js
let promise = fetch(url);
```

これは、`url` へネットワークリクエストを行い、promise を返します。promise はリモートサーバがヘッダーで応答するとき、*完全なレスポンスがダウンロードされる前に* `response` オブジェクトで解決されます。

完全なレスポンスを見るためには、`response.text()` メソッドを呼ぶ必要があります: これは完全なテキストがリモートサーバからダウンロードされたときに解決され、そのテキストを結果とする promise を返します。

以下のコードは `user.json` へリクエストを行い、サーバからそのテキストをロードします:

```js run
fetch('/article/promise-chaining/user.json')
  // この .then はリモートサーバが応答したときに実行されます
  .then(function(response) {
    // response.text() は、レスポンスのダウンロードが完了した際に
    // 完全なレスポンステキストで解決される新たな promise を返します
    return response.text();
  })
  .then(function(text) {
    // ...そして、ここではリモートファイルの中身が参照できます
    alert(text); // {"name": "iliakan", isAdmin: true}
  });
```

`fetch` から返却された `response` オブジェクトは、リモートデータを読み込んでJSON としてパースするメソッド `response.json()` も含んでいます。このケースでは、便利なのでそれに置き換えてみます。

わかりやすくするために、アロー関数も使います:

```js run
// 上と同じですが、response.json() はリモートコンテンツを JSON としてパースします
fetch('/article/promise-chaining/user.json')
  .then(response => response.json())
  .then(user => alert(user.name)); // iliakan
```

次に、ロードしたユーザで何かしてみましょう。

例えば、github へもう1つリクエストを行い、ユーザプロフィールを読み込みアバターを表示させてみます。:

```js run
// user.json へのリクエスト
fetch('/article/promise-chaining/user.json')
  // json としてロード
  .then(response => response.json())
  // github へのリクエスト
  .then(user => fetch(`https://api.github.com/users/${user.name}`))
  // json としてロード
  .then(response => response.json())
  // ３秒間アバター画像を表示 (githubUser.avatar_url) 
  .then(githubUser => {
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    img.className = "promise-avatar-example";
    document.body.append(img);

    setTimeout(() => img.remove(), 3000); // (*)
  });
```

このコードは動作します(コードの詳細についてはコメントをみてください)が、完全に自己記述的であるべきです。ここには promise を使い始める人が行う典型的な問題があります。

行 `(*)` を見てください: アバターの表示が終了して削除された *後* に何かをするにはどうすればいいでしょうか？例えば、ユーザ情報を編集するためのフォームを表示したいとします。今のところ、方法はありません。

チェーンを拡張可能にするには、アバターの表示が終了したときに resolve を行う promise を返す必要があります。

次のようになります:

```js run
fetch('/article/promise-chaining/user.json')
  .then(response => response.json())
  .then(user => fetch(`https://api.github.com/users/${user.name}`))
  .then(response => response.json())
*!*
  .then(githubUser => new Promise(function(resolve, reject) {
*/!*
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    img.className = "promise-avatar-example";
    document.body.append(img);

    setTimeout(() => {
      img.remove();
*!*
      resolve(githubUser);
*/!*
    }, 3000);
  }))
  // 3秒後にトリガされます
  .then(githubUser => alert(`Finished showing ${githubUser.name}`));
```

今、`setTimeout` は `img.resolve()` を実行した直後に `resolve(githubUser)` を呼び出します。なので、チェーン内の次の `.then` に制御を渡し、ユーザデータを転送します。

良いプラクティスとして、非同期アクションは常に promise を返すべきです。例え現時点ではチェーンの拡張予定がなくても、将来必要になった場合に何らかのアクションを行わせることができます。

最後に、先程のコードは再利用可能な関数に分割できます:

```js run
function loadJson(url) {
  return fetch(url)
    .then(response => response.json());
}

function loadGithubUser(name) {
  return loadJson(`https://api.github.com/users/${name}`);
}

function showAvatar(githubUser) {
  return new Promise(function(resolve, reject) {
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    img.className = "promise-avatar-example";
    document.body.append(img);

    setTimeout(() => {
      img.remove();
      resolve(githubUser);
    }, 3000);
  });
}

// Use them:
loadJson('/article/promise-chaining/user.json')
  .then(user => loadGithubUser(user.name))
  .then(showAvatar)
  .then(githubUser => alert(`Finished showing ${githubUser.name}`));
  // ...
```

## サマリ

`.then` (あるいは `catch/finally`)ハンドラが Promise を返した場合、チェーンの残りの部分はそれが確定するまで待ちます。その後、その結果(あるいはエラー)はさらに渡されていきます。

これは完全な図です:

![](promise-handler-variants.svg)
