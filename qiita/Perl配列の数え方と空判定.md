配列の要素数の取得と空チェックについてメモ

## 要素数を数える
```perl
my @fruits = ('apple', 'banana', 'orange');

my $count1 = scalar @fruits; # => 3
my $count2 = @fruits;        # => 3
```
- `scalar @配列` → 要素数を返す
- スカラ変数に直接代入しても同じ動き
- 個人的には `scalar @配列` 派（明示的で誤解がない）

## 空かどうか判定する
```perl
if (!@fruits) {
    print "配列は空\n";
}

if (@fruits == 0) {
    print "配列は空\n";
}
```
- `!@配列` は簡潔で結構好み
- `@配列 == 0` も読みやすいと思う

## 使用例
### 早期リターン
```perl
sub process_fruits {
    my @fruits = @_;
    return unless @fruits; # 空なら終了

    foreach my $fruit (@fruits) {
        print "$fruit を処理中...\n";
    }
}
```

### 入力チェック
```perl
my @fruits = grep { /\S/ } <STDIN>; # 空行は除外
warn "入力がありません" unless @fruits;
```

## 注意ポイント
- `undef` 配列と空配列は別物。
- `defined @配列` は使えない（配列のdefinedは無効）
- `defined` はスカラ値が undef かどうかを判定する関数で、配列全体には使えない
```perl
my @fruits  = ();     # 要素数 0
my @fruits2 = undef;  # 要素数 1 (中身は undef)

print scalar @fruits;  # 0
print scalar @fruits2; # 1
```
