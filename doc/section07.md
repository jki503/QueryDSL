# 스프링 데이터 JPA가 제공하는 Querydsl 기능

</br>

> 제약이 커서 실무에서 사용하기 부족  
> 스프링 데이터에서 제공하는 기능임으로 설명하시는 듯  
> 간단하게 듣고, 문법 정도 레퍼할 정도로 기록

</br>

## 인터페이스 지원 - QueryDslPredicateExecutor

</br>

```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom,
    QuerydslPredicateExecutor<Member> {

  List<Member> findByUsername(String username);
}
```

> 인터페이스 상속 받아서 사용

</br>

```java
  @Test
  public void querydslPredicateExecutorTest() {

    Team teamA = new Team("teamA");
    Team teamB = new Team("teamB");

    em.persist(teamA);
    em.persist(teamB);

    Member member1 = new Member("member1", 10, teamA);
    Member member2 = new Member("member2", 20, teamA);
    Member member3 = new Member("member3", 30, teamB);
    Member member4 = new Member("member4", 40, teamB);

    em.persist(member1);
    em.persist(member2);
    em.persist(member3);
    em.persist(member4);

    QMember member = QMember.member;
    Iterable<Member> result = memberRepository.findAll(member.age.between(10, 40)
        .and(member.username.eq("member1")));

    result.forEach(System.out::println);
  }
```

</br>

- 좋아 보이는 이유
  - 인터페이스 내부 들어가보면 조건 및 정렬 페이징 처리에대해 유연성있는 기능 제공해준다.
  - 실제로 편해보였음
  - 조건이 복잡한 것을 커버해주기 때문에 메서드 반환값을 보장해줄 수 있을 것같았다.

</br>

- 그런데 실무에서 쓰기에 부저적합한이유
  - 조인이 불가능
    - 묵시적 조인은 가능하나 `left join이 불가능`
  - 두 번째는 QueryDsl에 의존적이다.

</br>

## QueryDsl Web 지원

</br>

### QueyrDsl 의존도에 대한 생각

</br>

> `복잡한 쿼리에 대해 풀기위해 사용하는 것이 적합.`  
> 너무 많은 의존도를 가질 경우 비즈니스 로직의 핵심인  
> `service layer에 QueryDsl의 코드 의존도가 높아질 것.`  
> 지금도 Depredicate 된 기능들이 생기기 시작하는데  
> 하위 버전을 커버해줄 것인지..

</br>

</br>

- [공식 url spring-data-jpa/docs](https://docs.spring.io/spring-data/jpa/docs/2.2.3.RELEASE/reference/html/#core.web.type-safe)

```java
@Controller
class UserController {

  @Autowired UserRepository repository;

  @RequestMapping(value = "/", method = RequestMethod.GET)
  String index(Model model, @QuerydslPredicate(root = User.class) Predicate predicate,
          Pageable pageable, @RequestParam MultiValueMap<String, String> parameters) {

    model.addAttribute("users", repository.findAll(predicate, pageable));

    return "index";
  }
}
```

- 아래와 같은 조건으로 파라미터 바인딩을 할 때

```text
?firstname=Dave&lastname=Matthews
```

- 아래와 같은 로직으로 만들어 수행하게 한다는 것

```text
QUser.user.firstname.eq("Dave").and(QUser.user.lastname.eq("Matthews"))
```

</br>

- docs에서 보면 말이 안됨
  - presentation 계층에서 Entity 정보들이 매핑되는 문제
  - join 지원이 안돼 단순한 조건만 가능
    - 이게 api 특화 데이터에 의존하는 dto가 아닌
    - Entity를 뽑아오겠다는 소리

</br>

## 리포지토리 지원 - QueryDslRepositorySupport

</br>

```java

package org.springframework.data.jpa.repository.support;

...

@Repository
public abstract class QuerydslRepositorySupport {

	private final PathBuilder<?> builder;

	private @Nullable EntityManager entityManager;
	private @Nullable Querydsl querydsl;

	/**
	 * Creates a new {@link QuerydslRepositorySupport} instance for the given domain type.
	 *
	 * @param domainClass must not be {@literal null}.
	 */
	public QuerydslRepositorySupport(Class<?> domainClass) {

		Assert.notNull(domainClass, "Domain class must not be null!");
		this.builder = new PathBuilderFactory().create(domainClass);
	}

	/**
	 * Setter to inject {@link EntityManager}.
	 *
	 * @param entityManager must not be {@literal null}.
	 */
	@Autowired
	public void setEntityManager(EntityManager entityManager) {

		Assert.notNull(entityManager, "EntityManager must not be null!");
		this.querydsl = new Querydsl(entityManager, builder);
		this.entityManager = entityManager;
	}

  	/**
	 * Callback to verify configuration. Used by containers.
	 */
	@PostConstruct
	public void validate() {
		Assert.notNull(entityManager, "EntityManager must not be null!");
		Assert.notNull(querydsl, "Querydsl must not be null!");
	}
  // 이하 생략
}
```

</br>

> 메서드를 살펴보면, entityManager를 필드로 가지고 있고,  
> Setter 주입으로 의존성을 주입하는 것을 확인할 수 있다.

</br>

- 스프링 프레임워크는 기본적으로 생성자주입을 권장
  - 필드에서 볼 수 있듯 `EntityManger가 nullable하여 생성자 주입 X`
  - 즉, 애플리케이션 로딩 시점에 세터 주입 후
  - validate() 메서드를 통해 필드들에 대한 검증을 통해서 예외 터트리는 구조

</br>

```java

public class MemberRepositoryImpl extends QuerydslRepositorySupport implements
    MemberRepositoryCustom {

//  private final JPAQueryFactory queryFactory;
//
//  public MemberRepositoryImpl(EntityManager em) {
//    this.queryFactory = new JPAQueryFactory(em);
//  }


  public MemberRepositoryImpl() {
    super(Member.class);
  }
}

```

</br>

> 이런식으로 추상클래스를 상속받은 후 생성자에 클래스 정보를 넘겨준다.
> 그 후 우리가 계속해서 사용한 queryFactory를 통해 select절로 시작하지 않고
> from 절부터 사용 가능(이부분은 생략)

</br>

- 근데 의존성 맺어주고 from() 쓰는건 별로 좋아보이지 않는다.
  - 굳이 의존성 맺는 것까지 추상화?
  - from()보다 select()로 시작하는 것이 명시적
  - 사용자가 api를 쓰기위해 알아야할 사전지식이 너무 많다.

</br>

- 한계점

- Querydsl 3.x 버전을 대상으로 만듬
- Querydsl 4.x에 나온 JPAQueryFactory로 시작할 수 없음
  - select로 시작할 수 없음 (from으로 시작해야함)
- QueryFactory 를 제공하지 않음
- 스프링 데이터 Sort 기능이 정상 동작하지 않음

</br>

- getQuerydsl().applyPagination()

```java

	public <T> JPQLQuery<T> applyPagination(Pageable pageable, JPQLQuery<T> query) {

		Assert.notNull(pageable, "Pageable must not be null!");
		Assert.notNull(query, "JPQLQuery must not be null!");

		if (pageable.isUnpaged()) {
			return query;
		}

		query.offset(pageable.getOffset());
		query.limit(pageable.getPageSize());

		return applySorting(pageable.getSort(), query);
	}

```

> QueryDsl을 불러와 applyPagination 수행  
> 그냥 그렇다..

</br>

## Querydsl 지원 클래스 직접 만들기

- Advanced(생략)
  - 나중에 이부분 필요하면 code refer 하기...

</br>

> 스프링 데이터가 제공하는 QueryDslRepositorySupport가 지닌 한계를 극복하기 위해 직접  
> QueryDsl 지원 클래스를 만들어보자.

</br>

</br>
