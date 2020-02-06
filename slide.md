### Omotesando.rb #55
- - -

### ActiveRecord を拡張しよう

---

#### 自己紹介
- - -

* なまえ  : おしょー
* Twitter : [@pink_bangbi](https://twitter.com/pink_bangbi)
* github  : [osyo-manga](https://github.com/osyo-manga)
* ブログ  : [Secret Garden(Instrumental)](http://secret-garden.hatenablog.com)
* Rails 歴 2年弱
* 趣味で Ruby にパッチを投げたりしてます
  * Ruby 2.7 に [Time#floor](https://bugs.ruby-lang.org/issues/15653) / [Time#ceil](https://bugs.ruby-lang.org/issues/15772) を追加したり
* エディタは Vim
  * <del>わしの vimrc は5000行あるぞ</del>


---

### 今日話すこと
- - -

* ActiveRecord を拡張してパターンマッチで利用できるようにする             <!-- .element: class="fragment" -->
  * 最終的には gem 化する
* それまでの過程を簡単に解説             <!-- .element: class="fragment" -->
  * 詰まった場合にどう対処するかなどなど

---

#### 1. 下準備
- - -

* 最小限の動作環境を構築する
* こうすることで変な依存を減らす事ができる

```ruby
require "active_record"

ActiveRecord::Base.establish_connection(adapter: "sqlite3", database: ":memory:")
ActiveRecord::Schema.define do
  create_table :users, force: true do |t|
    t.string :name
    t.integer :age
    t.timestamps
  end
end

class User < ActiveRecord::Base
end
```

↓↓ Gemfile ↓↓

>>>

```ruby
source 'https://rubygems.org'

gem "rails", "6.0.0"
gem "sqlite3"
```

↓↓最終目標↓↓

>>>

```ruby
def check(user)
  # パターンマッチを使って判定したい
  case user
  in name:, age: (..20)
    "OK : #{name}"
  in name:, age: (21..)
    "NG : #{name}"
  else
    "???"
  end
end

homu = User.create(name: "homu", age: 14)

pp check(homu)
```

---

#### 2. 実装する
- - -

* deconstruct_keys メソッドを追加することでパターンマッチが利用できるようになる

```ruby
class User < ActiveRecord::Base
  # keys は in に渡されたハッシュのキーを受け取る
  # その keys 情報を元にして必要に応じてハッシュを返す
  def deconstruct_keys(keys)
    attributes.transform_keys(&:to_sym).select { keys.include? _1 }
  end
end

def check(user)
  # ...省略...
end

homu = User.create(name: "homu", age: 14)
pp check(homu)  # => "OK : homu"
```

---

#### 3. Refinements で再実装する
- - -

* Refinements を使って機能を制限する

```ruby
class User < ActiveRecord::Base; end
# Refinements で #deconstruct_keys を定義する
module UserWithPatternMatch
  refine User do
    def deconstruct_keys(keys)
      attributes.transform_keys(&:to_sym).select { keys.include? _1 }
    end
  end
end
using UserWithPatternMatch

def check(user)
  # ...省略...
end

homu = User.create(name: "homu", age: 14)
pp check(homu)  # => "???"   ← あれ…？
```

---

## 動かない！！！
## Refinements が怪しそう？🤔

---

#### 4. パターンマッチが Refinements で動作するか確認
- - -

```ruby
class User; end

module UserWithPatternMatch
  refine User do
    def deconstruct_keys(*)
      { name: "homu", age: 14 }
    end
  end
end
using UserWithPatternMatch

def check(user)
  # ...省略...
end

homu = User.new
pp check(homu)     # => "OK : homu"
```

---

### 動く！！！Refinements は無罪！！

---

#### 5. 何が原因か調べる
- - -

* 対処するためにはまず原因を調べる            <!-- .element: class="fragment" -->
* 本当に意図した実装になっているのか確認する            <!-- .element: class="fragment" -->
  * 追加したメソッドが呼ばれている？
  * メソッドが参照できる？
* 仕様を確認する            <!-- .element: class="fragment" -->
* Ruby だとメタプロでいろいろ悪さができるので確認する            <!-- .element: class="fragment" -->
  * method_missing や define_method などなど…
* 動くコードと動かないコードの違いを調べる            <!-- .element: class="fragment" -->
* 1つ1つ要因を潰していく            <!-- .element: class="fragment" -->
* (最悪)CRuby の実装を読む            <!-- .element: class="fragment" -->

↓↓↓↓            <!-- .element: class="fragment" -->

>>>

#### 正しくメソッドが定義されているか確認
- - -

* #method などでメソッドの定義元を確認

```ruby
class User < ActiveRecord::Base; end

module UserWithPatternMatch
  refine User do
    def deconstruct_keys(keys)
      attributes.transform_keys(&:to_sym).select { keys.include? _1 }
    end
  end
end
using UserWithPatternMatch

homu = User.create(name: "homu", age: 14)
# deconstruct_keys の定義元を確認する
# 定義したメソッドが参照されているので大丈夫そう
p homu.method(:deconstruct_keys).source_location
# => ["/home/hoge/code/test.rb", 20]
```

↓↓↓↓

>>>

#### 継承をやめてみる
- - -

```ruby
class User # < ActiveRecord::Base
end

module UserWithPatternMatch
  refine User do
    def deconstruct_keys(keys)
      { name: "homu", age: 14 }
    end
  end
end
using UserWithPatternMatch

def check(user)
  # ...省略...
end

homu = User.new
pp check(homu)     # => "OK : homu"
```

* こんな感じで1つ1つ調べていく

---

#### 6. 原因
- - -

* method(:deconstruct_keys) は正しく参照できている           <!-- .element: class="fragment" -->
* ActiveRecord::Base を継承しない場合は問題なく動作する           <!-- .element: class="fragment" -->
* User.ancestors で何が継承されているのか確認する           <!-- .element: class="fragment" -->
* 1つ1つ include して何が原因か調べる           <!-- .element: class="fragment" -->
* include ActiveModel::AttributeMethods したら再現する事がわかった           <!-- .element: class="fragment" -->
* ActiveModel::AttributeMethods の実装を見てみたら #respond_to? を再定義している事がわかった           <!-- .element: class="fragment" -->

## お　ま　え　か　！　！           <!-- .element: class="fragment" -->

↓↓↓↓            <!-- .element: class="fragment" -->

---

#### 7. 対処する
- - -

* respond_to? を再定義する
* こうすることで deconstruct_keys が参照できる

```ruby
module UserWithPatternMatch
  refine User do
    # 自身を using して deconstruct_keys が参照できるようにする
    using UserWithPatternMatch

    def deconstruct_keys(keys)
      attributes.transform_keys(&:to_sym).select { keys.include? _1 }
    end

    # refine 内で respond_to? を再定義する
    def respond_to?(name, *args)
      name == :deconstruct_keys ? true : super(name, *args)
    end
  end
end
```

---

#### 8. gem 化する
- - -

* 完成したものがこちらになります
* https://github.com/osyo-manga/activerecord-pattern_match

```ruby
require "activerecord/pattern_match"

using ActiveRecord::PatternMatch

user = User.create name: "homu", age: 14

case user
in age: (..10)
  p "A"
in age: (11..20)
  p "B"
in age: (21..)
  p "C"
end
# => "B"
```

---

#### まとめ
- - -

* まずは素の Ruby ファイルから書き始める           <!-- .element: class="fragment" -->
* 問題が発生したら何が原因なのか調べる           <!-- .element: class="fragment" -->
  * 最小構成のコードを構築してみると問題が切り分けやすくなる           <!-- .element: class="fragment" -->
  * 意図したコードが呼ばれているのか調べたりちゃんと参照できるか調べる           <!-- .element: class="fragment" -->
* 最小のコードで動作するようになってから gem 化したりプロダクトのコードに乗っけたりすると楽           <!-- .element: class="fragment" -->
* Refinements つらい           <!-- .element: class="fragment" -->


---

## ご清聴
## ありがとうございました
