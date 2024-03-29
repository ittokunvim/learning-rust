# RefCell\<T>

> Ref: https://doc.rust-jp.rs/book-ja/ch15-05-interior-mutability.html

内部可変性は、そのデータへの普遍参照があるときでさえもデータを可変できるRustのデザインパターンです。
この行動は借用規則により許可されません。
データを可変化するためにこのパターンは、データ構造内で`unsafe`コードを使用して、可変性と借用を支配するRustの通常の規則を捻じ曲げています。
`unsafe`コードについては第19章で学びます。

たとえコンパイラが保証できなくても、借用規則に実行時に従うことが保証されるとき、内部可変性パターンを使用した型を使用できます。
関係する`unsafe`コードは安全なAPIにラップされ、内側の方は不変です。

内部可変性パターンに従う`RefCell<T>`型を学びつつ、この概念も身につけていきましょう。

## 借用規則を強制する

`Rc<T>`と異なり`RefCell<T>`型は、保持するデータに対して単独の所有権を表します。
ではどうして`RefCell<T>`が`Box<T>`のような型と異なるのでしょうか。
異なる理由は借用規則にあります。

- いかなる時も1つの可変参照か、いくつもの不変参照のどちらかが可能になる
- 参照は常に有効でなければならない

参照と`Box<T>`の借用規則の不変条件は、コンパイル時に強制されます。
`RefCell<T>`の不変条件は、実行時に強制されます。
参照でこれらの規則を破ると、コンパイルエラーになります。
`RefCell<T>`でこれらの規則を破ると、プログラムはパニックを起こします。

コンパイル時に借用規則を精査することの利点は、エラーが開発過程の早い段階で捕捉されることと、あらかじめすべての分析が終わるので、実行パフォーマンスへの影響がないことです。
それらの理由により多くの場合で、コンパイル時に借用規則を精査することが最善の選択肢であり、これがRustの既定になっています。

借用規則を実行時に精査する利点は、コンパイル時の精査では許容されない特定のメモリ安全な筋書きが許容されることです。
Rustコンパイラのような性的解析は本質的に保守的です。
コードの特性にはコードを解析するだけでは検知できないものもあります。
最も有名な例は停止性問題であり、ここでは学びませんが、面白いです。

不可能な分析もあるのでRustのコンパイラが、コードが所有権規則に応じていると確証を得られない場合、正しいプログラムを拒否する可能性があります。実に保守的です。
コンパイラが不正なプログラムを受け入れるとユーザーは、コンパイラが行う保証を信じられなくなるでしょう。

しかしながらコンパイラが正当なプログラムを拒否するのなら、プログラマは不便に思うでしょうが悲劇的なことは何も起こり得ません。
コードが借用規則に従っているとプログラマは確証を得ているが、コンパイラがそれを理解し保証できないときに`RefCell<T>`型が有用になります。

`Rc<T>`と類似して`RefCell<T>`もシングルスレッドで使用するための斧であり、マルチスレッドで使おうとするとコンパイルエラーになります。
`RefCell<T>`の機能をマルチスレッドのプログラムで得る方法については第16章でみます。

これまでのまとめとして、`Box<T>, Rc<T>, RefCell<T>`を選ぶ理由を要約します。

- `Rc<T>`、同じデータに複数の所有者を持たせてくれる
- `Box<T>`、不変借用も可変借用もコンパイル時に精査できる
- `RefCell<T>`、可変借用を実行時に精査できる

不変な値の中の値を可変化することは内部可変性パターンです。
次にこれが有用になるケースを見ていき、使い方も学んでいきます。

## 内部可変性

借用規則の結果は、不変値があるときに可変で借用することはできないということです。

```rust
let x = 5;
let y = &mut x;
```

このコードをコンパイルしようとすると以下のようなエラーになります。

```
error[E0596]: cannot borrow `x` as mutable, as it is not declared as mutable
   --> src/main.rs:130:17
    |
129 |         let x = 5;
    |             - help: consider changing this to be mutable: `mut x`
130 |         let y = &mut x;
    |                 ^^^^^^ cannot borrow as mutable

For more information about this error, try `rustc --explain E0596`.
```

ですがメソッド内で値が自信を可変化するけど、他のコードでは不変に見えることが有用なケースもあります。
その値のメソッドの外のコードは、その値を可変化できないでしょう。
`RefCell<T>`は内部可変性を取得する1つの方法です。
しかし`RefCell<T>`は借用規則を完全に回避するものではありません。
コンパイラの借用チェッカーは、内部可変性を許可し、借用規則は実行時に精査されます。
この規則を侵害すると、コンパイルエラーではなく`panic!`になるでしょう。

`RefCell<T>`を使用して不変値を可変化する実践的な例に取り組み、それが役立つ理由を見ていきましょう。

## 内部可変性：モックオブジェクト

テストダブルは、テスト中に別の型の代わりに使用される型の一般的なプログラミングの概念です。
モックオブジェクトはテスト中に起きることを記録するテストダブルの特定の型なので、正しい動作が起きたことをテストできます。

> テストダブルとは、ソフトウェアテストにおいて、テスト対象が依存しているコンポーネントを置き換える代用品のこと

Rustには他の言語でいうオブジェクトは存在せず、他の言語のように標準ライブラリにモックオブジェクトの機能は組み込まれていません。
ですが同じ目的をモックオブジェクトとして提供する構造体を作成することはできます。

テストを行う筋書きは、値を最大値に対して追跡し、現在地がどれくらい最大値に近いかに基づいてメッセージを送信するライブラリを作成します。
このライブラリはユーザーが行うことのできるAPIコールの数の割り当てを追跡するのに使用されます。

このライブラリは値を見てメッセージを提供するだけです。
これを使用するアプリケーションは、メッセージを送信する機構を提供すると期待されます。
アプリケーションはメッセージを置いたり、メールを送ったり、テキストメッセージを送るなどをします。
ライブラリはその詳細を知る必要はありません。
必要なのは提供する`Messenger`トレイトを実装している何かなのです。

```rust
trait Messenger {
    fn send(&self, msg: &str);
}

struct LimitTracker<'a, T: 'a + Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T> LimitTracker<'a, T>
where
    T: Messenger,
{
    fn new(messenger: &T, max: usize) -> LimitTracker<T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }

    fn set_value(&mut self, value: usize) {
        self.value = value;

        let percentage_of_max = self.value as f64 / self.max as f64;

        if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        } else if percentage_of_max >= 0.9 {
            self.messenger.send("Urgent warning: You've used up over 90% of your quota!");
        } else if percentage_of_max >= 0.75 {
            self.messenger.send("Warning: You've used up over 75% of your quota!");
        }
    }
}
```

このコードの重要な部分の1つは、`Messenger`トレイトに`self`の不変参照とメッセージのテキストをとる`send`というメソッドがあることです。
これはモックオブジェクトが持つ必要のあるインターフェイスです。
もう1つは、`LimitTracker`の`set_value`メソッドの振る舞いをテストしたいということです。
このメソッドは`value`引数に渡すものを変えることはできるが、何も結果を返しません。
`LimitTracker`を`Messenger`トレイトを実装する何かと、`max`の特定の値で生成し、`value`に異なる数値を渡したときに`Messenger`は適切なメッセージを送ることをテストするにはどうすれば良いでしょう。

`send`を呼び出すときにメールやテキストメッセージを送る代わりに送ると指示されたメッセージを追跡するモックオブジェクトが必要です。
モックオブジェクトの新規インスタンスを生成し、モックオブジェクトを使用する`LimitTracker`を生成し、`LimitTracker`の`set_value`を呼び出し、モックオブジェクトに期待しているメッセージを確認します。

```rust
// RefCell<T> type
#[cfg(test)]
mod tests {
    use super::*;

    struct MockMessenger {
        sent_messages: Vec<String>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: vec![],
            }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);

        limit_tracker.set_value(80);

        assert_eq!(mock_messenger.sent_messages.len(), 1);
    }
}
```

このテストコードは`Vec<String>`で送信すると指示されたメッセージを追跡する`sent_messages`フィールドのある`MockMessenger`構造体を定義しています。
また空のメッセージリストから始まる新しい`MockMessenger`値を作るのに便利な`new`も定義しています。
それから`MockMessenger`に`Messenger`トレイトを実装しているので、`LimitTracker`に`MockMessenger`を与えられます。
`send`メソッドの定義で引数として渡されたメッセージを取り、`sent_messages`の`MockMessenger`リストに格納しています。

テストでは`max`値の75%以上になる何かに`value`をセットするように`LimitTracker`が指示されるときに起こることをテストしています。
まず新しい`MockMessenger`を生成し、からのメッセージリストを作成します。
そして新しい`LimitTracker`の参照と100の`max`値を与えます。
`LimitTracker.value`メソッドは80の値で呼び出します。
最後に`MockMessenger`のメッセージのリストに1つのメッセージを含んでいることをアサートします。

しかしこのテストには1つの問題があります。

```
error[E0596]: cannot borrow `self.sent_messages` as mutable, as it is behind a `&` reference
   --> src/main.rs:186:17
    |
37  |     fn send(&self, msg: &str);
    |             ----- help: consider changing that to be a mutable reference: `&mut self`
...
186 |                 self.sent_messages.push(String::from(message));
    |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

For more information about this error, try `rustc --explain E0596`.
```

`send`メソッドは`self`の不変参照をとるので、`MockMessenger`を変更してメッセージを追跡できません。
代わりに`&mut self`を使用するというエラーテキストからの提言も選ぶこともできません。
そうすると`send`のシグニチャが、`Messenger`トレイト定義のシグニチャと一致しなくなるからです。

これは内部可変性が役に立つ場面です！
`sent_messages`を`RefCell<T>`内部に格納すると`send`は`sent_messages`を変更して見かけたメッセージを格納できるようになります。

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: RefCell::new(vec![]),
            }
        }
    }

    impl Messenger for MockMessenger {
        fn send(&self, message: &str) {
            self.sent_messages.borrow_mut().push(String::from(message));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        // --snip--
        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```

これで`sent_messages`フィールドは、`Vec<String>`ではなく型`RefCell<Vec<String>>`になりました。

`send`メソッドの`self`でも不変借用で、トレイと定義と合致しています。
`RefCell<Vec<String>>`の`borrow_mut`を`self.sent_messages`に呼び出し、`RefCell<Vec<String>>`の中の値への可変参照を得てベクターになります。
それからベクターへの可変参照に`push`を呼び出し、テスト中に送られるメッセージを追跡します。

最後にアサートで内部のベクターにある要素の数を数えるために`borrow`を呼び出し、ベクターへの不変参照を得ています。

`RefCell<T>`の使い方は見たので、動作の仕方を深掘りしていきましょう。

## RefCell\<T>の借用

不変および可変参照を作成する際、それぞれ`&, &mut`記法を使用します。
`RefCell<T>`では、`borrow, borrow_mut`メソッドを使用しており、これはRefCellにある安全なAPIの一部です。
`borrow`メソッドはスマートポインタ型の`Ref<T>`を返し、`borrow_mut`メソッドはスマートポインタ型の`RefMut<T>`を返します。
どちらの型も`Deref`を実装しているので、普通の参照のように扱うことができます。

`RefCell<T>`は現在活動中の`Ref<T>, RefMut<T>`スマートポインタの数を追いかけます。
`borrow`を呼び出すたびに、`RefCell<T>`は活動中の不変参照の数を増やします。
`Ref<T>`の値がスコープを抜けたら、不変参照の数は1下がります。
コンパイル時の借用規則と全く同じように、`RefCell<T>`はいかなる時も、複数の普遍借用または1つの可変借用を持たせてくれます。

これらの規則を侵害しようとすれば、参照のようにコンパイルエラーになるのではなく、`RefCell<T>`の実装は実行時にパニックになります。
以下のコードは、`send`実装に変更を加えています。

```rust
impl Messenger for SomeMessenger {
    fn send(&self, msg: &str) {
        let mut one_borrow = self.values.borrow_mut();
        let mut two_borrow = self.values.borrow_mut();

        one_borrow.push(String::from(msg));
        two_borrow.push(String::from(msg));
    }
}
```

`borrow_mut`から返ってきた`RefMut<T>`スマートポインタに対して変数`one_borrow`を生成しています。
`two_borrow`も同じように生成しています。
これにより同じスコープで2つの可変参照ができ、これは許可されません。
このコードはコンパイルはできますが、実行時にエラーを吐きます。

```
thread 'tests::it_sends_an_over_75_percent_warning_message' panicked at 'already borrowed: BorrowMutError', src/main.rs:93:42
```

コードは、`already borrowed: BorrowMutError`のメッセージと共にパニックになります。
このようにして`RefCell<T>`は実行時に借用規則の侵害を扱います。

コンパイル時ではなく実行時に借用エラーをキャッチすることは、開発過程の遅い段階でコードのミスを発見することになります。
これはコードを本番環境にデプロイするまで発見されない可能性があることを意味します。
またコンパイル時ではなく実行時に借用を追いかける結果として、パフォーマンスを多少犠牲とします。

しかしながら`RefCell<T>`を使うことで、不変値のみが許可される文脈で使用しつつ、自身を変更して見かけたメッセージを追跡するモックオブジェクトを書くことが可能になります。
代償はありますが、`RefCell<T>`を使用すれば普通の参照よりも多くの機能を得ることができます。

## Rc\<T>とRefCell\<T>の組み合わせ

`RefCell<T>`の一般的な使用法は、`Ref<T>`と組み合わせることにあります。
`Rc<T>`は何らかのデータに複数の所有者を持たせますが、不変のアクセスしかすることができません。
`RefCell<T>`と`Rc<T>`を組み合わせることで、複数の所有者と可変化できる値を得ることができます。

例として`Rc<T>`で複数のリストに別のリストの所有権を共有したコンスリストの例があります。
`Rc<T>`は不変値だけを抱えるので一旦生成したら、リストの値は変更できません。
これに`RefCell<T>`を組み合わせることで、リストの値を変更することができます。

```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, RefCell<Rc<ListV3>>),
    Nil,
}

use ListV3::{Cons, Nil};

let value = Rc::new(RefCell::new(5));

let a = Rc::new(Cons(Rc::clone(&value), RefCell::new(Rc::new(Nil))));
let b = Cons(Rc::new(RefCell::new(3)), RefCell::new(Rc::clone(&a)));
let c = Cons(Rc::new(RefCell::new(4)), RefCell::new(Rc::clone(&a)));

*value.borrow_mut() += 10;

println!("a after = {:?}", a);
println!("b after = {:?}", b);
println!("c after = {:?}", c);
```

上記のコードをコンパイルすると以下のような出力になります。

```
a after = Cons(RefCell { value: 15 }, RefCell { value: Nil })
b after = Cons(RefCell { value: 3 }, RefCell { value: Cons(RefCell { value: 15 }, RefCell { value: Nil }) })
c after = Cons(RefCell { value: 4 }, RefCell { value: Cons(RefCell { value: 15 }, RefCell { value: Nil }) })
```

このテクニックは非常に綺麗です！
`RefCell<T>`を使用することで表面上は不変な`List`値を持てます。
しかし内部可変性へのアクセスを提供する`refCell<T>`のメソッドを使用できるので、必要な時にはデータを変更できます。
借用規則を実行時に精査することでデータ競合を防ぎ、時としてデータ構造でちょっとのスピードを犠牲にこの柔軟性を得るのは価値があります。

標準ライブラリには、`Cell<T>`などの内部可変性を提供する他の型もあります。
この型は内部値への参照を与える代わりに、値は`Cell<T>`の内部や外部へコピーされる点を除き似ています。
また`Mutex<T>`もあり、これはスレッド間で使用するのが安全な内部可変性を提供します。
第16章ではその使い道を学びます。またこれらの型の違いの詳細は、標準ライブラリのドキュメントで知りましょう。
