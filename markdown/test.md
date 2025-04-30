# Spring Data JPA - 새로운 Entity 판별

Spring Data JPA를 사용하다 보면 `save()` 메서드 호출 시 내부적으로 `persist()`를 호출할지, `merge()`를 호출할지 결정하게 됩니다.  
이 결정은 해당 Entity가 **신규 Entity인지 여부**에 따라 이루어지는데, Spring Data JPA는 이를 어떻게 판단할까요?

---

## 신규 Entity 판단 방식

Spring Data JPA는 내부적으로 `JpaEntityInformation`의 `isNew(T entity)` 메서드를 호출해서 판단합니다.

```java
@Override
public boolean isNew(T entity) {
    if (versionAttribute.isEmpty()
        || versionAttribute.map(Attribute::getJavaType).map(Class::isPrimitive).orElse(false)) {
        return super.isNew(entity);
    }
    BeanWrapper wrapper = new DirectFieldAccessFallbackBeanWrapper(entity);
    return versionAttribute.map(it -> wrapper.getPropertyValue(it.getName()) == null).orElse(true);
}
```

1) `@Version` 필드가 있다면 → null 여부로 판단  
2) `@Version` 필드가 없다면 → `@Id` 필드가 null이거나, primitive 타입인 경우 0인지 확인

> 🏷️ 즉, ID가 null이면 **신규 Entity**로 간주되어 `persist()`가 호출됩니다.

---

## 직접 ID를 지정한 경우의 동작

ID를 직접 지정하면 JPA는 해당 Entity가 이미 존재하는 것으로 판단하여 `merge()`를 호출합니다.  
하지만 Database에는 존재하지 않는 경우 다음과 같은 문제가 발생합니다:

- `SELECT` 쿼리로 존재 여부 확인
- 실제 `INSERT`가 아닌 **UPDATE** 시도
- → 실패하거나 잘못된 데이터 상태 유발

이를 해결하기 위한 방법으로는 `Persistable<T>` 인터페이스를 구현하는 것입니다.

```java
// User.class

@Entity
@Table(name = "users")
@EntityListeners(UserEntityListener.class)
public class User implements Persistable<String> {

    @Id
    private String id;

    private String name;

    private boolean isNew = true;

    protected User() {}

    public User(String id, String name) {
        this.id = id;
        this.name = name;
    }

    @Override
    public String getId() {
        return this.id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return this.name;
    }

    @Override
    public boolean isNew() {
        return this.isNew;
    }

    public void setIsNew(boolean isNew) {
        this.isNew = isNew;
    }
}

// UserEntityListener.class

public class UserEntityListener {
    @PostPersist
    @PostLoad
    public void setNotNew(User user) {
        System.out.println("@PostPersist/@PostLoad called");
        user.setIsNew(false);
    }
}
```

---

## persist() vs merge()

| 구분 | persist() | merge() |
|------|-----------|---------|
| 동작 | 새로운 Entity를 영속성 컨텍스트에 등록 | 준영속 객체를 병합하여 관리 |
| SELECT 쿼리 | ❌ 없음 | ✅ 먼저 조회 후 merge |
| ID 필요 여부 | ❌ 필요 없음 | ✅ 필요 |
| 성능 | 빠름 (직접 INSERT) | 느릴 수 있음 (SELECT + UPDATE) |

> 🏷️ 신규 객체를 `merge()`로 처리하면 불필요한 SELECT 쿼리가 발생하고 성능 저하가 발생할 수 있습니다.

---

## 신규 Entity 판단이 중요한 이유는 무엇일까?

Spring Data JPA의 `SimpleJpaRepository`는 `save()` 메서드에서 다음과 같이 동작합니다.

```java
@Transactional
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        entityManager.persist(entity); // INSERT
    } else {
        return entityManager.merge(entity); // SELECT → UPDATE
    }
}
```

ID를 직접 설정했지만 `isNew()`는 false가 되어 `merge()`를 호출하게 되고,  
Database에는 해당 ID가 존재하지 않지만,  
신규 Entity임에도 불구하고 `SELECT` 후 `UPDATE`를 하게 되어  
→ 실패 또는 데이터 무결성 오류가 발생할 수 있고, 비효율적입니다.

> 🏷️ 정확한 `isNew()` 제어는 성능, 정합성, 쿼리 효율성 측면에서 매우 중요합니다.

---

## 정리

| 상황 | 처리 방식 |
|------|-----------|
| ID 없거나 null → 신규 Entity | `persist()` |
| ID가 존재하지만 실제 DB에는 없음 | `merge()` 호출 → 실패 가능성 / 비효율 |
| ID를 직접 설정한 신규 Entity | `Persistable<T>` + `isNew()` 명시 필요 |

---

## 📗 핵심 요약

- Spring Data JPA는 내부적으로 `isNew()`를 통해 신규 Entity 여부를 판단합니다.  
- ID는 존재하지만 실제 DB에 없는 경우 `SELECT` + `UPDATE`를 수행하며, 이는 정합성 저하 및 성능 저하로 이어질 수 있습니다.  
- ID를 직접 설정했을 경우는 `Persistable<T>`와 `isNew()` 구현으로 명확히 신규 여부를 표시해야 합니다.
