## はじめに
業務でテーブルを空にしたいときに使う **`TRUNCATE`**。
でも「これってロールバックできるの？」と疑問に思ったことはありませんか。

実は、データベースによって挙動が違います。
本記事では、**有名なDBに絞って**挙動を整理します。

## 主要DBごとの挙動

### PostgreSQL
- **ロールバック可能**
- `TRUNCATE` もトランザクション制御下に入るため、`BEGIN … ROLLBACK` で元に戻せます。
- `RESTART IDENTITY` でシーケンス初期化も可能（これもロールバック対象）。

### SQL Server
- **ロールバック可能**
- `BEGIN TRAN; TRUNCATE TABLE ...; ROLLBACK;` で戻せます。
- ただし外部キー参照があると実行不可。

### MySQL / MariaDB
- **ロールバック不可**
- 内部的に `DROP + CREATE` 相当の挙動。
- 暗黙コミットが走るため、ロールバックできません。

### Oracle Database
- **ロールバック不可**
- DDLは基本的に即時コミットされる仕様。
- `DELETE` ならロールバック可能。

### SQLite
- **TRUNCATE構文なし**
- テーブルを空にする場合は `DELETE FROM` を使う。こちらはトランザクションでロールバック可能。

## まとめ
- **ロールバックできる** → PostgreSQL、SQL Server
- **ロールバックできない** → MySQL / MariaDB、Oracle Database、SQLite（TRUNCATEなし）

要は、**「DBによって `TRUNCATE` がロールバックできるかどうかが違う」** という一点を押さえておけば十分です。
実務で安全に扱いたいときは、**使っているDBの仕様を必ず確認してから実行**しましょう。
