# ハッシュマップでキーと値を関連付けて保存

> Ref: https://doc.rust-lang.org/book/ch08-03-hash-maps.html

`HashMap<K, V>`型は、ハッシュ関数を用いてK型のキーとV型の値のマッピングを格納し、これらのキーと値をどのようにメモリに配置するか決定するものです。
多くのプログラミング言語がこの種のデータ構造をサポートしており、ハッシュ、マップ、オブジェクト、ハッシュテーブル、辞書、連想配列など、さまざまな名称で呼ばれることがあります。

ハッシュマップは、ベクターのようなインデックスではなく、任意の型を持つキーを使用してデータを調べたい場合に有効です。
例えば、ゲームで各チームの得点をハッシュマップで管理したいとすれば、チーム名を指定することでそのチームのスコアを取得したりできます。

この節では、ハッシュマップの基本的なAPIについての説明と、標準ライブラリの`Hash<K, V>`に定義されている多くの優れた機能を持った関数についてみていきます。

## ハッシュマップの作成

空のハッシュマップを作る方法の1つは`new`を使って、`insert`で要素を追加することです。

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
```

まず、標準ライブラリのコレクションから`HashMap`を`use`する必要があります。3つの共通コレクションのうち、このコレクションは最も使用頻度が低いので、プレリュードで自動的にスコープに入る機能には含まれません。
ハッシュマップも標準ライブラリのサポートは少なく、ハッシュマップを作成する組み込むマクロもありません。

ベクターと同じようにハッシュマップも、ヒープ上にデータを保存します。このHashMapはString型のキーとi32型の値を持ちます。
ベクターと同様にハッシュマップも同質で、すべてのキーは互いに同じ型を持ち、すべての値は同じ型でなければなりません。

## ハッシュマップの値のアクセス

ハッシュマップの値のアクセスには、`get`メソッドを使用します。

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

let team_name = String::from("Blue");
let score = scores.get(&team_name).copied().unwrap_or(0);
```

ここでは、`score`は青チームに関連する値を持ち、結果は10になります。
`get`メソッドは`Option<&V>`を返します。もしハッシュマップにそのキーの値がなければ、`get`は`None`を返します。
`copied`メソッドは`Option<&i32>`ではなく、`Option<i32>`を得るために呼び出し、`scores`がそのキーのエントリを持っていなければ、`score`をゼロに設定する`unwrap_or`メソッドを呼ぶことによって、`Option`を処理します。

ハッシュマップの各キーの値のペアは、ベクターと同じように`for`ループを使ってループ処理することが出来ます。

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

for (key, value) in &scores {
    println!("{key}: {value}");
}
```

## ハッシュマップの所有権

`i32`のような`Copy`トレイトを実装する型に対して、値はハッシュマップにコピーされます。

```rust
use std::collections::HashMap;

let field_name = String::from("Favorite color");
let field_value = String::from("Blue");

let mut map = HashMap::new();
map.insert(field_name, field_value);

```

`field_name, field_value`は、ハッシュマップに移動された後、使用することが出来なくなります。

ハッシュマップに値への参照を挿入しても、その値はハッシュマップに移動しません。参照が指す値はハッシュマップが有効である限り有効でなければなりません。
この問題については、第10章のライフタイムのところで説明します。

## ハッシュマップの更新

キーと値のペアは増やすことはできるが、一意のキーは一度に1つの値しか持つことが出来ません。

ハッシュマップのデータを変更したい場合、あるキーにすでに値が割り当てられている場合をどう処理するかを決めねばなりません。
古い値を新しい値に置き換えて、古い値を完全に無視することもできますが、古い値を残して新しい値を無視し、キーにまだ値のない場合のみ新しい値を追加することもできます。
あるいは古い値と新しい値を組み合わせることもできます。

**値の上書き**

キーと値をハッシュマップに挿入し、次に同じキーに異なる値を挿入するとそのキーに関連する値は置き換えられます。

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Blue"), 25);

println!("{:?}", scores);  // {"Blue": 25}
```

**キーがない場合のみ、キーと値を追加**

ハッシュマップにキーと値がすでに存在するか確認し、あれば値をそのままにし、なければそのキーと値を挿入する、といった処理は一般的です。

ハッシュマップには`entry`という特別なAPIがあり、チェックしたいキーをパラメータとして受け取ります。
`entry`メソッドの戻り値は、Entryと呼ばれる列挙型で、これは存在するかもしれない値、存在しないかもしれない値を表します。

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);

scores.entry(String::from("Yellow")).or_insert(50);
scores.entry(String::from("Blue")).or_insert(50);

println!("{:?}", scores);
```

Entryの`or_insert`メソッドは、対応するEntryキーが存在すればその値へのmutableな参照を返し、存在しなければ新しいキーと値のmutableな参照を返すように定義されています。
このテクニックはロジックを自分で描くよりもずっと綺麗で、さらに借用規則ともうまく連携します。

**古い値を元にした値の更新**

ハッシュマップのもう1つの一般的な使用例は、キーの値を調べてから古い値に基づいて更新することです。
例えば、あるテキストにそれぞれの単語が何回現れるか数えるコードを書き、その単語をキーとするハッシュマップを使用し、その単語が何回出たか記録するために値をインクリメントする実装などができます。

```rust
let text = "Hello world wonderful world";

let mut map = HashMap::new();

for word in text.split_whitespace() {
    let count = map.entry(word).or_insert(0);
    *count += 1;
}

println!("map: {:?}", map);  // map: {"Hello": 1, "world": 2, "wonderful": 1}
```

`split_whitespace`メソッドは、テキスト中の値を空白で区切ったサブスライスに対するイテレータを返します。
`or_insert`メソッドは指定されたキーの値への変更可能な参照(`&mut V`)を返します。
ここではその変更可能な参照を変数`count`に格納しているので、その値に代入するにはアスタリスク(*)を使って`count`を非参照にする必要ばあります。
ミュータブル参照は、`for`ループの最後でスコープ外に出ますので、これらの変更はすべて安全で、借用規則によって許可されます。

## ハッシュ関数

デフォルトでは、HashMapはSipHashと呼ばれるハッシュ関数を使用し、ハッシュテーブルを含むサービス拒否(DoS)攻撃への耐性を提供することが出来ます。
これは最速のハッシュアルゴリズムではありませんが、性能の低下に伴うより良いセキュリティのためのトレードオフには価値があります。
コードをプロファイリングして、デフォルトのハッシュ関数が目的に対して遅すぎることがわかったら、別のハッシャーを指定して別の関数に切り替えることが出来ます。
ハッシャーはBuildHasherトレイトを実装した型です。トレイトとその実装方法については第10章で説明します。
crates.ioには、他のRustユーザが共有するライブラリがあり、多くの一般的なハッシュアルゴリズムを実装したハッシャーが提供されています。

### まとめ

ベクター、文字列、ハッシュマップはプログラムでのデータの保存、アクセス、変更が必要なときに多くの機能を提供します。これらの機能をマスターできているか以下の課題に挑戦してみてください。

- 整数のリストが与えられたときにベクターを使って、リストの中央値(ソースされたときに中の位置にある値)と最頻値（最も頻繁に出現する値）を返す関数の作成
- 文字列をビッグラタンに変換する関数の作成。各単語の最初の子音を語尾に移動し、"ay"を追加。"first"は"irst-fay"となる。母音で始まる単語は、代わりに"hay"が末尾に追加される("apple"は"apple-hay")。UTF-8エンコーディングの詳細について覚えておくこと
- ハッシュマップとベクターを使って、ユーザーが従業員名を会社の部署に追加できるようなテキストインターフェイスを作成。例えば、"サリーを技術部に"や"アミールを営業部に"といった感じです。そしてユーザーに部門に属するすべての人、または部門別に会社に属するすべての人をアルファベット順に並べたリストを取得する

標準ライブラリのAPIドキュメントには、ベクター、文字列、ハッシュマップが持つメソッドが記述されており、これらの演習に役立ちます。