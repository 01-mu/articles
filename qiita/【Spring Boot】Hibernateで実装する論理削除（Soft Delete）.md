## はじめに
業務アプリでデータを「削除」する場合、**物理削除（Hard Delete）** でなく **論理削除（Soft Delete）** を求められる場面は多々あります。
監査や復元の必要性があるからです。(もちろん参画しているプロジェクトの要件によって様々ですが...)

Hibernate には `@SQLDelete` と `@Where` を組み合わせる方法があります。
この記事では **サービス層に `softDelete()` を明示する前提** で、実用的な実装方法を紹介します。

## 1. テーブル設計
PostgreSQL を例に `deleted_at` を追加します。

```sql
ALTER TABLE users
  ADD COLUMN deleted_at timestamptz NULL;

-- 削除されていないものだけ UNIQUE を効かせたい場合
-- ※これはPosetgreSQLの部分インデックスですが、DBによって対応は異なってきます
CREATE UNIQUE INDEX ux_users_email_active
  ON users(email) WHERE deleted_at IS NULL;
```

---

## 2. エンティティ定義
`@SQLDelete` で DELETE を UPDATE に置き換え、
`@Where` で削除済みを自動的に除外します。

```java
import jakarta.persistence.*;
import org.hibernate.annotations.SQLDelete;
import org.hibernate.annotations.Where;
import java.time.OffsetDateTime;

@Entity
@Table(name = "users")
@SQLDelete(sql = "UPDATE users SET deleted_at = now() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
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

---

## 3. リポジトリ（復元・物理削除）
復元や物理削除を行いたい場合は、明示的にメソッドを追加します。

```java
public interface UserRepository extends JpaRepository<User, Long> {

  // 論理削除解除（復元）
  @Modifying
  @Query("update User u set u.deletedAt = null where u.id = :id")
  void restoreById(@Param("id") Long id);

  // 本当に削除（物理削除）
  @Modifying
  @Query(value = "delete from users where id = :id", nativeQuery = true)
  void hardDeleteById(@Param("id") Long id);
}
```

---

## 4. サービス層（softDelete を明示）
サービス層で `softDelete()` を明示することで、呼び出し側が意図を理解しやすくなります。

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

  // データの復元 ※必要があれば
  public void restore(Long id) {
    users.restoreById(id);
  }

  // 物理削除 ※必要があれば
  public void hardDelete(Long id) {
    users.hardDeleteById(id);
  }
}
```

業務システムだと、データの復元や削除済データの調査は保守・運用担当者が直接SQLを叩く場面が少なくないので、`restore`と`hardDelete`を実装する場面はあまりないかも。

```java
userService.softDelete(1L);
userService.restore(1L);
userService.hardDelete(1L);
```

---

## 5. SELECT の挙動

`userRepository.findAll()` を実行すると、自動的に `deleted_at IS NULL` が付きます。

```sql
select u.id, u.email, u.name, u.deleted_at
from users u
where u.deleted_at is null;
```
`@Where` により「削除済みを除外する」挙動が保証されるため**アプリ側で毎回条件を書く必要はありません**

---

## 6. 実務での注意点
- **ユニーク制約の衝突**
  削除済みが残るため、`email` の UNIQUE が衝突する可能性あり。
  → PostgreSQLなら部分インデックスで対応。

- **集計・監査**
  削除済みも含めたい場合は、`nativeQuery` で書くのが確実。

- **インデックス設計**
  `deleted_at IS NULL` 条件が必ず入るため、インデックスを適切に貼る。

---

## まとめ
- 論理削除は`@SQLDelete + @Where` で実装できる
- SELECT は自動的に削除済みを除外※クラスに対する`@Where`
- サービス層に `softDelete()` / `restore()` / `hardDelete()` を明示すると実務でわかりやすいと思う
- UNIQUE制約・集計・インデックス設計に注意する

---

## 参考リンク（実在確認済み）
- [How to implement a soft delete with Hibernate](https://thorben-janssen.com/implement-soft-delete-hibernate/)
