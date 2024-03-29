# Cargoを拡張する

> Ref: https://doc.rust-jp.rs/book-ja/ch14-05-extending-cargo.html

Cargoは変更する必要なく新しいサブコマンドで拡張できるように設計されています。
`PATH`にあるバイナリが`cargo-something`という名前なら、`cargo something`を実行することで、Cargoのサブコマンドであるかのように実行することができます。
このような独自のコマンドは、`cargo --list`を実行すると列挙もされます。
`cargo install`を使用して拡張をインストールし、それから組み込みのCargoツール同様に実行できることは、Cargoの設計上の非常に便利な恩恵です！

## まとめ

Cargoでcrates.ioとコードを共有することは、Rustのエコシステムを多くの異なる作業に有用です。
Rustの標準ライブラリは、小さく安定的ですが、クレーとは共有および使用しやすく、言語とは異なるタイムラインで進化します。
積極的にcrates.ioで自分にとって有用なコードを共有してみてください。
他のユーザーにとって、役に立つものかもしれません！
