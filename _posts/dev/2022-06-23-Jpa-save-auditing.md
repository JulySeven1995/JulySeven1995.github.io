---
title: "언제 @PreUpdate 어노테이션이 실행될까?"
categories:
  - Dev
tags:
  - JPA
  - Trouble Shooting
layout: single
comments: true
---

### When @PreUpdate Works?


---

실무에서 작업을 하던중 어떤 아이템의 최근 수정 날짜가 변경되지 않는다는 이슈가 발생했다.

Where
    프로젝트에서 모듈 Item Delete 시에 프로젝트 수정 일자가 반영되지 않음
Why
    모듈 Item Delete를 실행해도 프로젝트 엔티티 오브젝트의 수정일자가 업데이트 되지 않음.

이슈가 되었던 코드의 예시는 다음과 같다.

```java
@Transactional
public void deleteProjectModule(Long projectId, Long moduleId) {

    Project project = this.projectQueryRepository.findById(projectId).orElseThrow();
    project.deleteModule(moduleId);
    this.projectCommandRepository.save(project);
}
```

Project 도메인의 @OneToMany로 종속된, 모듈 아이템을 제거 하고(orphanremoval = true) 반영 하는 메서드가 실행 되고,
JPA Auditing으로 인해 최근 수정 일자 컬럼이 업데이트가 되지 않는다는것.
하위 도메인을 수정 혹은 삭제를 할 때에 Auditing 이 동작하지 않는지 검증을 해보기로 했다.

---

### Question

의문점 1.

변경 사항이 없으면 save 메서드로 저장하려 해도 Update 동작을 안하는가(flush 하는 시점에 쿼리를 실행시키지 않는가)

의문점 2.

JPA Auditing 기능은 언제 동작하는가? (언제 오브젝트의 필드 값을 변경하는가) 

```java
@Configurable
public class AuditingEntityListener {

	private @Nullable ObjectFactory<AuditingHandler> handler;

	/**
	 * Configures the {@link AuditingHandler} to be used to set the current auditor on the domain types touched.
	 *
	 * @param auditingHandler must not be {@literal null}.
	 */
	public void setAuditingHandler(ObjectFactory<AuditingHandler> auditingHandler) {

		Assert.notNull(auditingHandler, "AuditingHandler must not be null!");
		this.handler = auditingHandler;
	}

	/**
	 * Sets modification and creation date and auditor on the target object in case it implements {@link Auditable} on
	 * persist events.
	 *
	 * @param target
	 */
	@PrePersist
	public void touchForCreate(Object target) {

		Assert.notNull(target, "Entity must not be null!");

		if (handler != null) {

			AuditingHandler object = handler.getObject();
			if (object != null) {
				object.markCreated(target);
			}
		}
	}

	/**
	 * Sets modification and creation date and auditor on the target object in case it implements {@link Auditable} on
	 * update events.
	 *
	 * @param target
	 */
	@PreUpdate
	public void touchForUpdate(Object target) {

		Assert.notNull(target, "Entity must not be null!");

		if (handler != null) {

			AuditingHandler object = handler.getObject();
			if (object != null) {
				object.markModified(target);
			}
		}
	}
}
```

Jpa Auditing 기능을 수행하는 AuditingEventListener 클래스 확인

`@PreUpdate` 어노테이션이 걸린 touchForUpdate 메서드에서 필드를 업데이트 하는 것을 확인 할 수 있음

```java
<T> T markModified(Auditor auditor, T source) {

		Assert.notNull(source, "Source entity must not be null!");

		return touch(auditor, source, false);
}

private <T> T touch(Auditor auditor, T target, boolean isNew) {

		Optional<AuditableBeanWrapper<T>> wrapper = factory.getBeanWrapperFor(target);

		return wrapper.map(it -> {

			touchAuditor(auditor, it, isNew);
			Optional<TemporalAccessor> now = dateTimeForNow ? touchDate(it, isNew) : Optional.empty();

			if (logger.isDebugEnabled()) {

				Object defaultedNow = now.map(Object::toString).orElse("not set");
				Object defaultedAuditor = auditor.isPresent() ? auditor.toString() : "unknown";

				logger.debug(
						LogMessage.format("Touched %s - Last modification at %s by %s", target, defaultedNow, defaultedAuditor));
			}

		return it.getBean();
	}).orElse(target);
}
```

→ JPA에서 Update 할 때에 Auditing이 되는 것을 확인

---

### Validation

1. 환경
    
    JPA 동작을 확인하고 검증하기위해 간단한 테스트 환경 구축
    
    엔티티
        
    ```java
    @Entity
    @Getter
    @Setter
    @Table(name = "record")
    public class Record extends AbstractAuditingEntity implements Serializable {


        @Id
        @GeneratedValue(strategy = GenerationType.AUTO)
        @Column(name = "record_id", nullable = false)
        private Long id;

        @Column(name = "name", nullable = false)
        private String name;

        @OneToMany(mappedBy = "record", fetch = FetchType.LAZY, orphanRemoval = true, cascade = CascadeType.ALL)
        List<RecordItem> items = new ArrayList<>();

        public void addRecordItems(List<RecordItem> recordItems) {

            recordItems.forEach(i -> i.setRecord(this));
            items.addAll(recordItems);
        }

        public void deleteItem(RecordItem recordItem) {
            items.remove(recordItem);
        }
    }
    ```

    ```java
    @Getter
    @Setter
    @Entity
    @Table(name = "record_item")
    public class RecordItem extends AbstractAuditingEntity implements Serializable {

        @Id
        @GeneratedValue(strategy = GenerationType.AUTO)
        @Column(name = "id", nullable = false)
        private Long id;

        @Column(name = "name", nullable = false)
        private String data;

        @ManyToOne(fetch = FetchType.LAZY)
        @JoinColumn(name = "record_id")
        private Record record;

    }
    ```
        
    ```java
    @Getter
    @Setter
    @MappedSuperclass
    @EntityListeners(AuditingEntityListener.class)
    public abstract class AbstractAuditingEntity implements Serializable {
    
        private static final long serialVersionUID = 1L;
    
        @CreatedBy
        @Column(name = "reg_id", length = 50, updatable = true)
        @JsonIgnore
        private String regId;
    
        @CreatedDate
        @Column(name = "reg_date", updatable = true)
        @JsonIgnore
        private Instant regDate = Instant.now();
    
        @LastModifiedBy
        @Column(name = "mod_id", length = 50)
        @JsonIgnore
        private String modId;
    
        @LastModifiedDate
        @Column(name = "mod_date")
        @JsonIgnore
        private Instant modDate = Instant.now();
    
    }
    ```
        
    테스트 코드
        
    * 단순 데이터 변경을 확인하기 위해 assert 작업은 하지 않음
    
    ```java
    @EnableJpaAuditing
    @SpringBootTest
    @ActiveProfiles("test")
    @Transactional
    public class SaveTest {

        @Autowired
        RecordRepository recordRepository;

        @Autowired
        RecordItemRepository recordItemRepository;

        @BeforeEach
        void setup() {
            System.out.println("=====================================");
            System.out.println("Phase 1 데이터 셋업");
            Record record = new Record();
            record.setName("최재호");

            RecordItem recordItem = new RecordItem();
            recordItem.setData("재빠른호돌이 데이터");
            record.addRecordItems(Collections.singletonList(recordItem));

            recordRepository.saveAndFlush(record);
        }

        @Test
        void 객체_속성_수정() {
            System.out.println("=====================================");
            Record record = recordRepository.findAll().stream().findAny().orElseThrow();
            System.out.println("Phase 2 데이터 수정");
            System.out.println("변경 전 레코드 네임 : " + record.getName());
            System.out.println("변경 전 수정 일자 : " + record.getModDate());
            record.setName("재빠른 호돌이");
            recordRepository.save(record);
        }

        @Test
        void 객체_하위_아이템_삭제() {
            System.out.println("=====================================");
            Record record = recordRepository.findAll().stream().findAny().orElseThrow();
            System.out.println("Phase 2 아이템 데이터 삭제");
            System.out.println("아이템 삭제 전 수정 일자 : " + record.getModDate());
            RecordItem recordItem = record.getItems().get(0);
            record.deleteItem(recordItem);
            recordRepository.save(record);
        }


        @AfterEach
        void teardown() {
            Record record = recordRepository.findAll().stream().findAny().orElseThrow();
            System.out.println("=====================================");
            System.out.println("Phase 3 데이터 확인");
            System.out.println("변경 후 레코드 네임 : " + record.getName());
            System.out.println("변경 후 수정 일자 : " + record.getModDate());
            System.out.println("레코드 아이템 개수 : " + recordItemRepository.findAll().size());
            recordRepository.deleteAll();
        }
    }
    ```
        
2. 테스트 결과
    
    case 1. 값이 변경 되었을 때
    
    ```bash
    =====================================
    Phase 1 데이터 셋업
    Hibernate: call next value for hibernate_sequence
    Hibernate: insert into record (mod_date, mod_id, reg_date, reg_id, name, id) values (?, ?, ?, ?, ?, ?)
    =====================================
    Hibernate: select record0_.id as id1_0_, record0_.mod_date as mod_date2_0_, record0_.mod_id as mod_id3_0_, record0_.reg_date as reg_date4_0_, record0_.reg_id as reg_id5_0_, record0_.name as name6_0_ from record record0_
    Phase 2 데이터 수정
    변경 전 레코드 네임 : 최재호
    변경 전 수정 일자 : 2022-03-22T05:47:27.348513Z
    Hibernate: update record set mod_date=?, mod_id=?, reg_date=?, reg_id=?, name=? where id=?
    Hibernate: select record0_.id as id1_0_, record0_.mod_date as mod_date2_0_, record0_.mod_id as mod_id3_0_, record0_.reg_date as reg_date4_0_, record0_.reg_id as reg_id5_0_, record0_.name as name6_0_ from record record0_
    =====================================
    Phase 3 데이터 확인
    변경 후 레코드 네임 : 재빠른호돌이
    변경 후 수정 일자 : 2022-03-22T05:47:27.448360Z
    Hibernate: select record0_.id as id1_0_, record0_.mod_date as mod_date2_0_, record0_.mod_id as mod_id3_0_, record0_.reg_date as reg_date4_0_, record0_.reg_id as reg_id5_0_, record0_.name as name6_0_ from record record0_
    Hibernate: select recorditem0_.id as id1_1_, recorditem0_.mod_date as mod_date2_1_, recorditem0_.mod_id as mod_id3_1_, recorditem0_.reg_date as reg_date4_1_, recorditem0_.reg_id as reg_id5_1_, recorditem0_.name as name6_1_, recorditem0_.record_id as record_i7_1_ from record_item recorditem0_
    레코드 아이템 개수 : 1
    ```
    
    Phase 2에서 save 시, 기존 값을 불러와 비교하여 변경사항이 있음을 감지하고 update하는 것을 확인
    
    mod_date 또한 변경되었음을 확인 할 수 있음
    
    case 2. 값이 변경 되지 않았을 때
    
    setName 메서드를 setup 할 때의 값 그대로 세팅한 후 결과 출력(데이터 변경 없음)
    
    ```bash
    =====================================
    Phase 1 데이터 셋업
    Hibernate: call next value for hibernate_sequence
    Hibernate: insert into record (mod_date, mod_id, reg_date, reg_id, name, id) values (?, ?, ?, ?, ?, ?)
    =====================================
    Hibernate: select record0_.id as id1_0_, record0_.mod_date as mod_date2_0_, record0_.mod_id as mod_id3_0_, record0_.reg_date as reg_date4_0_, record0_.reg_id as reg_id5_0_, record0_.name as name6_0_ from record record0_
    Phase 2 데이터 수정
    변경 전 레코드 네임 : 최재호
    변경 전 수정 일자 : 2022-03-22T05:47:27.348513Z
    Hibernate: update record set mod_date=?, mod_id=?, reg_date=?, reg_id=?, name=? where id=?
    Hibernate: select record0_.id as id1_0_, record0_.mod_date as mod_date2_0_, record0_.mod_id as mod_id3_0_, record0_.reg_date as reg_date4_0_, record0_.reg_id as reg_id5_0_, record0_.name as name6_0_ from record record0_
    =====================================
    Phase 3 데이터 확인
    변경 후 레코드 네임 : 재빠른호돌이
    변경 후 수정 일자 : 2022-03-22T05:47:27.448360Z
    Hibernate: select record0_.id as id1_0_, record0_.mod_date as mod_date2_0_, record0_.mod_id as mod_id3_0_, record0_.reg_date as reg_date4_0_, record0_.reg_id as reg_id5_0_, record0_.name as name6_0_ from record record0_
    Hibernate: select recorditem0_.id as id1_1_, recorditem0_.mod_date as mod_date2_1_, recorditem0_.mod_id as mod_id3_1_, recorditem0_.reg_date as reg_date4_1_, recorditem0_.reg_id as reg_id5_1_, recorditem0_.name as name6_1_, recorditem0_.record_id as record_i7_1_ from record_item recorditem0_
    레코드 아이템 개수 : 1
    ```
    
    update 쿼리가 실행되지 않음

    case 3. 하위 데이터가 삭제 되었을 때

    ```bash
    =====================================
    Hibernate: select record0_.record_id as record_i1_0_, record0_.mod_date as mod_date2_0_, record0_.mod_id as mod_id3_0_, record0_.reg_date as reg_date4_0_, record0_.reg_id as reg_id5_0_, record0_.name as name6_0_ from record record0_
    Phase 2 아이템 데이터 삭제
    아이템 삭제 전 수정 일자 : 2022-06-23T03:21:33.863976Z
    Hibernate: select record0_.record_id as record_i1_0_, record0_.mod_date as mod_date2_0_, record0_.mod_id as mod_id3_0_, record0_.reg_date as reg_date4_0_, record0_.reg_id as reg_id5_0_, record0_.name as name6_0_ from record record0_
    =====================================
    Phase 3 데이터 확인
    변경 후 레코드 네임 : 최재호
    변경 후 수정 일자 : 2022-06-23T03:21:33.863976Z
    Hibernate: delete from record_item where id=?
    Hibernate: select recorditem0_.id as id1_1_, recorditem0_.mod_date as mod_date2_1_, recorditem0_.mod_id as mod_id3_1_, recorditem0_.reg_date as reg_date4_1_, recorditem0_.reg_id as reg_id5_1_, recorditem0_.name as name6_1_, recorditem0_.record_id as record_i7_1_ from record_item recorditem0_
    레코드 아이템 개수 : 0
    ```

    
    변경사항이 없으면 Auditing 동작이 실행되지 않음 (`@PreUpdate` 실행되지 않음)을 알 수 있음
    

---

### Result

결과적으로 이슈를 처리하기 위해선 필드의 값이 ‘변경’ 되어야 함.

속성의 값이 그대로 유지 되는 경우 → 값이 변경되지 않으므로 명시적으로 값을 변경해 줘야 할 필요가 있음