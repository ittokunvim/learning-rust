# オブジェクト指向プログラミング

> Ref: https://doc.rust-jp.rs/book-ja/ch17-00-oop.html

オブジェクト指向プログラミング(OOP)は、プログラムをモデル化する手段です。
オブジェクトは1960年代の[Simula](https://ja.wikipedia.org/wiki/Simula)に端緒を発しています。
このオブジェクトはお互いにメッセージを渡し合うというAlan Kayのプログラミングアーキテクチャに影響を及ぼしました。
彼はこのアーキテクチャを解説するために、オブジェクト指向プログラミングという用語を造語しました。

多くの競合する定義がOOPが何かを解説しています。
Rustをオブジェクト指向と区分する定義もありますし、しない定義もあります。
この章では広くオブジェクト指向と捉えられる特定の特徴と、それらの特徴がこなれたRustでどう表現されるかを探究します。
それからオブジェクト指向のデザインパターンをRustで実装する方法を示し、そうすることとRustの強みを活用して代わりの解決策を実装する方法の代償を議論します。
