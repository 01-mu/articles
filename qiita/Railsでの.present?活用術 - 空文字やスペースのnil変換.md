## はじめに
Railsを使っていると、フォーム入力やパラメータ処理で「空文字」「スペースだけの文字列」をどう扱うかが問題になることがあります。
普通にRubyの`nil?`や`empty?`を使うと「スペースだけの文字列」は`true`にならず、判定ロジックが冗長になりがちです。

そこで便利なのが、Railsが提供する`.present?`と`.blank?`です。
この記事では、特に`.present?`を中心に、その特徴と活用例を紹介します。

## `.present?`とは？
Railsでは`Object#present?`メソッドがActiveSupportによって拡張されています。

### 定義の概要

- `blank?`の反対として定義されている
  -  [補足]Railsでは `.blank?` は 、`nil` や空文字、空配列だけでなく 「スペースだけの文字列」も `true` となります
- `nil`、`false`、空文字列（`""`）、空配列、空ハッシュはもちろん、「スペースのみの文字列（半角/全角）」も`false`判定になる

- 逆に`.present?`は、それらの場合には`false`を返す

つまり、`.present?`は「実際に意味のある値が存在する(=present)か」を簡潔に判定できるメソッドです。

## サンプルコード
``` ruby
"".present?
# => false

"   ".present? # 半角スペースのみ
# => false

"　　".present? # 全角スペースのみ
# => false

nil.present?
# => false

"Hello".present?
# => true
```
通常のRubyでは `" ".empty? #=> false` となるため、余計な処理を書かなければなりません。
Railsの`.present?`なら、そのまま「スペースのみの文字列（半角/全角）」を無効値として扱えます。

## 典型的な活用シーン

### 1. フォーム入力のバリデーション前処理
```ruby
if params[:name].present?
  user.name = params[:name]
end
```
このように書くと、ユーザーが空白だけを入力した場合も値を無効と扱えます。

### 2. モデルのコールバックでのデータ整形
```ruby
before_save :normalize_name

def normalize_name
  self.name = nil unless name.present?
end
```
空白入力を`nil`に正規化できるので、DBのカラムが`NULL`か有効値だけになるように整えられます。

### 3. 短絡的な条件分岐
```ruby
puts "値があります" if params[:keyword].present?
```
冗長な`if params[:keyword] && !params[:keyword].empty?`のような記述は不要です。

## 注意点

- `.present?`はRails独自拡張であり、純粋なRubyには存在しません。Rails環境外で使うとエラーになります
- 「数値の`0`」は`present?`で`true`判定されます。空扱いにしたい場合は別途条件を設ける必要があります
  ```ruby
  0.present?
  # => true
  ```

## まとめ

- `.present?`はRailsが提供する便利な判定メソッド
- `nil`、空文字、空配列、「スペースのみの文字列（半角/全角）」 も無効扱いにできる
- フォーム入力やDB保存前の整形に非常に有用