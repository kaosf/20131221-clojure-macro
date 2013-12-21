# Clojure macro

* author: ka
  * Twitter: [@ka_](https://twitter.com/ka_)
  * GitHub: [kaosf](https://github.com/kaosf)
* date: 2013-12-21


## Clojure のマクロ入門

* list, eval の説明
* unless という文法を追加する
* 記号の説明
  * ' (quote)
  * ` (backquote)
  * ~ (tilde)


## 参考文献

[プログラミングClojure 第2版  
![](http://ssl.ohmsha.co.jp/imgm/978-4-274-06913-0.gif)
](http://www.amazon.co.jp/dp/4274069133)


## list 関数

与えられた引数が並んだ「リスト」を返す

```clj
(list a b c)
;-> (a b c)

(list f 1 2 3)
;-> (f 1 2 3)

(list '+ 1 2)
;-> (+ 1 2)
```


## eval 関数

与えられたリストを評価する

```clj
(eval '(+ 1 2))
;-> 3

(eval (list '+ 1 2))
;-> 3
```

## 文法を拡張する


## unless が欲しい

if は存在する

```clj
(if true "ok" "ng")
;-> "ok"

(if false "ok" "ng")
;-> "ng"
```

unless は無い

```clj
(unless false "ok" "ng")
;-> "ok"
(unless true "ok" "ng")
;-> "ng"

; こういうのが欲しいな
```


## 関数で作ってみる

```clj
(defn unless-f [cond then else]
  (if cond else then))
```


## 実際使ってみると

```clj
(unless-f false "ok" "ng")
;-> "ok"
```


お？これでいいのでは？


## ダメな例

```clj
(unless-f false
  (println "ok")
  (println "ng"))
;->
; ok
; ng
; nil
```

(println "ng") が実行されてしまった


## なぜか

unless-f は関数である

引数は全て「評価されてから」関数に渡される

なので unless-f は false, nil, nil という引数を受け取っている

※println 関数が評価された結果は nil になる


## 無名関数を使う

```clj
(unless-f false
  (fn [] (println "ok"))
  (fn [] (println "ng")))
;-> #<user$eval678$fn__679 user$eval678$fn__679@2e1ddadc>
```

だめだ関数を渡しただけで実行されてない


## 無名関数を実行する

```clj
(defn unless-f-2 [cond then else]
  (if cond (else) (then)))

(unless-f-2 false
  (fn [] (println "ok"))
  (fn [] (println "ng")))
;->
; ok
; nil
```

これで出来た


## やったと思ったがそんなことは無かったぜ

```clj
(unless-f-2 false "ok" "ng")
;->
; ClassCastException java.lang.String cannot be cast to
; clojure.lang.IFn  user/unless-f-2 (NO_SOURCE_FILE:2)
```

当然関数以外も関数として扱おうとするのでその旨のエラーが出る


## 関数か否かで条件分岐

```clj
(defn unless-f-3 [cond then else]
  (if cond
    (if (function? else) (else) else)
    (if (function? then) (then) then)))
```

さて関数かどうか判定する function? 関数に相当するものを調べるか -> 見つからない


そろそろつらみが溢れてくる (´・_・`)


そもそも unless の度に処理を関数として作り直すとか面倒臭すぎる

if と同じで出来なければ意味無いやん


そこでマクロというものがあるそうですよ


## マクロを使えば出来る

```clj
(defmacro unless [cond then else]
  (list 'if cond then else))
```

```clj
(unless false
  (println "ok")
  (println "ng"))
;->
; ok
; nil
```


## 何が起こっている？

マクロによってコードそのもの (リスト) が生成されて実行されている


## 様子を見る

macroexpand-1 という関数でマクロの展開を追うことが出来る

```clj
(macroexpand-1
  '(unless false
    (println "ok")
    (println "ng")))
;-> (if false (println "ng") (println "ok"))
```

これはマクロを一段階だけ展開する関数

全て展開するには macroexpand を使う

今回はどちらも同じ


## 展開されて出来たもの

```clj
(if false (println "ng") (println "ok"))
```

我々の望みは確かにこの式である

これが評価されれば良い


## マクロの動作

1. マクロが展開される
  * 引数は評価はされない
  * 置換されるだけ
2. 展開出来るマクロがまだあるなら 1. に戻る
3. 展開されて生成されたリストを評価する


多分これと等価

```clj
(eval (macroexpand form))
```


## 記号の説明


## ' (quote)

評価を抑制する

```clj
'(x y z)
;-> (x y z)

(x y z)
;->
; CompilerException java.lang.RuntimeException: 
; Unable to resolve symbol: x in this context, 
; compiling:(NO_SOURCE_PATH:1:1)
```


## 例えば macroexpand-1

```clj
(macroexpand-1 (unless false "ok" "ng"))
;-> "ok"

(macroexpand-1 "ok")
;-> "ok"
```

先に (unless ...) の部分が評価されてしまったことになる

マクロが含まれたリストそのものを渡さなければならない

なので quote を付ける

```clj
(macroexpand-1 '(unless false "ok" "ng"))
;-> (if false "ng" "ok")
```


## 例えば先ほどの unless

```clj
(defmacro unless [cond then else]
  (list if cond else then))
```

こうだとどうなるか？

```clj
;->
; CompilerException java.lang.RuntimeException: 
; Unable to resolve symbol: if in this context, 
; compiling:(NO_SOURCE_PATH:2:3)
```

if ってシンボルが無いと言われる

※ごめんなさいこの辺よく理解し切れてません orz


## ` (backquote)

実はこれ単体では quote とまったく同じ


## ~ (tilde)

` (backquote) は ~ (tilde) と組み合わさったときに意味がある

※なお CommonLisp では , (comma) がこの機能である

※Clojure では comma はただの目印でしかない (あっても無視される)


## quote と backquote の違い

```clj
(def x 1)
(def y 2)

'(x y x y)
;-> (x y x y)

`(x y ~x ~y)
;-> (user/x user/y 1 2)
```

`() の中では，~ が付いたものだけは評価される

user/ と付いているいないはちょっと目を瞑って…


## unless マクロ修正

```clj
(defmacro [cond then else]
  `(if ~cond ~else ~then))
```

これで (list 'if cond else then) と等価になる

※と思う


## まとめ


マクロ難しい


理解出来てる自信がない

コードを生成するコードが通常のコードとほぼ同じレベルで書けてしまう (template だの proc だの Proc だのじゃない)

メタプログラミングたのしい (白目)


## それでも分かる重要なこと

unless が作れた

これは他の言語では無理なのである


## 本からのアドバイス

マクロ書くな

マクロじゃなければ出来ないこと以外でマクロ書くな


## おまけ

マクロの再帰呼び出しも面白いです

Clojure はリーダマクロがプログラマによって作れない (混沌抑制)

リスト操作系の念能力をもっと伸ばさなければならない

cons car cdr だけでは死ぬ

quote とそもそも「シンボル」の正しい理解を得よう

if がどうして 'if にしないとダメだったのかの辺り


## 読もう On Lisp

[On Lisp](http://www.asahi-net.or.jp/~kc7k-nd/onlispjhtml/)


## この本もありがたくお借りしております

[Common Lisp 入門  
![](http://ecx.images-amazon.com/images/I/51gfHrLAtTL._SL500_AA300_.jpg)](http://www.amazon.co.jp/dp/400007685X)

1986 年の本だけど読んでるとためになる辺り凄い


Lisp って楽しいよね！

![](./img/saki-san.jpg)

いっしょに楽しもうよ！


つづく

(Lisp との戦いはまだ始まったばかりだ！)
