RubyやRailsに触れ始めると、あちこちに「`:` コロン」が登場します。
最初は「なんでこんなにコロンが多いんだ…？」と戸惑う方も多いはずです。（僕だけかな…？）

この記事では、Ruby におけるシンボルの基本と、Railsでの実用例を整理して紹介します。

---

## 1. シンボルとは？

Rubyで `:hoge` と書くと、それは **シンボル** というオブジェクトです。

```ruby
:sushi.class
# => Symbol
```

### 文字列との違い

```ruby
"sushi".object_id   # 毎回異なる
"sushi".object_id   # また異なる

:sushi.object_id    # 常に同じ
:sushi.object_id    # これも同じ
```

- **文字列**: 作るたびに別オブジェクト
- **シンボル**: 常に一意、イミュータブル

そのため、ハッシュのキーなどに好んで使われます。

## 2. ハッシュでの利用

シンボルはハッシュキーに頻出します。

```ruby
menu = { sushi: 1200, ramen: 1000 }
puts menu[:sushi]  # => 1200
```

これは次の書き方と同じ意味です。

```ruby
{ :sushi => 1200, :ramen => 1000 }
```

つまり `sushi:` は `:sushi =>` の省略記法です。

## 3. 値としてもシンボルを使う

キーだけでなく値にシンボルを置くこともできます。

```ruby
order = { main: :gyoza, drink: :beer }

puts order[:main]       # => gyoza
puts order[:drink].to_s # => "beer"
```

Railsでよく見る `hoge: :fuga` のパターンも、
「左のコロンはキー」「右のコロンは値としてのシンボル」と理解すればOKです。

## 4. Railsでの利用例

Railsではシンボルが DSL 的に多用されます。

### バリデーション
```ruby
class User < ApplicationRecord
  validates :email, presence: true
end
```

### enum
```ruby
class User < ApplicationRecord
  enum role: { guest: 0, member: 1, admin: 2 }
end

user = User.new(role: :guest)
user.role  # => "guest"
```

### i18nキー
```ruby
I18n.t(:welcome_message)
```

## 5. 周辺でコロンが出てくるケース

シンボル以外にも、Rubyではコロンを含む記法があります。

### 範囲
```ruby
(1..3).to_a   # => [1, 2, 3]
(1...3).to_a  # => [1, 2]
```

### キーワード引数
```ruby
def greet(name:)
  puts "Hello, #{name}!"
end

greet(name: "Okarun")
# => Hello, Okarun!
```

## 余談

自分は Ruby の `:symbol` を最初に見たとき、「なんだこれ…？」と理解に苦しみました。
でも一度「enumみたいなラベル」と思ったらスッと理解できて、だいぶ楽になりました。

ただ正直、今でも **コロンが左右に出てくる書き方**（例: `hoge: :fuga`）には少し混乱します。
「左がキー、右が値」と整理してますが、慣れるまでは引っかかりポイントでした。

それでも一度理解してしまえば、enumよりもシンボルの方が「名前そのものに意味があるラベル」として直感的で、こちらの方がわかりやすいですね
