# Step Away From The Database

[AfterArch]: src/site/markdown/images/after-arch.png "Image after-arch.png"
[BeforeArch]: src/site/markdown/images/before-arch.png "Image before-arch.png"
[BeforeDB1]: src/site/markdown/images/before-database-1.png "Image before-database-1.png"
[BeforeDB2]: src/site/markdown/images/before-database-2.png "Image before-database-2.png"
[BeforeMain1]: src/site/markdown/images/before-main-1.png "Image before-main-1.png"
[BeforeMain2]: src/site/markdown/images/before-main-2.png "Image before-main-2.png"
[BeforeMain3]: src/site/markdown/images/before-main-3.png "Image before-main-3.png"
[AfterHZ1]: src/site/markdown/images/after-hz-main-1.png "Image after-hz-main-1.png"
[AfterHZ2]: src/site/markdown/images/after-hz-main-2.png "Image after-hz-main-2.png"
[AfterHZ3]: src/site/markdown/images/after-hz-main-3.png "Image after-hz-main-3.png"
[AfterHZ4]: src/site/markdown/images/after-hz-main-4.png "Image after-hz-main-4.png"
[AfterHZ5]: src/site/markdown/images/after-hz-main-5.png "Image after-hz-main-5.png"
[AfterMain1]: src/site/markdown/images/after-main-1.png "Image after-main-1.png"
[AfterMain2]: src/site/markdown/images/after-main-2.png "Image after-main-2.png"
[AfterMain3]: src/site/markdown/images/after-main-3.png "Image after-main-3.png"

기존 데이터베이스에 연결된 어플리케이션에 헤이즐캐스트를 적용하는 사례를 순차적으로 소개합니다.

이예제는 스프링 JPA 예제와 [Spring Data Hazelcast](https://github.com/hazelcast/spring-data-hazelcast) 를 사용하여 속도와 복원력을 기존것에 추가한다.

개발자에게는 쉽겠지만 아키텍쳐에게는 굉장히 힘든 예제일 수도 있습니다.

## Hello World
이 어플리케이션은 간단한 영어 > 스페인어 번역을 제공합니다.

"_hello world_" 라고 입력하면 "_hola al mundo_" 이라고 답변합니다.

코드는 [여기](https://github.com/hazelcastkorea/codesamples/tree/master/hazelcast-integration/spring-data-jpa-hazelcast-migration)에 있습니다.

## 프로젝트 구성도
이코드는 메이븐 멀티 모듈 프로젝트입니다. 아래의 구성을 봐주세요.


```
├── after/
│   ├── after-domain/
│   ├── after-hz-main/
│   ├── after-jpa-repository/
│   ├── after-jpa-service/
│   ├── after-kv-repository/
│   ├── after-kv-service/
│   └── after-main/
├── before/
│   ├── before-domain/
│   ├── before-jpa-repository/
│   ├── before-jpa-service/
│   └── before-main/
└── database/
└── shared/
├── pom.xml
├── README.md
```

### After
'after' 모듈은 헤이즐캐스트가 적용된 이후의 예제입니다. 각 클래스는 Javadoc 안에 어떤것이 변환되었는지 기술되어있습니다.

### Before
이 모듈은 오리지널이므로 헤이즐캐스트가 없습니다. 기본 스프링 구조로 데이터베이스에 연결되어있습니다. 

### Database
Standalone 모듈 데이터베이스 입니다 따로 설치할 것은 없습니다.

### Shared
'before' 와 'after' 의 로그와 디버깅을 위한 쉐어드 오브젝트들의 모음입니다.


### \*\*Notes\*\*
탑 레벨에서 모든 빌드가 가능합니다. 'mvn install' 을 커맨드창에서 실행하시면 됩니다.

도메인 오브젝트들은 [Lombok](https://projectlombok.org) 을 사용해 getters, setters 를 적용하기 때문에 IDE 클래스패스에 Lombok 이 들어 있어야 합니다. [`Running Lombok`](https://projectlombok.org/features) 이곳을 참조하여 주세요.

헤이즐캐스트는 자바 8 이 꼭 필요하지 않지만 이예제에서는 필요하기 때문에 꼭 자바 8 버전을 설치해주세요.

## Before - Architecture
아래의 다이아그램 참조:

![Image of the architecture before Hazelcast is introduced][BeforeArch] 

매우 간단한 아키텍쳐 구조를 가지고 있습니다.

오른쪽에는 데이터베이스 테이블인 "*Noun*" 과 "*Verb*" 를 보실수 있습니다.

왼쪽에는 2개의 어플리케이션이 있습니다. 일반적인 스프링 기반 모듈로서 레이어로 구성됩니다. "*Service*" 레이어와 JPA 를 사용하는 "*Repository*" 레이어가 "*Domain Model*" 에 대한 액세스를 캡슐화합니다.

어플리케이션에 두개의 인스턴스가 표시되지만 하나 또는 여려 개일 수 있습니다. 여기에는 하나의 데이터베이스만 있습니다.

어플리케이션은 JDBC 를 사용하여 데이터베이스에 연결됩니다.

## Step 1 - 데이터베이스

첫 번째 단계는 RDBMS 또는 SQL 데이터베이스를 실행하는 것입니다.
이 프로젝트는 서버 모드에서 [HSQLDB](http://hsqldb.org/) 을 사용합니다.
이것을 사용하시거나 원하시는 다른것을 사용하셔도 무방합니다.

### Using HSqlDB

'database' 모듈은 HSQLDB 의 인스턴스를 다음과 같이 실행이 가능한 jar 를 빌드합니다.

#### The tables

두개의 테이블은 명사와 동사를 위해  `src/main/resources/schema.sql` 에 의해 생성됩니다.

테이블 "*Noun*" 은 다음과 같이 정의 됩니다.
```
CREATE TABLE IF NOT EXISTS noun (
	id                     INTEGER PRIMARY KEY,
	english                VARCHAR(20),
	french                 VARCHAR(20),
	spanish                VARCHAR(20)
);
``` 
기본 키에 대한 *id* 열과 영어, 프랑스어, 스페인어의 명사.

마찬가지로 "*Verb*" 테이블은 다음과 같습니다.
```
CREATE TABLE IF NOT EXISTS verb (
	id                     INTEGER PRIMARY KEY,
	english                VARCHAR(20),
	french                 VARCHAR(20),
	spanish                VARCHAR(20),
	tense                  INTEGER
);
```
"*Verb*" 는 "*Noun*" 과 같은 열을 가지고 있으며, 시제를 나타내는 단순한 숫자 방식을 사용합니다.
0은 과거 시제, 1은 시제, 2 시제는 미래 시제입니다. 분명히 더 좋은 방법이 있지만 이것은 예제일 뿐입니다.

#### The data
"*Noun*" 과 "*Verb*" 테이블은 `src/main/resources/data.sql` 에 의해 채워지며, 이는 명료하게 4개의 명사와 1개의 동사를 내용으로 추가합니다.

#### Try It

`java -jar database/target/database-0.1-SNAPSHOT.jar` 를 커맨드창에서 실행하십시오.

이것은 스프링 부트를 사용하여 모든것을 실행 가능한 Jar 로 묶고, 스프링쉘은 명령을 처리하기위해 사용합니다.
실행후에는 아래와 같이 나와야 합니다.
![Image of the database after start-up][BeforeDB1] 

스프링쉘에 익숙하지 않다면 "_help_" 로 명령어들을 보실수 있고 "_quit_" 로 나갈 수 있습니다.

![Image of the database showing some content][BeforeDB2] 

마지막으로 HSQLDB 는 데이터베이스 디렉토리를 유지하기 위해 현재 디렉토리에 'mydatabasedb.*' 파일을 생성합니다.
데이터베이스가 실행중이지 않거나 그대로두면 모두 삭제가 가능합니다.

### Using your database
HSQLDB 가 아닌 다른 데이터베이스를 사용하셔도 됩니다.


이것을 하기위해 아래의 항목들을 수행해야합니다:
* 이름, 구조 및 내용이 같은 테이블 생성
* 올바른 드라이버와 종속물을 로드하기 위해 `pom.xml` 을 수정합니다.
* 모든 `application.properties` 를 새로운 JDBC 연결 문자열로 업데이트 하세요.
* `BeforeTranslatorConfiguration.java` 와 `HazelcastServerConfiguration.java` 를 업데이트하여 드라이버 클래스를 설정하세요.

어려운 것은 아니지만 오류가 발생하기 쉽습니다. 제공된 데이터베이스를 먼저 사용해 보십시오.

## Step 2 - 마이그레이션 전
이 어플리케이션은 메이븐 모듈에서 빌드 된 독립 실행형 jar 파일로 존재합니다.

### `before-domain`
이 모듈은 데이터 모델을 정의합니다.

세가지 클래스가 있습니다:
* `Noun` 명사의 영어, 스페인 및 프랑스어 표현을 나타내는 명사.
* `Tense` 동사의 과거, 현재 또는 미래 여부를 나타내는 열거 형입니다.
* `Verb` 동사의 영어, 스페인어, 프랑스어 표현을 가지고 있으며 시제입니다.

`Noun` 과 `Verb` 는 표준 어노테이션 `@Entity` 와 `@Id` 를 모두 사용하여 스프링 데이터 JPA 를 데이터베이스 테이블과 연관시킵니다.

`Verb` 는 아래와 같습니다.
```
@Entity
public class Verb implements Serializable {
	
	@javax.persistence.Id
	@org.springframework.data.annotation.Id
	private int		id;
	
	private String	english;
	private String	french;
	private String	spanish;
	private Tense	tense;
```
`Noun` 는 `tense` 필드가 없다는 점을 제외하고는 동일합니다.

### `before-jpa-repository`
이 모듈은 데이터 액세스를 위해 JPA 를 사용하여 데이터 모델에 액세스하는 스프링 저장소를 정의합니다.

`NounJPARepository` 와 `VerbJPARepository` 둘 다 `findByEngish(String s)` 메소드가 정의 되어있습니다.

예:
```
public interface VerbJPARepository extends CrudRepository<Verb, Integer> {
	public Verb findByEnglish(String s);
}
```

이 메소드는 스트링에 쿼리 메소드를 작성하여 데이터베이스 테이블에서 `English` 행에 일치하는 항목을 검색하도록 지시합니다.

읽기와 쓰기를 위한 다른 기본적인 방법은 `CrudRepository` 에서 상속 받았습니다.

### `before-jpa-service`
다음 모듈의 이름은 `before-jpa-service` 입니다. 그러나 이 명명법은 명확성을 위한 것일뿐 일반적으로 캡슐화 에서는 서비스가 JPA 를 사용하고 있는것을 알 필요가 없습니다.

이 서비스는 번역을 위한 비즈니스로직을 제공합니다.

단순화를 위해 이는 단지 문자열 교체 일 뿐입니다. 인풋의 각 단어는 if 가 영어 명사인지 검사하고, 그렇다면 스페인어가 대체됩니다. 영어 명사가 아닌 경우 동사도 똑같이 시도됩니다. 일치하는 항목이 없으면 "?" 사용.

코딩은 그다지 중요하지 않습니다. 이 모든 것은 repositories 검색에 사용되는 방법을 보여줍니다.

### `before-jpa-main`
마지막 모듈은 서비스를 내장하고 명령 줄 상호 작용을 제공하는 기본 모듈입니다.

`application.properties` 파일은 데이터베이스에 대한 연결을 지정합니다 -- Step 1 에서 생성된 데이터 베이스

### 시도해보기
Step 1 의 데이터베이스가 실행 중이라고 가정하고 다음을 사용하여 어플리케이션의 인스턴스를 시작합니다.
`java -jar before/before-main/target/before-main-0.1-SNAPSHOT.jar` 로 시작.

아래와 같은 결과값이 보여야 합니다:
![Image of the before application immediately after started][BeforeMain1] 

원하는 경우 다른 창에서 어플리케이션의 두번째 인스턴스를 시작하세요.

이제 일부 텍스트를 번역하기 위해 하나 또는 둘 모두에 명령을 입력 해보세요.
`translate --text "hello world"` 를 실행해보세요.

아래와 같은 결과값이 보여야 합니다:
![Image of the before application after a sentence has been translated][BeforeMain2] 

"*Hello*" 는 명사 또는 동사가 아니므로 번역되지 않습니다.
"*World*" 는 "*Mundo*" 가 됩니다.

마지막으로, 데이터베이스를 __stop__ 하고 번역을 다시 시도하세요.

아래와 같은 결과값이 보여야 합니다:
![Image of the before application when the database is down][BeforeMain3] 

데이터베이스를 사용할 수 없으므로 액세스가 실패하고 오류가 기록됩니다 (`JDBCConnectionException`).

## 이전 - 요약
이것은 매우 간단한 어플리케이션 이지만 일반적인 스프링 개발 방식을 보여줍니다.

데이터는 SQL 데이터베이스에 보관됩니다.

데이터 모델은 데이터 테이블에 대한 어노테이션으로 정의 된 매핑을 가지고 있으며 스프링의 @Repository 를 통해 JPA 에 의해 액세스되며 이 액세스는 스프링의 @Service 에 숨겨져 있습니다.

여러 어플리케이션을 실행할 수 있지만 데이터베이스가 데이터가 저장되는 유일한 위치이므로 데이터베이스를 사용할 수 없으면 어플리케이션도 사용할 수 없게됩니다.

## 이후 - 아키텍쳐
다음 다이어그램을 참조하십시오:

![Image of the architecture after Hazelcast is introduced][AfterArch] 

이것은 단순한 아키텍쳐이지만 이전보다 몇 가지 더 많은 화살표가 있습니다.

오른쪽에는 여전히 "*Noun*" 과 "*Verb*" 라는 테이블이있는 데이터베이스가 있습니다.

왼쪽에는 어플리케이션이 있고, 다시 두개의 인스턴스가 있습니다. 이전의 변경 사항은 JPA "*Repository*" 가 키밸류 "*Repository*" 가 되었다는 것입니다.

가운데에는 데이터베이스 컨텐츠의 캐시 된 사본을 보유한 3개의 헤이즐캐스트 서버가 표시됩니다.

왼쪽의 각 어플리케이션에는 모든 헤이즐캐스트 서버에 대한 헤이즐캐스트 연결이 있습니다.

중간에있는 각 헤이즐캐스트 서버에는 데이터베이스에 대한 JDBC 연결이 있습니다.

## 이후 - 마지막 목표
여기서 우리가해야 할 일은 헤이즐캐스트를 어플리케이션과 데이터베이스 사이의 캐싱 레이어로 소개하는 것입니다.

몇 가지 변경 사항이 있지만 모두 작고 쉽습니다.

비교하기 쉽도록 새로운 버전은 이전 버전과 구별되는 상상의 'after' 의 하위 트리에 있습니다.

## Step 3 - 데이터 모델 변경
`after`에 있는 `after-domain` 은 `before.` 에 `before-domain` 을 수정 한 버전입니다. 

### `after-domain`
데이터 모델 변경이 없습니다.

우리가 하는 일은 "*Noun*" 와 "*Verb*" 에 하나의 어노테이션인 `@KeySpace` 를 소개하는 것입니다.

그래서 "*Noun*" 다음과 같이 변경됩니다.
```
@Entity
public class Noun implements Serializable {
	
	@javax.persistence.Id
	@org.springframework.data.annotation.Id
	private int		id;
	private String	english;
	private String	french;
	private String	spanish;
```
to
<pre>
@Entity
<b>@KeySpace</b>
public class Noun implements Serializable {
	
	@javax.persistence.Id
	@org.springframework.data.annotation.Id
	private int		id;
	private String	english;
	private String	french;
	private String	spanish;
</pre>

`@Entity` 어노테이션은 스프링에게 이 객체가 SQL 테이블에 저장 될 수 있음을 알려줍니다.
`@KeySpace` 를 추가하면 스프링에 이 객체가 자바 맵에 저장 될 수 있음을 알립니다.

*NOTE*: 자바 맵은 키밸류 저장소이며 전체 맵 콘텐츠는 키밸류 또는 키스페이스 입니다.

## Step 4 - 저장소 변경
이제 두 개의 저장소 모듈이 있습니다.

`after-jpa-repository` 는 도메인 모델에 대한 JPA 액세스를 위한 것입니다.
`after-kv-repository` 는 도메인 모델에 대한 키밸류 접근을 위한 것입니다.

### `after-jpa-repository`
이 저장소는 헤이즐캐스트에서 SQL 데이터를 메모리에 로드 하는데 사용됩니다.

정의는 다음과 같이 변경됩니다.

AS-IS
```
public interface VerbJPARepository extends CrudRepository<Verb, Integer> {
	public Verb findByEnglish(String s);
```

TO-BE
<pre>
public interface VerbJPARepository extends CrudRepository<Verb, Integer> {
	<b>
	@Query("SELECT v.id FROM Verb v")
    public Iterable<Integer> findAllId();
    </b>
</pre>

오리지날 쿼리 `findByEnglish()` 가 제거 되었습니다. 헤이즐캐스트가 데이터베이스에서 검색하지 않기 때문에 필요하지 않습니다.

헤이즐캐스트는 데이터베이스의 모든 것을 로드합니다. 그래서 `findAllId()` 가 필요합니다.
쿼리를 사용하여 데이터베이스의 모든 행에 대한 ID 열을 찾습니다. 이것과 일치하는것은 없으므로 이 메소드에 SQL 쿼리를 사용하여 스프링이 무엇을 해야하는 파악할 수 있도록 어노테이션을 해야합니다.

### `after-kv-repository`
이것은 어플리케이션이 데이터에 액세스하는데 사용할 새로운 저장소 입니다.
JPA 저장소를 사용하는 대신 헤이즐캐스트에서 가져옵니다.

그러나 Step-2 에서의 저장소와 매우 유사하게 보입니다.
```
public interface VerbKVRepository extends HazelcastRepository<Verb, Integer> {
	public Verb findByEnglish(String s);
```

정말로 다른 점은 저장소가 JPA 의 `CrudRepository` 대신에 `HazelcastRepository`를 확장한다는 것입니다.
이는 어플리케이션이 헤이즐캐스트에 연결하고 데이터베이스에 직접 연결하지 않기 때문입니다.

`findByEnglish()` 쿼리는 Step-4 에서 JPA 저장소에서 제거 된 것입니다.

어플리케이션은 여전히 수행중인 영어 단어를 헤이즐캐스트를 상대로 검색합니다. 따라서 여기에 쿼리 매소드를 옮겨야합니다.

## Step 5 - 서비스 변경
우리는 각각의 저장소 모듈을 래핑하는 두개의 서비스 모듈을 가지고 있습니다.

### `after-jpa-service`
이 서비스는 헤이즐캐스트 서버가 테이블 컨텐츠를 데이터베이스 컨텐츠로 초기로드하는데 사용됩니다.

4가지 메소드가 정의되며 모두 헤이즐캐스트 서버 레이어에 대해 새롭게 정의 됩니다.

```
	public Noun findNoun(Integer id)
	public Iterable<Integer> findNounIds()
	
	public Verb findVerb(Integer id)
	public Iterable<Integer> findVerbIds()
```

이것들은 모든 명사에 대한 "*id*" 열을 찾고 각각의 명사를 로드하는 방법을 정의 하며 동사도 동일합니다.

### `after-kv-service`
다시 "*TranslationService*" 입니다. 이것은 이전과 동일하지만 지금은 JPA 저장소 대신 키밸류 저장소를 사용합니다.

그래서 이것은 AS-IS 에서 TO-BE 로 바뀝니다

AS-IS
```
public class TranslationService {
	@Autowired
	private NounJPARepository nounJPARepository;
	
	@Autowired
	private VerbJPARepository verbJPARepository;
```

TO-BE
<pre>
public class TranslationService {
	@Autowired
	<b>
	private NounKVRepository nounKVRepository;
	</b>
	
	@Autowired
	<b>
	private VerbKVRepository verbKVRepository;
	</b>
</pre>

다른 모든 코딩과 비지니스 로직은 변경되지 않습니다.

## Step 6 - 메인 프로그램 변경
`after` 예제에는 두가지 유형의 메인 프로그램이 있습니다. 어플리케이션 자체는 `after-main` 그리고 새로운 `after-hz-main` 이 있습니다.

### `after-hz-main`
이 프로세스는 완전히 새로운 것입니다.

`src/main/resources/hazelcast.xml` 파일 을 설정값으로 사용하는 헤이즐캐스트 서버 인스턴스를 시작합니다.

이 구성에는 두 부분이 있습니다.

* 127.0.0.1 포트 5701 에서 다른 헤이즐캐스트 프로세스를 찾도록 지정하는 네트워크 섹션 밑 헤이즐캐스트 서버의 클러스터를 형성하기 위해 발견 된 것들과 합류 할 수 있습니다.

* 자바 클래스에 대한 콜백을 사용하여 명사와 동사를 저장하는데 사용할 자바 맵을 지정하는 데이터베이스에서 이 맵을 채웁니다.

### `after-main`
이 어플리케이션은 `before-main` 과 비슷합니다.

그것이 어떻게 다른지는 헤이즐캐스트 설정파일 `src/main/resources/hazelcast-client.xml` 을 가지고 있다는 것입니다.
헤이즐캐스트 클러스터 `after-hz-main` 에서 데이터를 찾을 수 있는 곳을 제시합니다.

#### 시도해보기 
Step 2 에서 데이터베이스를 중지했다고 가정하면 `java -jar database/target/database-0.1-SNAPSHOT.jar` 를 실행하여 데이터베이스를 다시 시작하십시오.

이제 헤이즐캐스트 서버를 시작하기 위해 `java -jar after/after-hz-main/target/after-hz-main-0.1-SNAPSHOT.jar` 를 사용하여 독립 실행형 자바 어플리케이션으로 실행하십시오.

아래와 같은 결과가 보여야합니다:

![Image of the first Hazelcast in the after application][AfterHZ1] 

위에서 참조할 줄은 아래와 같습니다.
<pre>
Members [1] {
	<b>Member [127.0.0.1]:5701 - 240652e9-b986-4e54-bf66-ff63650289fb this</b>
}
</pre>
이것은 헤이즐캐스트 서버 클러스터가 형성되었음을 보여주며 이 프로세스는 해당 클러스터의 구성원입니다. 현재는 이것이 유일한 멤버입니다.

하나의 클러스터는 일반적으로 쓸모가 없으므로 다른 창에서 동일한 명령을 사용하여 다른 헤이즐캐스트 서버를 시작하십시오 
`java -jar after/after-hz-main/target/after-hz-main-0.1-SNAPSHOT.jar`.

아래와 같은 결과가 보여야합니다:

![Image of the second Hazelcast in the after application][AfterHZ2] 

시작 메시지의 끝 부분에있는 클러스터 구성원 섹션을 보면 다음과 같은 내용을 볼 수 있습니다.
<pre>
Members [2] {
	Member [127.0.0.1]:5701 - 240652e9-b986-4e54-bf66-ff63650289fb
	<b>Member [127.0.0.1]:5702 - cd7ca2f0-181b-4e8d-8a6b-20fd9dc839dc this</b>
}
</pre>
이제 클러스터에 두개의 프로세스가 있으며 이 프로세스는 두번째 프로세스입니다(포트 5702).

커맨드 라인에서 `debugHZ` 명령을 사용하여 어떤 내용이 헤이즐캐스트에 있는지 보십시오.

![Image of the second Hazelcast displaying it has no data yet][AfterHZ3] 

현재 코딩에서 지연로드를 선택 했으므로 로드 된 데이터베이스 컨텐츠는 없습니다.

이제 연결된 헤이즐캐스트 서버 레이어가 있습니다. 번역 어플리케이션을 시작할 수 있습니다.
`java -jar after/after-main/target/after-main-0.1-SNAPSHOT.jar` 를 실행하면 아래와 같이 나옵니다: 

![Image of the after application just having started][AfterMain1] 

여기 라인에서:
<pre>
INFO: hz.client_0 [dev] [3.8-SNAPSHOT] <b>HazelcastClient</b> 3.8-SNAPSHOT (20161128 - 4a00c4f) is <b>CLIENT_CONNECTED</b>
</pre>
어플리케이션 프로세스가 헤이즐캐스트 클라이언트로 작동하고 서버에 성공적으로 연결되었는지 확인합니다.


이제 "*hello world*" 를 다시 번역 해 봅시다.

![Image of the after application after the first translation attempt][AfterMain2] 

지연로딩 때문에 우리가 해야 할 일을 요구할 때 시스템이 작동하게 됩니다. 이 시점에서 일부 로그 메시지가 나타나지만 "*? mundo*" 라는 결과 값이 나타납니다.

헤이즐캐스트 서버를 살펴 보겠습니다. 첫번째는 헤이즐캐스트 서버입니다:

![Image of the first Hazelcast showing data loading logging][AfterHZ4] 

우리가 위에서 본 것은 로그 메시지 입니다.
<pre>
18:52:10.959 INFO  c.h.s.s.d.m.<b>MyNounLoader - load(3)</b>
18:52:10.962 INFO  c.h.s.s.d.m.<b>MyNounLoader - load(4)</b> 
</pre>

`MyNounLoader` 에 의해 생성되는 명사 3 ("<i>milk</i>") 과 명사 4 ("<i>world</i>") 입니다. 

**NOTE** 로딩 워크로드는 헤이즐캐스트 서버에 분산되어 있습니다. 각각의 명사 와 동사 중 일부를 로드합니다. 집합적으로 각 명사와 동사는 한번만 로드됩니다. 어느 헤이즐캐스트 서버가 어떤 일을 하는지 쉽게 예측할 수는 없지만 두개의 멤버로 실행하면 각각 로드의 절반 정도를 수행하게 됩니다.

로딩이 실행 되었다면 우리는 `debugHZ` 명령을 다시 시도하여 내용을 볼 수 있습니다.

![Image of the first Hazelcast displaying it has data now][AfterHZ5] 

마지막으로 데이터베이스르 **중지** 시키고 번역을 다시 시도 하십시오.

![Image of the after application after the second translation attempt][AfterMain3] 

콘텐츠가 헤이즐캐스트에 있기 때문에 번역 서비스는 계속 작동합니다. 오류가 발생하지 않습니다.

## 이후  - 요점 정리
이것은 여전히 독립적인 캐싱 레이어가 도입된 일반적인 스프링 어플리케이션입니다.

데이터는 여전히 SQL 데이터베이스에 보관됩니다.

어플리케이션의 데이터 액세스는 한줄을 변경하여 JPA 스타일에서 키밸류 스타일로 변경되어 스프링 저장소를 통해 계속 수행됩니다.

데이터베이스가 오프라인 상태가되면 어플리케이션은 알지 못해도 계속 사용할 수 있습니다.

## 추가 작업
이 예제는 몇가지 핵심 원칙을 보여주지만 여러가지 방법으로 확장 될 수 있습니다.

* 인터젝션은 테이블 및 도메인 객체로 추가 될 수 있습니다. 그런 다음 "*hello*" 를 "*hola*" 로 번역되며 "*hello world*" 를 번역하면 예상한대로 작동합니다.

현실적으로는 대부분의 가치는 데이터베이스 로직을 향상 시킴으로서 발생합니다.

* `before-main` 과 `after-hz-main` 은 데이터베이스를 사용할 수 없다면 예외를 던집니다. 헤이즐캐스트 서버가 클러스터에 합류하는 것이 유효 할 것이고 그 후에 그것은 어떤 데이터를 `after-main` 에 제공 할 수 있습니다.

* 재연결 로직에 데이터베이스 연결이 추가 될 수 있으며 데이터베이스에 일시적인 장애 발생시에 데이터베이스를 다시 시작할 필요가 없습니다. 

## 이점
이 에제는 헤이즐캐스트를 소개 할때 두가지 중요한 이점을 보여줍니다.

### 탄력성(resilience)
예제에서 충분히 탄력성에 대해 보여주었다고 생각됩니다. 데이터베이스가 오프라인 상태가 되어도 어플리케이션은 계속 작동 할 수 있게 됩니다.

헤이즐캐스트 서버조차도 오프라인이 될 수 있으며 어플리케이션은 계속 작동 할 수 있습니다. 이 에제에서 하나의 헤이즐캐스트 서버만 있으면 충분합니다.

### 확장성(scaling)
덜 명확한 것은 확장성이지만 명사와 동사가 어떻게 헤이즐캐스트에 로드 되는지 자세히 살펴 보십시오.

두개의 헤이즐캐스트 서버를 실행하면 각각의 데이터가 절반 정도의 부하가 생기는 것을 발견하게 됩니다. 네 개의 명사가 있으므로 각각 두번입니다.

어떤 이유로 데이터의 절반을 차지하는 두 서버의 로드가 너무 많으면 서버를 하나 추가하여  세개에서 실행할 수 있으며 각 서버는 3분의 1로 로드가 나누어지게 됩니다. 또는 네번째 서버를 추가해서 4분의 1로 나눌수도 있습니다.

## 단점
물론 모든 이점이 있는 것은 아니며 몇가지 단점이 있습니다.

이전보다 더 많은 JVM 을 실행해야하고 빌드 및 배포가 좀 더 복잡해지며 적어도 하나의 개발자가 네트워킹 대역폭에 대한 이해도가 필요합니다.

다른 단점은 JPA 와 키밸류가 충분히 상호 교환 할 수 없다는 것입니다. 특정 JPA 동작에 의존하는 방식으로 빌드를 크게 최적화 한 경우 변경하기가 쉽지 않게됩니다.

## 요약
JPA 를 사용하여 데이터베이스에 연결하는 어플리케이션은 데이터베이스의 중단(장에)에 노출됩니다.

어플리케이션이 표준 코딩을 따르는 경우 헤이즐캐스트를 분산 캐싱 레이어로 도입하면 코딩이 거의 필요하지 않습니다.

헤이즐캐스트가 도입되면 확장성(scaling)과 탄력성(resilience) 의 즉각적인 이점이 있습니다.

데이터베이스에 증가하는 쿼리의 수와 종류를 줄이면 데이터베이스를 저렴한 제품으로(디비 소프트웨어 + 하드웨어) 대체가 가능 해집니다.

## 리소스
코드는 [여기](https://github.com/hazelcastkorea/codesamples/tree/master/hazelcast-integration/spring-data-jpa-hazelcast-migration) 에서 받아 볼 수 있습니다.

