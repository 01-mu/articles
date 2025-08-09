## はじめに
実務でPerlを扱う機会に恵まれたので備忘録。
Perlでデータベースを扱う場合、DBIモジュールが標準のようです。
この記事では以下を扱います。
- 接続（`dbh`作成）
- CRUD（Create/Read/Update/Delete）
- トランザクション

## 1. DBIとは？ハンドルとは？
- DBI = Database Interface。PerlからMySQLやPostgreSQLなどのRDBMSに接続するための共通インターフェース

- ハンドル（handle） = データベースとのやり取りやSQL文の実行状態を管理する「取っ手（オブジェクト）」のこと。Perlでは、DBIを扱う変数を慣習的に以下のように命名するようです
    + `dbh`（database handle） → DB接続そのものを表すハンドル
    + `sth`（statement handle） → SQL文を準備・実行するためのハンドル

## 2. DBIの準備
CPANまたはシステムパッケージからDBIとドライバをインストールします。
（MySQLの場合）
```sh
cpanm DBI DBD::mysql
# または
apt install libdbi-perl libdbd-mysql-perl
```

## 3. 接続（`dbh`作成）
Perlでは真偽値を `1`（true） / `0`（false）で表現します。
```perl
use strict;
use warnings;
use DBI;

# DSN（Data Source Name）は接続先DBやホスト情報をまとめた文字列
my $dsn  = "DBI:mysql:database=db_name;host=localhost";
my $user = "db_user";
my $pass = "password";

my $dbh = DBI->connect(
  $dsn, $user, $pass,
  {
    # 以下の属性オプションを1(有効)にする
    # ※属性オプション = DBIの動作を制御するための設定ハッシュ
    RaiseError => 1,           # エラー時に例外
    AutoCommit => 1,           # 自動コミット ※詳細はトランザクションの項で後述
    mysql_enable_utf8mb4 => 1, # MySQL用オプション
  }
);
```
## 4. CRUDの例

### CREATE
```perl
# do → 単発でSQLを実行する
$dbh->do("INSERT INTO users (name, email) VALUES (?, ?)",
  undef, # 未定義値（undefined value）。属性オプションを指定しない
  'Mu', 'Mu@example.com');
```

### READ
```perl
my $sth = $dbh->prepare("SELECT id, name FROM users WHERE id = ?");
$sth->execute(1); # プレースホルダに1を渡している

# fetchrow_hashref →SELECTの結果を1行ずつハッシュリファレンスで取得するメソッド
while (my $row = $sth->fetchrow_hashref) {
  print "$row->{id}: $row->{name}\n";
}
$sth->finish; # 結果セットを閉じる（MySQLでは次のクエリ実行前に必要な場合あり）
```

### UPDATE
```perl
$dbh->do("UPDATE users SET email = ? WHERE id = ?",
          undef,
         'new-mu@example.com', 1);
```
### DELETE
```perl
$dbh->do("DELETE FROM users WHERE id = ?", undef, 1);
```

## 5. トランザクション
複数の処理をまとめて実行し、途中で失敗したらすべて取り消します(=ロールバック)。
```perl
$dbh->{AutoCommit} = 0; # トランザクション開始。明示的にコミットするまで反映されない

eval {
  $dbh->do("INSERT INTO accounts (user_id, balance) VALUES (?, ?)", undef, 1, 1000);
  $dbh->do("UPDATE accounts SET balance = balance - 500 WHERE user_id = ?", undef, 1);
  $dbh->commit; # 正常終了
};
if ($@) { # エラーがあれば$@に格納される
  warn "Transaction failed: $@";
  $dbh->rollback;
}

$dbh->{AutoCommit} = 1; # 元に戻す
```

## 6. まとめ
- `dbh` = データベース接続（ハンドル）
- `sth` = SQLステートメント（ハンドル）
- `do`で単発実行、`prepare`→`execute`で繰り返し実行
- トランザクションは`AutoCommit => 0`にして`commit`/`rollback`

## おわりに
DBIのコードを初めて読んだとき、`$dbh` や `$sth` という変数名が突然出てきて
「いったい`h`はどこからでてきた？？？」と混乱しました。

調べてみたら、`dbh` = database handle、`sth` = statement handle というPerl界隈では当たり前の命名ルールだそうです。
…そんなの知らんがな！！！と思ったものですが、気づけば自分も自然に `$dbh` と書いている今日この頃。
プログラミング界の“方言”って、気づいたら自分もしゃべってるあたりが面白いですね