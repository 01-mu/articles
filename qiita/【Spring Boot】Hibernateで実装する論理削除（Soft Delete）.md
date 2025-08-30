## 冒頭補足。バージョン差異について

- Hibernate 6.3（2023年9月リリース）で`@Where` が deprecated（非推奨） になりました
代わりに `@SQLRestriction` / `@SQLJoinTableRestriction` が推奨されています
**使用しているHibernateのバージョンに応じて、本記事の`@Where`を`@SQLRestriction`に読み替えて進めてください**

- Hibernate 6.4（2023年12月リリース）
`@SoftDelete` が導入され、論理削除はさらにシンプルに書けるようになったそうです
しかし、`@SoftDelete`はboolean型前提であり、`deleted_at`のような日付型は対応していません

## はじめに
業務アプリでデータを「削除」する場合、**物理削除（Hard Delete）** でなく **論理削除（Soft Delete）** を求められる場面は多々あります。

これは監査や復元の必要性があるからです。
(もちろん参画しているプロジェクトの要件によって様々ですが...)

Hibernate には `@SQLDelete` と `@SQLRestriction` を組み合わせる方法があります。
この記事では `softDelete()`の実用的な実装方法を紹介します。

**【注意】論理削除（Soft Delete）がアンチパターンか否かの議論は本記事では扱っておりません。顧客要件で実装せざるを得ないケースを想定しています**

## 1. テーブル設計
PostgreSQL を例に `deleted_at` を追加します。

```sql
ALTER TABLE users
  ADD COLUMN deleted_at timestamptz NULL;

-- 削除されていないものだけ UNIQUE を効かせたい場合
-- ※これはPostgreSQLの部分インデックスですが、DBによって対応は異なってきます
CREATE UNIQUE INDEX ux_users_email_active
  ON users(email) WHERE deleted_at IS NULL;
```


## 2. エンティティ定義
`@SQLDelete` で DELETE を UPDATE に置き換え、`@SQLRestriction` で削除済みを自動的に除外します。

```java
import jakarta.persistence.*;
import org.hibernate.annotations.SQLDelete;
import org.hibernate.annotations.Where;
import java.time.OffsetDateTime;

@Entity
@Table(name = "users")
@SQLDelete(sql = "UPDATE users SET deleted_at = now() WHERE id = ?")
@SQLRestriction(clause = "deleted_at IS NULL")
public class User {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(nullable = false, unique = false)
  private String email;

  @Column(nullable = false)
  private String name;

  private OffsetDateTime deletedAt;
}
```

これで `userRepository.deleteById(id)` を呼んでも **物理削除ではなく UPDATE** が実行されます。
さらに `findAll()` や `findById()` は `deleted_at IS NULL` 条件付きで動くため、削除済みデータは返りません。


## 3. リポジトリ（復元・物理削除）
復元や物理削除を行いたい場合は、明示的にメソッドを追加します。

```java
public interface UserRepository extends JpaRepository<User, Long> {

  // 論理削除データの復元
  @Modifying
  @Query("update User u set u.deletedAt = null where u.id = :id")
  void restoreById(@Param("id") Long id);

  // 物理削除
  @Modifying
  @Query(value = "delete from users where id = :id", nativeQuery = true)
  void hardDeleteById(@Param("id") Long id);
}
```



## 4. サービス層（softDelete）
サービス層で `softDelete()` のように、論理削除であることを明示した命名をすることで、呼び出し側が意図を理解しやすくなります。

```java
@Service
@RequiredArgsConstructor
@Transactional
public class UserCommandService {
  private final UserRepository users;

  // 論理削除
  public void softDelete(Long id) {
    users.deleteById(id); // @SQLDelete により UPDATE に置き換わる
  }

  // 論理削除データの復元 ※必要があれば
  public void restore(Long id) {
    users.restoreById(id);
  }

  // 物理削除 ※必要があれば
  public void hardDelete(Long id) {
    users.hardDeleteById(id);
  }
}
```

```java
// 呼び出し例。id=1に対する操作
userService.softDelete(1L);
userService.restore(1L);
userService.hardDelete(1L);
```

（飽くまで私の経験則なのですが...）業務システムだと、データの復元や削除済データの調査は保守・運用担当者が直接SQLを叩く場面が少なくないので、`restore`と`hardDelete`を実装する場面はあまりないかもしれません。


## 5. SELECT の挙動

`userRepository.findAll()` を実行すると、自動的に `deleted_at IS NULL` が付きます。

```sql
select u.id, u.email, u.name, u.deleted_at
from users u
where u.deleted_at is null;
```
`@SQLRestriction` により「削除済みを除外する」挙動が保証されるため
**アプリ側で毎回条件を書く必要はありません。**


## 6. 実務での注意点
### ユニーク制約の衝突
  削除済みが残るため、`email` の UNIQUE が衝突する可能性あり。
  → PostgreSQL なら部分インデックスで対応。

### 集計・監査（削除済みも含めたい場合）
  通常の JPQL/HQL クエリは、エンティティに付与した `@SQLRestriction(clause = "deleted_at IS NULL")` が **常に有効** になります。
  そのため、たとえば次のコードを書いても：

  ```java
  @Query("select count(u) from User u")
  long countAll();
  ```

  実際に発行される SQL は：

  ```sql
  select count(u.id) from users u
  where u.deleted_at is null;
  ```

  → 論理削除済みの行は自動的に除外されます。

  削除済みも含めた集計をしたい場合は、**`nativeQuery` を使うのが確実**です。

  ```java
  @Query(value = "select count(*) from users", nativeQuery = true)
  long countAllUsers();
  ```

  これなら `deleted_at IS NULL` が付かず、削除済みも含めた件数を取得できます。

  ※ もう一つの方法として Hibernate の `@Filter` を使い、セッション単位で「削除済みも含める」切替を行う方法もあります。ただし入門向けには `nativeQuery` の方がシンプルです。

### インデックス設計
  `deleted_at IS NULL` 条件が必ず入るため、これを考慮したインデックスが必要です。
  例：

  ```sql
  CREATE INDEX idx_users_deleted_at ON users(deleted_at);
  CREATE UNIQUE INDEX ux_users_email_active
    ON users(email) WHERE deleted_at IS NULL;
  ```

  → 前者はクエリの性能改善に、後者はユニーク制約の衝突回避に有効です。

## まとめ
- 論理削除は`@SQLDelete + @SQLRestriction` で実装できる
- SELECT は自動的に削除済みを除外（クラスに付けた `@SQLRestriction` が効く）
- サービス層に `softDelete()` / `restore()` / `hardDelete()` を明示すると実務でわかりやすいと思う
- UNIQUE制約・集計・インデックス設計に注意する


## 参考
- [How to implement a soft delete with Hibernate](https://thorben-janssen.com/implement-soft-delete-hibernate/)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
