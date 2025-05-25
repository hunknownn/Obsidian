만약, spring boot 프레임워크와 Hibernate ORM을 사용하고 있다면 PhysicalNamingStrategy, ImplicitNamingStrategy 을 통해 많은 편리함을 얻을 수 있다.

오래전에 변경된 부분이긴하지만 spring boot 2.x.x와 3.x.x에서 변경된 부분에 대해 조금 확인해보자.

### ImplicitNamingStrategy
우선 ImplicitNamingStrategy은 논리이름을 위한 네이밍 전략이라고 보면 된다. 즉 우리가 Entity을 설계할 때 @Column(name = "column_name") 과 같이 명시적으로 이름을 지어주지 않았을 때 사용되는 전략이다.

구현체들을 살펴보면

![[스크린샷 2025-05-25 오후 12.04.16.png]]
와 같은 여러개의 구현체들이 있는데, spring boot은 기본전략으로 `SpringImplicitNamingStrategy` 을 사용하고 있다. (org.springframework.boot.autoconfigure.orm.jpa.HibernateProperties에서 정적 중첩 클래스 Naming의 applyNamingStrategies 함수에 기본전략을 확인 할  수 있다. 이 함수는 아래에서 다시 한번 확인해본다)

### PhysicalNamingStrategy
PhysicalNamingStrategy은 논리이름을 물리(DB)이름으로의 변환을 위한 네이밍 전략이다. 

![[스크린샷 2025-05-25 오후 2.28.38.png]]
위 구현체들은 spring boot 2.6.0 기준이다. (마지막 구현체인 SpringPhysicalNamingStrategy는 2.8.0에서 완전히 제거되고,  CamelCaseToUnderscoresNamingStrategy로 대체된다)
SpringPhysicalNamingStrategy 의 패키지가 `org.springframework.boot.orm.jpa.hibernate` 인것을 보면, Hibernate가 아닌 Spring boot에서 제공해주는 구현체임을 볼 수 있다.

> [PhysicalNmaingStrategy의 함수들의 파라미터명이 name -> logicalName 으로 변경된 commit을 확인할 수 있다]( https://github.com/hibernate/hibernate-orm/commit/5761e7801b9b52201feb65c68bd343a7c5781d75#diff-28b0596e02db3cb3f4a0aee0075783afb74e63141469565f184e62973fe460ec)
> 
> logicalName 으로 파라미터명이 변경되면서, 기능이 좀 더 분명해진것같다


### spring boot 2.6.0
위에서 spring boot의 PhysicalNamingStrategy 기본전략이 SpringPhysicalNamingStrategy에서 CamelCaseToUnderscoresNamingStrategy 으로 변경되었다고 했는데, 이 [commit](https://github.com/spring-projects/spring-boot/commit/4d30eb453f3d033784ca307257ae9436047bb54e)을 확인해보면 변경점을 볼 수 있다. 두 클래스 코드를 보면 알 수 있지만, 코드가 다르지않다. 

그럼 왜 SpringPhysicalNamingStrategy을 완전히 제거하고 CamelCaseToUnderscoresNamingStrategy으로 기본전략을 변경했는지에 대한 이유는 다음 [gh-27352](https://github.com/spring-projects/spring-boot/issues/27352)에서 확인할 수 있다. 

> 간단히 정리하면, Hibernate측에서 사람들이 spring boot의CamelCaseToUnderscoresNamingStrategy 을 복사해서 많이 쓰니깐, 아예 지원해주기로 했다. 이후에 spring boot에서는 이 변경점이 release 되었으니, 아예 기존 SpringPhysicalNamingStrategy을 제거한것으로 보인다.


###  Log와 Debug로 네이밍 전략이 사용되는 흐름 추적하기

```kotlin
@Entity  
class Item(  
    user: User,  
    content: String,  
    hash: String,  
    status: ItemStatus,  
    dueDate: LocalDateTime?,  
) {  
    @Id  
    @GeneratedValue(strategy = GenerationType.IDENTITY)  
    val id: Long = 0  
  
    @ManyToOne(fetch = FetchType.LAZY)  
    @JoinColumn(name = "user_id")  
    val user = user  
  
    var content = content  
  
    @Column(name = "this_is_hash")  
    val hash = hash  
  
    @Enumerated(EnumType.STRING)  
    var status = status  
        protected set  
  
    var dueDate = dueDate  
        protected set
}
```
간단하게 위와 같은 Entity을 만들고, 모두 기본 네이밍전략을 사용하게 하자.
hash 필드는 @Column 을 사용하여 직접 'this_is_hash' 라는 컬럼명을 사용하도록 설정해주었다.

디버깅모드로 확인해볼텐데, spring에서 너무 많은 일을 대신 해주고 있어 모든 과정을 추적하지는 못했겠지만 네이밍 전략이 사용되는 지점과 변환된 이름들이 어떻게 사용되는지 까지는 확인해보겠다.

우선 디버깅한 환경은 Hibernate,spring boot 3.4.2 버전을 사용하고, 테스트에 사용된 mysql 8.0은 docker로 띄워져있다.

#### ㄱ. NamingStrategy의 기본전략 설정까지의 흐름

`org.springframework.boot.autoconfigure.orm.jpa.JpaBaseConfiguration 의 entityManagerFactoryBuilder()` 부터 확인해보자.  모든 흐름을 보여줄 수는 없지만, 큰 흐름만 확인해보자.

![[스크린샷 2025-05-25 오후 11.29.21.png]]

![[스크린샷 2025-05-25 오후 11.34.45.png]]
함수명 처럼 우리가 사용할 Hibernate의 properties 들을 초기화한다.  이 함수 내부로 들어가다보면 `getAdditionalProperties()` 함수에서 `applyNamingStrategies()`함수를 호출하는데, 이 함수가 바로 위에서 말한 기본 전략을 지정해주는 함수가 된다.

![[스크린샷 2025-05-25 오후 11.38.35.png]]

아무런 설정을 하지 않는다면 위 함수에 의해 PHYSICAL_NAMING_STRATEGY가 `CamelCaseToUnderscoresNamingStrategy`로 설정된다. 참고로 jpaProperties가 채워지기 시작하는데

![[스크린샷 2025-05-25 오후 11.37.18.png]]
이 properties에는 '네이밍 전략' 말고도  앞으로 많은 설정들이 매핑되기 시작한다.

이제 기본전략 설정까지 흐름을 알았으니, 실제 `CamelCaseToUnderscoresNamingStrategy` 가 호출되어 사용되는 흐름을 보자.

#### ㄴ. CamelCaseToUnderscoresNamingStrategy(이하 camel-strategy) 호출까지의 흐름

이어서 진행하다보면, EntityManagerFactory 객체를 생성하는 지점을 만난다. 
![[스크린샷 2025-05-25 오후 11.46.45 1.png]]
여기서 build()을 하게되면 연결된 database의 metadata 정보를 들고온다. build()을 따라 내부로 들어가면 `org.hibernate.boot.model.relational.Database` 을 생성해준다. 

![[스크린샷 2025-05-25 오후 11.49.55.png]]
Database에 physicalNamingStrategy도 설정해준다. 기본전략인 camel-strategy 로 초기화되어 있을텐데, 내부 필드를 초기화하고 setImpliciNamespaceName() 함수를 호출하면서 `처음으로 camel-strategy가 사용된다!! `

![[스크린샷 2025-05-25 오후 11.51.42.png]]

PhysicalNamingStrategy 인터페이스에 정의된 함수들은 모두 반환타입이 Identifier 임을 참고하자.
camel-strategy 의 함수는 apply() 함수를 통해 최종 반환하게 되니, apply() 함수의 생김새 정도만 잠시 보고가자.

![[스크린샷 2025-05-25 오후 11.56.46.png]]

이후에는 진짜로 Table 및 Column에서 사용되는 부분이다. 테이블명이나 컬럼명이 실제 database에는 어떻게 생성되었고, 이 이름들을 spring boot에서는 네이밍 전략을 통해 Identifier들을 Map으로 관리하며 서로 매칭시켜준다.
#### ㄷ. Table 및 Column 을 위한 CamelCaseToUnderscoresNamingStrategy 사용으로의 흐름

이제 Table을 생성하고, Column들도 생성해야한다. `org.hibernate.boot.model.internal.EntityBinder` 에서 bindTable을 하며 테이블들을 생성한다. 내부로 들어가면 `org.hibernate.boot.model.relational.Namespace` 에서 테이블명을 만드는데에 camel-strategy 전략이 활용됨을 알 수 있다.

![[스크린샷 2025-05-26 오전 12.06.05.png]]

이후로는 컬럼들을 생성(매핑) 해준다. `dueDate` 필드는 camel-strategy에 의해 `due_date`로 변환되는것을 볼 수 있다.

![[스크린샷 2025-05-26 오전 12.14.36.png]]

모든 Table, Column 등의 작업이 끝나고 나면 아래와 같은 Log들을 볼 수 있다. (참고용)
![[스크린샷 2025-05-26 오전 12.15.53.png]]

작업이 마치고 나면, 데이터베이스에서 테이블정보들을 가져온다. 만약 database에 테이블이 없는 최초의 상태면, 당연히 가져오는 정보들이 없다. 따라서 Entity 설계에 맞게 테이블이 생성해둔 상태로 확인해보기로 한다. (테스트를 위해 ddl-auto : update로 생성)

#### ㄹ. CamelCaseToUnderscoresNamingStrategy을 통해 생성된 Identifier와 database의 스키마와의 매핑으로의 흐름

이후 많은 과정을 거쳐오긴 하지만, `org.hibernate.tool.schema.extract.internal.DatabaseInformationImpl` 에서부터 확인한다. `getTablesInformation()` 함수가 호출되는데 database의 테이블, 컬럼 정보들을 가져와서, 네이밍 전략을 통해 생성해둔 Identifier와의 매핑을 시작한다. 

다음은 `getTablesInformation()` 함수의 내부로 들어온 후`org.hibernate.tool.schema.extract.internal.AbstractInformationExtractorImpl` 의 `getTables()` 함수의 일부이다.

![[스크린샷 2025-05-26 오전 12.41.58.png]]

lambda 식을 보면 쿼리 결과를 통해 `테이블 정보`을 추출하고, `column 정보`을 채우게된다.
1. extractNameSpaceTablesInformation()
2. populateTablesWithColumns()
위 2개의 람다 식 내부 로직으로 나누어 확인해보겠다.

##### 1. extractNameSpaceTablesInformation()

`org.hibernate.tool.schema.extract.internal.AbstractInformationExtractorImpl` 
![[스크린샷 2025-05-26 오전 12.35.17.png]]
테이블을 조회해서 , tables에 tableInformation을 추가해주고 있는데 tables은 `Map<String, TableInforamtion` 형태이다. 여기서 TableInformation의 구현체인 `TableInformationImpl` 클래스의 필드를 보면 `private Map<Identifier, ColumnInformation> columns = new HashMap<>(  );` 가 보이는데, 바로 camel-strategy을 통해 생성해둔 Identifier가 key로 사용되어, 매핑이 됨을 유추해볼 수 있다.

##### 2. populateTablesWithColumns()
모든 테이블 정보를 세팅해두었다면, 각 테이블의 컬럼 정보를 매칭해줄 차례이다.
![[스크린샷 2025-05-26 오전 12.56.53.png]]
마찬가지로 while문을 보면 현재 테이블의 컬럼 정보들을 추출해내고 있다.(addExtractedColumnInformation) 그리고 add 라는 키워드가 함수에 붙은것으로 보아 1번과정에서 생성된 TableInformation의 columns 필드를 채울 것 이라는 것을 유추해볼 수 있다.

여기까지 진행된 이후에는, 이제 database의 테이블 정보들과 위의 ㄷ. 흐름에서 생성해둔 TableInformation들과 매칭과정을 진행한다.

#### ㅁ. 정보 매칭
ㄹ. 흐름에서 `getTablesInformation()` 함수가 호출된다고 얘기했는데, 이 함수는 
`org.hibernate.tool.schema.internal.GroupedSchemaMigratorImpl`클래스의 `performTablesMigration()` 함수 내부에서 호출되고 있었다.

![[스크린샷 2025-05-26 오전 1.20.01.png]]
72번 라인에서 database의 테이블들을 얻어오고, for문으로 ㄷ. 에서의 TableInformation에 컬럼 정보까지 매칭하여 넣어주게된다.