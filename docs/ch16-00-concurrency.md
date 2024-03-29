# 並行性

> Ref: https://doc.rust-jp.rs/book-ja/ch16-00-concurrency.html

並行性を安全かつ効率的に扱うことは、Rustの目標の1つです。
並行プログラミングはプログラムの異なる部分が独立して実行することであり、並列プログラミングはプログラムの異なる部分が同時に実行することです。
多くのコンピュータが複数のプロセッサの利点を活かすようになるにつれ、重要度を増してきます。
歴史的にこれらの文脈で行うプログラミングは困難で、エラーが起きやすいものでした。
Rustはこれを変えると願っています。

当初Rustチームは、メモリ安全性を保証することと、並行性問題を回避することは、異なる方法で解決すべき別々の課題だと考えていました。
時間と共にチームは、所有権と型システムはメモリ安全性と並行性問題を管理する強力な道具であると発見しました。
所有権と型チェックを活用することで、多くの並行性エラーは実行時エラーではなくコンパイル時エラーになります。

故に実行時に並行性のバグが起きた状況と全く同じ状況を再現しようと時間を浪費させるよりも、不正なコードはコンパイルを拒み、問題を説明するエラーを提示するでしょう。
結果としてプロダクトになった後ではなく、作業中にコードを修正できます。
Rustのこの方向性を「恐るな！並行性」とニックネームをつけました。
これにより潜在的なバグがなく、かつ新しいバグを導入することなく簡単にリファクタリングできるコードを書くことができます。

> 簡潔性のため、並行または並列と述べることで正確を期するのではなく、多くの問題を平行と割り切ってしまいます。
> この本がもし並行性あるいは並列性に関した本ならば、詳述していたでしょう。
> この章に対しては並行を使ったら、脳内で並行または並列と置き換えてください。

多くの言語は自分が提供する並行性問題を扱う解決策について独断的です。
例えばErlangには、メッセージ受け渡しの並行性に関する素晴らしい機能がありますが、スレッド間で状態を共有することに関しては、曖昧な方法しかありません。
可能な解決策の一部のみをサポートすることは、高級言語にとっては合理的な施策です。
なぜなら高級言語は一部の制御を失う代わりに抽象化することから恩恵を受けるからです。

ところが低級言語は、どんな場面でも最高のパフォーマンスで解決策を提供すると想定され、ハードウェアに関してほとんど抽象化はしません。
そのためRustは、自分の状況と必要性に適した方法が何であれ、問題をモデル化するための色々な道具を備えています。

こちらがこの章で講義する話題です。

- スレッドを生成し、複数のコードを同時に走らせる
- チャンネルがスレッド間でメッセージを送るメッセージ受け渡し並行性
- 複数のスレッドが何らかのデータにアクセスする状態共有並行性
- 標準ライブラリが提供する型だけでなく、ユーザが定義した型に対してのRustの並行性の安全保証を拡張する`Sync, Send`トレイト
