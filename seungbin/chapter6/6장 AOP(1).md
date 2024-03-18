AOP를 이용해 더욱 세련되고 깔끔한 방식으로 바꿀 것이다. 그 과정에서 이것이 왜 필요한 지 알아보자

AOP란?

관점 지향은 쉽게 말해어떤 로직을 기준으로 핵심적인 관점, 부가적인 관점으로 나누어서 보고 그 관점을 기준으로 각각 모듈화하겠다는 것이다. 여기서 모듈화란 어떤 공통된 로직이나 기능을 하나의 단위로 묶는 것을 말한다.

## 6.1 트랜잭션 코드의 분리

현재 UserService는 트랜잭션 경계설정을 위해 넣은 코드가 많은 범위를 차지하고 있는 모습을 고칠 것이다.

### 6.1.1 메소드 분리

경계설정과 비즈니스 로직이 공존하는 메소드

```java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}
```

이 코드는 비즈니스 로직 코드를 사이에 두고 트랜잭션 시작과 종료를 담당하는 코드가 앞뒤에 위치하고 있는 두 가지 종류의 코드로 구분되어 있다.

또한 비즈니스 로직 코드에서 직접 DB를 사용하지 않기 때문에 서로 주고 받는 정보가 없다.

그렇기에 비즈니스 로직을 담당하는 코드를 메소드로 추출해서 독립시킨다.

```java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
        upgradeLevelsInternal();
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}

private void upgradeLevelsInternal() {
    List<User> users = userDao.getAll();
    for (User user : users) {
        if (canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}

```

이렇게 코드를 분리할 수 있다.

DI 적용을 이용한 트랜잭션 분리

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/4b08ede6-7aa6-4e93-ab2d-45d0bddd0a3f/Untitled.png)

지금의 문제는 강한 결합도로 연결되어 있기 때문에 이 부분을 해결해야 한다.

그렇기에 이제 UserService를 인터페이스로 만들고 기존 코드는 UserService 인터페이스의 구현 클래스를 만들어넣도록 한다.

그러면 직접 결합을 막아주고 유연한 확장이 가능하게 만들어 진다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/37841b97-083d-49dd-b8e3-e9e447ae08b2/Untitled.png)

DI를 적용하는 방법을 쓰는 이유는 일반적으로 구현 클래스를 바꿔가면서 사용하기 위해서다.

한 번에 두 개의 UserService 인터페이스 구현 클래스를 동시에 이용하는 것도 가능할 것 같다. 하지만 클라이언트가 UserService의 기능을 제대로 이용하려면 트랜잭션이 적용돼야 한다.

그래서 바뀐 구조는

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/f50b6420-4240-45c2-bac5-79902fbf4e36/Untitled.png)

UserService 인터페이스 도입

기존 클래스를 Impl로 변경한다음 UserService 인터페이스로 메서드로 만든다.

대부분 클래스의 내용을 유지한다. 다만 인스턴스 변수와 수정자 메소드, 또 트랜잭션 관료 코드도 전부 제거하며 코드를 구성한다.

```java
package springbook.user.service;

import org.springframework.transaction.annotation.Transactional;

public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;

    public void upgradeLevels() {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
    }
...

```

이렇게 하여 UserDao라는 인터페이스를 이용하고, User라는 도메인 정보를 가진 비즈니스 로직에만 충실한 코드로 바꿨다.

분리된 트랜잭션 기능

이제 비즈니스 트랜잭션 처리를 담은 UserServiceTx를 만들것이다.

위임 기능을 가진 UserServiceTx 클래스

```java
package springbook.user.service;

public class UserServiceT implements UserService {
    UserService userService;

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    // UserService 오브젝트에 모든 기능을 위임한다.
    public void add(User user) {
        userService.add(user);
    }

    // UserService 오브젝트에 모든 기능을 위임한다.
    public void upgradeLevels() {
        userService.upgradeLevels();
    }
}

```

UserServiceTx는 사용자 관리라는 비즈니스 로직을 전혀 갖지 않고 고스란히 다른 UserService 구현 오브젝트에 기능을 위임한다.

transactionManager라는 이름의 빈으로 등록된 트랜잭션 매니저를 DI로 받아뒀다가 트랜잭션 안에서 동작하도록 만들어줘야 하는 메소드 호출의 전과 후에 필요한 트랜잭션 경계설정 API를 사용해주면 된다.

```java
public class UserServiceT implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    public void add(User user) {
        this.userService.add(user);
    }

    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            userService.upgradeLevels();
            this.transactionManager.commit(status);
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}

```

UserService에서 트랜잭션 처리 메소드와 비즈니스 로직 메소드를 분리했을 때 트랜잭션을 담당한 메소드와 거의 한 메소드가 되었으며 추상화된 트랜잭션 구현 오브젝트를 DI 받을 수 있도록 PlatformTransactionManager 타입의 프로퍼티도 추가되었다.

트랜잭션 적용을 위한 DI 설정

이제는 설정파일을 수정할 것이다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/2b9af421-a78c-4a86-9422-9139cb9aea11/Untitled.png)

이렇게 구성을 바꾸기 위해서는 UserServiceImpl 빈이 각각 의존하도록 프로퍼티 정보를 분리한다.

```java
<bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager" />
    <property name="userService" ref="userServiceImpl" />
</bean>

<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
    <property name="userDao" ref="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```

이제 클라이언트는 UserServiceTx 빈을 호출해서 사용하도록 만들어야 한다. 따라서 userService라는 대표적인 빈 아이디는 UserServiceTx 클래스로 정의된 빈에게 부여해준다.

userServiceImpl인 빈을 DI하게 만든다.

트랜잭션 분리에 따른 테스트 수정

UserService라는 클래스를 직접 사용하는 테스트 구조에 각종 의존 오브젝트를 테스트용 DI 기법을 이용해 바꿔치기해서 사용했으니 테스트에서도 적합한 타입과 빈을 사용하도록 변경해야 한다.

@Autowired은 기본적으로 타입을 이용해 빈을 찾지만 만약 타입으로 하나의 빈을 결정할 수 없는 경우에는 필드 이름을 이용해 빈을 찾는다.

따라서 UserServiceTest에서 다음과 같이 설정해주면 userService인 빈이 주입될 것이다.

```java
@Autowired UserService userService;
```

이후 UserServiceImpl 클래스로 정의된 빈을 하나 더 가져와야 한다.

그런데 앞 장에서 만든 MailSender 목 오브젝트를 이용한 테스트 해서는 직접 DI해줘야 할 필요가 있었다.

→ 즉 MailSender를 DI 해줄 대상을 구체적으로 알고 있어야 하기 때문에 UserServiceImpl 클래스의 오브젝트를 가져올 필요가 있다.

목 오브젝트를 이용해 수동 DI를 적용하는 테스트라면 어떤 클래스의 오브젝트인지 분명하게 알 필요가 있다. 따라서 다음과 같이 @Autowired를 지정해서 해당 클래스로 만들어진 빈을 주입받도록 한다.

→ 그래야만 MockMailSender를 설정해주기 위한 수정자 메소드에 접근할 수 있기 때문이다.

```java
@Autowired UserServiceImpl userServiceImpl;
```

다음은 upgradelLevels() 테스트 메소드인데 별도로 가져온 userServiceImpl 빈에 해주는 코드로 수정해야 한다.

```java
@Test
public void upgradeLevels() throws Exception {
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);
}

```

upgradeAllOrNothing() 테스트는 수정해야 한다.

→ 직접 테스트용 확장 클래스도 만들고 수동 DI도 적용하고 한 만큼, 바뀐 구조를 모두 반영해주는 작업이 필요하다.

TestUserService가 트랜잭션 기능은 빠진 UserServiceImpl을 상속하도록 해야 한다.

그래서 우리는 TestUserService 오브젝트를 UserServiceTx 오브젝트에 수동 DI시킨 후에 트랜잭션 기능까지 포함된 UserServiceTx의 메소드를 호출하면서 테스트를 수행하도록 해야 한다.

트랜잭션 테스트용으로 정의한 TestUserService 클래스는 이제 UserServiceImpl 클래스를 상속하도록 바꿔주면 된다.

```java
static class TestUserSErvice extends UserServiceImpl {
```

트랜잭션 경계설정 코드 분리의 장점

1. 이제 비즈니스 로직을 담당하고 있는 UserServiceImpl의 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용에는 전혀 신경쓰지 않아도 된다.
2. 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있다는 것이다.

## 6.2 고립된 단위 테스트

작은 단위의 테스트는 당연하게 논리적인 오류를 찾기가 쉽기 때문에 좋다.

### 6.2.1 복잡한 의존관계 속의 테스트

UserService의 구현 클래스들이 동작하려면 세 가지 타입의 의존 오브젝트가 필요하다. 

1. UserDao 타입의 오브젝트를 통해 DB와 데이터를 주고받는다.
2. MailSender를 구현한 오브젝트를 이용해 메일을 발송해야 한다. 
3. 트랜잭션 처리를 위해 PlatformTransactionManager과 커뮤니케이션이 필요하다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/20350121-5c88-4b67-94e0-b77bdc5315e5/Untitled.png)

→ 결국 하나의 비즈니스 로직 코드를 테스트 해야하는데 지금은 전부 다 끌어와서 테스트 하게 된다는 점이다.

### 6.2.2 테스트 대상 오브젝트 고립시키기

테스트를 의존 대상으로부터 분리해서 고립시키는 방법은 테스트를 위한 대역을 사용하는 것이다.

테스트를 위한 UserServiceImpl 고립

고립된 테스트가 가능하도록 UserService를 재구성해보면 아래 그림 같은 구조가 될 것이다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/3b479ef1-2a06-4eb8-b13a-b56f81dde5a9/Untitled.png)

UserDao는 단지 테스트 대상의 코드가 정상적으로 수행되도록 도와주기만 하는 스텁이 아니라, 부가적인 검증 기능까지 가진 목 오브젝트로 만들었다. 그 이유는 고립된 환경에서 동작하는 upgradeLevels()의 테스트 결과를 검증할 방법이 필요하기 때문이다.

UserServiceImpl의 upgradeLevels() 메소드는 리턴 값이 없는 void형이다.

UserServiceImpl은 아무리 기능이 수행돼도 그 결과가 DB 등을 통해서 남지 않으니 검증하기 힘들다.

→ 이럴 땐 테스트 대상인 UserServiceImpl과 그 협력 오브젝트인 UserDao에게 어떤 요청을 했는지를 확인하는 작업이 필요하다.

즉 DB에 결과가 반영되지는 않더라도 update 메소드를 호출하는 것을 확인할 수 있다면 결국 DB에 반영될 것이라고 결론을 내릴 수 있기 때문이다.

고립된 단위 테스트 활용

UserServiceTest의 upgradeLevels() 테스트에 적용해본 코드이다.

```java
@Test
public void upgradeLevels() throws Exception {
    // 데이터베이스의 모든 사용자 데이터를 삭제하고 테스트 데이터 추가
    userDao.deleteAll();
    for (User user : users)
        userDao.add(user);

    // 테스트를 위한 MockMailSender 설정
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);

    // UserService의 upgradeLevels() 메소드 실행
    userService.upgradeLevels();

    // 사용자 레벨 업그레이드 여부 확인
    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), true);
    checkLevelUpgraded(users.get(2), false);

    // 데이터베이스에 저장된 결과 확인
    checkLevelUpgraded(users.get(3), true);
    checkLevelUpgraded(users.get(4), false);

    // MockMailSender를 이용하여 메일 발송 여부 확인
    List<String> requests = mockMailSender.getRequests();
    assertThat(requests.size(), is(2));
    assertThat(requests.get(0), is(users.get(1).getEmail()));
    assertThat(requests.get(1), is(users.get(3).getEmail()));
}

// 사용자의 레벨 업그레이드 여부를 확인하는 메소드
private void checkLevelUpgraded(User user, boolean upgraded) {
    User userUpdate = userDao.get(user.getId());
    ...
}

```

이 테스트는 다섯 단계의 작업으로 구성된다.

1. UserDao를 통해 가져올 테스트용 정보를 DB에 넣는다.
2. 메일 발송 여부를 확인하기 위해 MailSender 목 오브젝트를 DI 해준다.
3. 실제 테스트 대상인 userService의 메소드를 실행한다.
4. UserDao를 이용해 DB에서 데이터를 가져와 결과를 확인한다.
5. 목 오브젝트를 통해 UserService에 의한 메일 발송이 있었는지를 확인하면 된다.

UserDao 목 오브젝트

첫 번째와 네 번째의 테스트 방식도 목 오브젝트를 만들어서 적용해보겠다.

사용자 레벨 업그레이드 작업 중에 UserDao를 사용하는 코드

```java
 public void upgradeLevels() {
    List<User> users = userDao.getAll(); //업그레이드 후보 사용자 목록을 가져온다.
    for (User user : users) {
        if (canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}

protected void upgradeLevel(User user) {
    user.upgradeLevel();
    userDao.update(user); // 수정된 사용자 정보를 DB에 반영한다.
    sendUpgradeEmail(user);
}

```

여기서 userDao.update(user)의 호출은 리턴 값이 없다. 따라서 준비 필요 X

하지만 update 메소드의 사용은 핵심 로직인 전체 사용자 중에서 업그레이드 대상자는 레벨을 변경해준다에서 변경에 해당하는 부분을 검증할 수 있는 중요한 기능이다.

UserDao 타입의 테스트 대역이 필요하기 때문에 MockUserDao라고 하여 스태틱 내부 클래스로 만들도록 한다.

```java
import java.util.ArrayList;
import java.util.List;

public static class MockUserDao implements UserDao {
    private List<User> users; // 레벨 업그레이드 후보 User 오브젝트 목록
    private List<User> updated = new ArrayList<>(); // 업그레이드 대상 오브젝트를 저장해둘 목록

    private MockUserDao(List<User> users) {
        this.users = users;
    }

    // 업그레이드 대상 오브젝트 목록을 반환하는 메소드
    public List<User> getUpdated() {
        return this.updated;
    }

    // UserDao 인터페이스의 메소드 구현
    public List<User> getAll() {
        return this.users; // 모든 사용자 목록 반환
    }

    public void update(User user) {
        updated.add(user); // 업그레이드 대상 목록에 추가
    }

    // 아래의 메소드들은 기능을 제공하지 않고 예외를 던지는 것으로 구현되어 있음
    public void add(User user) {
        throw new UnsupportedOperationException();
    }

    public void deleteAll() {
        throw new UnsupportedOperationException();
    }

    public User get(String id) {
        throw new UnsupportedOperationException();
    }

    public int getCount() {
        throw new UnsupportedOperationException();
    }
}
```

MockUserDao는 UserDao 구현 클래스를 대신해야 하니 당연히 UserDao 인터페이스를 구현해야 한다.

인터페이스를 구현하려면 인터페이스 내의 모든 메소드를 만들어줘야 한다.

MockUserDao에는 두 개의 User 타입 리스트를 정의해둔다.

하나는 생성자를 통해 전달받은 사용자 목록을 저장해뒀다가, getAll() 메소드가 호출되면 DB에서 가져온 것처럼 돌려주는 용도다.

다른 하나는 update() 메소드를 실행하면서 넘겨준 업그레이드 대상 User 오브젝트를 저장해뒀다가 검증을 위해 돌려주기 위한 것이다.

이제 upgradeLevels() 테스트가 MockUserDao를 사용하도록 수정한다.

```java
List‹User> updated = mockUserDao.getUpdated();
```

테스트 하고 싶은 로직을 담은 클래스인 UserServiceImpldml 오브젝트를 직접 생성한다. 

고립된 테스트를 위한 준비가 끝났으면 테스트 대상인 UserServiceImpl 오브젝트의 메소드를 실행시킨다.

→ MockDaoUser가 의존 오브젝트로 DI 되어 있으므로 미리 준비해둔 사용자 목록을 받는다.

→ 로직에 따라 업그레이드 대상을 선정해서 레벨 변경 후 MockDaoUser의 update() 메소드를 호출하게 된다.

전체 개수를 확인했으면 순서에 따라서 업그레이드된 사용자의 아이디와 바뀐 레벨을 확인해보면 된다.

**테스트 수행 성능의 향상**

이제 수행시간을 비교하면 훨씬 빨라졌다.

모든 테스트를 고립으로 만든다면 1000개를 돌려도 1초도 걸리지 않는다.

### 6.2.3 단위 테스트와 통합 테스트

단위 테스트라는 용어를 명확하게 정의하고 해야만 한다.

이 책에서의 단위 테스트의 정의

⇒ upgradelLevels() 테스트처럼 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트 하는 것

반면 두 개 이상의 연동 DB나 파일등의 리소스가 참여하는 테스트는 통합 테스트라고 하겠다.

어떤 방법을 쓸 지 보는 가이드라인

- 항상 단위 테스트를 먼저 고려한다.
- 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다.
- 여러 개의 단위가 의존관계를 가지고 동작할 때를 위한 통합 테스트는 필요하다.

스프링이 지지하고 권장하는 깔끔하고 유연한 코드를 만들다보면 테스트도 그만큼 만들기 쉬워지고, 테스트는 다시 코드의 품질을 높여주고, 리펙토링과 개선에 대한 마음을 줄 것이다.

### 6.2.4 목 프레임워크

단위 테스트를 만들기 위해서는 목 오브젝트나 스텁의 사용이 필수적이다.

번거로운 목 오브젝트를 편리하게 작성하도록 도와주는 다양한 목 오브젝트 지원 프레임워크를 알아볼 것이다.

Mockito 프레임워크

그중에서도 Mockito를 사용해 바꿔주자.

```java
when (mockUserDao.getAll()). thenReturn(this.users);
```

mockUserDao의 getAll() 메소드가 호출되면 users리스트가 자동으로 리턴된다.

```java
verify (mockUserDao, times(2)) .update(any(User.class));
```

mockUserDao의 update() 메소드가 두 번 호출됐는 지 확인하는 코드이다.

```java
@Test
public void mockUpgradeLevels() throws Exception {
    UserServiceImpl userServiceImpl = new UserServiceImpl();

    UserDao mockUserDao = mock(UserDao.class);
    when(mockUserDao.getAll()).thenReturn(this.users);
    userServiceImpl.setUserDao(mockUserDao);

    MailSender mockMailSender = mock(MailSender.class);
    userServiceImpl.setMailSender(mockMailSender);

    userServiceImpl.upgradeLevels();

    verify(mockUserDao, times(2)).update(any(User.class));
    verify(mockUserDao).update(users.get(1));
    assertThat(users.get(1).getLevel(), is(Level.SILVER));
    verify(mockUserDao).update(users.get(3));
    assertThat(users.get(3).getLevel(), is(Level.GOLD));

    ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
    verify(mockMailSender, times(2)).send(mailMessageArg.capture());
    List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
    assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
    assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
}

```

Mockito를 사용해 목 오브젝트를 편하게 쓸 수 있었다.

하지만 익숙하게 사용할 수 있도록 학습해두자

## 6.3 다이내믹 프록시와 팩토리 빈

### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴

현재 트랜잭션 기능에는 추상화 작업을 통해 전략 패턴이 적용되어 있지만 아직 적용한다는 사실은 코드에 그대로 남아 있다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/bc263e5a-1521-4a90-9a6f-8a8052833752/Untitled.png)

부가기능 전부를 UserServiceTx를 만들며 UserServiceImpl에는 관련 코드가 하나도 남지 않게 되었다.

부가기능 외의 나머지 모든 기능은 원래 핵심기능을 가진 클래스로 위임해줘야 한다. 부가기능은 자신이 핵심 기능을 가진 클래스인 것처럼 꾸며서, 클라이언트가 자신을 거쳐서 핵심기능을 사용하도록 만들어줘야 한다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/2713c00b-9a63-4485-9c40-dbd92155db28/Untitled.png)

위 그림처럼 사용하게 된다.

이렇게 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리자, 대리인과 같은 역할을 한다고 해서 프록시라고 부른다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/138ddca8-06c3-44f1-a5d6-a95cf514d773/Untitled.png)

위 그림은 클라이언트가 프록시를 통해 타깃을 사용하는 구조를 보여주고 있다.

프록시의 사용 목적

1. 클라이언트가 타깃에 접근하는 방법을 제어하기 위해
2. 타깃에 부가적인 기능을 부여해주기 위해

→ 두 가지 모두 대리 오브젝트라는 개념의 프록시를 두고 사용한다는 점은 동일하지만, 목적에 따라서 디자인 패턴에서는 다른 패턴으로 구분한다.

**데코레이터 패턴**

타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴을 말한다.

데코레이터 패턴에서는 인터페이스를 구현한 타겟과 여러 개의 프록시를 사용할 수 있고, 프록시의 순서를 정해서 단계적으로 위임하는 구조로 만들면 된다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/1d8534e0-1a2d-4d4b-90f6-fcbd12d382b0/Untitled.png)

데코레이터의 다임 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메소드를 통해 위임 대상을 외부에서 런타임 시에 주입받을 수 있도록 만들어야 한다.

예시

자바 IO 패키지의 InputStream이나 OutputStream 구현 클래스는 데코레이터 패턴이 사용된 대표적인 예다.

데코헤이터 패턴은 타깃의 코드를 손대지 않고 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.

**프록시 패턴**

일반적으로 사용하는 프록시: 클라이언트와 사용 대상 사이에 대리 역할을 맡은 오브젝트를 두는 방법을 총칭한다.

디자인 패턴에서 말하는 프록시: 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우를 말한다.

프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다. 대신 클라이언트가 타깃에 접근하는 방식을 변경해준다.

타깃 오브젝트에 대한 레퍼런스가 필요할 경우 프록시 패턴을 적용해 타깃 오브젝트를 만드는 대신 프록시를 넘겨준다. 이후 프록시의 메소드를 통해 타깃을 사용하려고 시도하면 그때 프록시가 타깃 오브젝트를 생성하고 요청을 위임해주도록 한다.

다른 서버에 존재하는 오브젝트를 사용해야 한다면, 원격 오브젝트에 대한 프록시를 만들어두고, 클라이언트는 마치 로컬에 존재하는 오브젝트를 쓰는 것처럼 프록시를 사용하게 한다. 프록시는 클라이언트의 요청을 받으면 네트워크를 통해 원격의 오브젝트를 실행하고 결과를 받아서 클라이언트에게 돌려준다.

수정 가능한 오브젝트가 있는데 특정 레이어로 넘어가서는 읽기 전용으로 동작하게 해야한다면 오브젝트의 프록시를 만들어 사용하도록 한다.

프록시의 특정 메소드를 사용하려고 하면 접근이 불가능하다고 예외를 발생시키도록 한다.

⇒ 프록시 패턴은 타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시를 이용하는 것이다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/88fba12d-4036-40a3-b736-2a3464f22d6e/Untitled.png)

**프록시에 대한 책에서의 정의**

타깃과 동일한 인터페이스를 구현하고 클라이언트와 타깃 사이에 존재하면서 기능의 부가 또는 접근 제어를 담당하는 오브젝트를 모두 프록시라고 부를 것이다.

### 6.3.2 다이내믹 프록시

프록시를 만드는 과정은 매우 귀찮다

→ 자바에는 java.lang.reflect 패키지 안에 프록시를 손쉽게 만들 수 있도록 지원해주는 클래스들이 있다.

**프록시의 구성과 프록시 작성의 문제점**

프록시의 기능

- 타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
- 지정된 요청에 대해서는 부가기능을 수행한다.

UserServiceTx 프록시의 기능 구분

```java
public class UserServiceTx implements UserService {
    UserService userService; // 타깃 오브젝트
    ...

    public void add(User user) {
        this.userService.add(user); // 메소드 구현과 위임
    }

    // upgradeLevels 메소드 구현
    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            userService.upgradeLevels(); // 위임
            this.transactionManager.commit(status);
        } catch (RuntimeException e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```

프록시의 역할은 위임과 부가작업이라는 두 가지로 구분할 수 있다.

프록시를 만들기 번거로운 이유

1. 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기가 번거롭다는 점
2. 부가기능 코드가 중복될 가능성이 많다는 점

첫 번째 문제를 해결하기 위한 것이 JDK의 다이내믹 프록시이다.

**리플랙션**

리플랙션은 자바의 코드 자체를 추상화해서 접근하도록 만든 것이다.

클래스 오브젝트를 이용하면 클래스 코드에 대한 메타정보를 가져오거나 오브젝트를 조작할 수 있다.

Method를 이용해 메소드를 호출되는 방법

```java
package springbook.learningtest.jdk;

public class ReflectionTest {

    @Test
    public void invokeMethod() throws Exception {
        String name = "Spring";
        
        // length()
        assertThat(name.length(), is(6));
        Method lengthMethod = String.class.getMethod("length");
        assertThat((Integer)lengthMethod.invoke(name), is(6));
        
        // charAt()
        assertThat(name.charAt(0), is('S'));
        Method charAtMethod = String.class.getMethod("charAt", int.class);
        assertThat((Character)charAtMethod.invoke(name, 0), is('S'));
    }
}
```

String 클래스의 length() 메소드와 charAt() 메소드를 코드레엇 직접 호출하는 방법과 Method를 이용해 리플랙션 방식으로 호출되는 방법을 비교한 것이다.

**프록시 클래스**

다이내믹 프록시를 이용한 프록시 만들기

```java
interface Hello {
    String sayHello(String name);
    String sayHi(String name);
    String sayThankYou(String name);
}
```

```java
public class HelloTarget implements Hello {
    public String sayHello(String name) {
        return "Hello " + name;
    }

    public String sayHi(String name) {
        return "Hi " + name;
    }

    public String sayThankYou(String name) {
        return "Thank You " + name;
    }
}
```

이렇게 Hello 인터페이스를 통한 HelloTarget 오브젝트를 사용하는 클라이언트 역할을 하는 테스트를 만들었다.

```java
@Test
public void simpleProxy() {
    Hello hello = new HelloTarget(); // 타깃은 인터페이스를 통해 접근하는 습관을 들이자.
    assertThat(hello.sayHello("Toby"), is("Hello Toby"));
    assertThat(hello.sayHi("Toby"), is("Hi Toby"));
    assertThat(hello.sayThankYou("Toby"), is("Thank You Toby"));
}
```

이제 Hello 인터페이스를 구현한 프록시를 만든다.

리런하는 문자를 모두 대문자로 바꿔주는 기능을 추가한다.

HelloUppercase 프록시를 통해 문자를 바꿔준다. HelloUppercase 프록시는 Hello 인터페이스를 구현하고, Hello 타입의 타깃 오브젝트를 받아서 저장한다.

또한 타깃 오브젝트의 결과를 대문자로 바꿔주는 부가기능을 적용하고 리턴한다.

위임과 부가기능을 모두 처리하는 전형적인 프록시 클래스

```java
public class HelloUppercase implements Hello {
    Hello hello; // 위임할 타깃 오브젝트. 여기서는 타깃 클래스의 오브젝트인 것은 알지만 다른 프록시를 추가할 수도 있으므로 인터페이스로 접근한다.

    public HelloUppercase(Hello hello) {
        this.hello = hello;
    }

    public String sayHello(String name) {
        return hello.sayHello(name).toUpperCase(); // 위임과 부가기능 적용
    }

    public String sayHi(String name) {
        return hello.sayHi(name).toUpperCase();
    }

    public String sayThankYou(String name) {
        return hello.sayThankYou(name).toUpperCase();
    }
}

```

이 프록시의 문제점

1. 인터페이스의 모든 메소드를 구현해 위임하도록 코드를 만들어야 한다.
2. 부가기능인 리턴 값을 대문자로 바꾸는 기능이 모든 메서드에 중복돼서 나타난다.

**다이내믹 프록시 적용**

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/f8da8e8c-93ea-482a-849e-d3754fe73791/Untitled.png)

다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다.

다이내믹 프록시 오브젝트는 타깃의 인터페이스와 같은 타입으로 만들어진다. 이것을 사용함으로 인터페이스를 모두 구현해가면서 클래스를 정의하는 수고를 덜 수 있다.

다이내믹 프록시 오브젝트는 클라이언트의 모든 요청을 리플랙션 정보로 변환해서 InvocationHandler 구현 오브젝트의 invoke() 메소드로 넘기는 것이다. 타깃 인터페이스의 모든 메소드 요청이 하나의 메소드로 집중되기 때문에 중복되는 기능을 효과적으로 제공할 수 있다.

다이내믹 프록시 오브젝트와 InvocationHandler 오브젝트, 타깃 오브젝트 사이의 메소드 호출이 일어나는 과정

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/cfdaafa8-a231-495c-a1ef-1473e1c01ff3/Untitled.png)

HelloUppercase 클래스와 마찬가지로 모든 요청을 타깃에 위임하면서 리턴 값을 대문자로 바꿔주는 부가기능을 가진 InvocationHandler를 구현했다.

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class UppercaseHandler implements InvocationHandler {
    Hello target;

    public UppercaseHandler(Hello target) {
        this.target = target;
    }

    // 다이내믹 프록시로부터 전달받은 요청을 다시 타깃 오브젝트에 위임해야 하기 때문에 타깃 오브젝트를 주입받아 둔다.
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String ret = (String) method.invoke(target, args); // 부가기능 제공 후 타깃 오브젝트의 메소드 호출
        return ret.toUpperCase(); // 부가기능 적용 (대문자 변환)
    }
}
```

다이내믹 프록시로부터 요청을 전달받으려면 InvocationHandler를 구현해야 한다.

다이내믹 프록시를 통해 요청이 전달되면 리플랙션 API를 이용해 타깃 오브젝트의 메소드를 호출한다.

타깃 오브젝트의 메소들 호출이 끝났으면 프록시가 제공하려는 부가기능인 리턴 값을 대문자로 바꾸는 작업을 수행하고 결과를 리턴한다.

다이내믹 프록시를 생성하는 코드

```java
Hello proxiedHello = (Hello) Proxy.newProxyInstance(
        getClass().getClassLoader(), // 동적으로 생성되는 다이내믹 프록시 클래스의 로딩에 사용할 클래스 로더
        new Class[] {Hello.class}, // 구현할 인터페이스
        new UppercaseHandler(new HelloTarget()) // 부가기능과 위임 코드를 담은 InvocationHandler
);
```

첫 번째 파라미터는 클래스 로더를 제공해야 한다.

두 번째 파라미터는 다이내믹 프록시가 구현해야 할 인터페이스다.

⇒ 리플랙션 API를 적용하고, 복잡한 다이내믹 프록시 생성방법을 적용했는데도 원래 만들었던 HelloUppercase 프록시 클래스보다 장점이 뭐가 있는가?

**다이내믹 프록시의 확장**

1. 메소드가 늘어나도 다이내믹 프록시는 자동으로 포함된다.
2. Hello 타입의 타깃으로 제한할 필요가 없다.

InvacationHandler는 단일 메소드에서 모든 요청을 처리하기 떄문에 어떤 메소드에 어떤 기능을 적용할지를 선택하는 과정이 필요할 수도 있다.

### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능

트랜잭션 부가기능을 제공하는 다이내믹 프록시를 만들어 적용하는 방법이 효율적이다.

**트랜잭션 InvocationHandler**

요청을 위임할 타깃을 DI로 제공받는다.

타깃을 저장할 변수는 Object로 선언했다. 따라서 UserServiceImpl 외에 트랜잭션 적용이 필요한 어떤 타깃 오브젝트에도 적용할 수 있다.

UserServiceTx와 마찬가지로 트랜잭션 추상화 인터페이스인 PlatformTransactionManager를 DI 받도록 한다.

타깃 오브젝트의 모든 메소드에 무조건 트랜잭션이 적용되지 않도록 트랜잭션을 적용할 메소드 이름의 패턴을 DI 받는다.

트랜잭션을 적용하면서 타깃 오브젝트의 메소드를 호출하는 것은 UserServiceTx에서와 동일하다. 한 가지 차이점은 롤백을 적용하기 위한 예외는 RuntimeException 대신에 InvocationTargetException을 잡아야 한다는 차이가 있다.

**TransactionHandler와 다이내믹 프록시를 이용하는 테스트**

앞에서 만든 다이내믹 프록시에 사용되는 TransactionHandler가 UserService를 대신 할 수 있는지 확인하기 위해 UserServiceTest에 적용해보았습니다.

## 6.4 스프링의 프록시 팩토리 빈

앞서 나온 트랜잭션 부가기능을 스프링에서 제공하는 방법을 통해 해결할 것이다.

### 6.4.1 ProxyFactoryBean

스프링은 일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어를 제공

생성된 프록시는 스프링의 빈으로 등록되어야 함. 또한 프록시 오브젝트를 생성해주는 기술을 추상화한 팩토리 빈을 제공

ProxyFactoryBean은 빈 오브젝트로 등록하게 해주는 팩토리 빈이다.

→ 순수하게 프록시를 생성하는 작업만을 담당하고 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있게 한다.

MethodInterceptor 오브젝트는 타깃이 다른 여러 프록시에서 함께 사용할 수 있고 싱글톤 빈으로 등록이 가능하다.

**어드바이스: 타깃이 필요 없는 순수한 부가기능**

ProxyFactoryBean을 적용한 코드와의 차이점

1. InvocationHandler를 구현했을 때와 다르게 타깃 오브젝트를 사용하지 않는다.
2. MethodInterceptor로는 메소드 정보와 함께 타깃 오브젝트가 담긴 MethodInvocation 오브젝트가 전달된다.
3. MethodInvocation 구현 클래스는 공유 가능한 템플릿처럼 동작한다.
4. ProxyFactoryBean은 작은 단위의 템플릿/콜백 구조를 응용해서 적용했기 때문에 템플릿 역할을 하는 MethodInvocation을 싱글톤으로 두고 공유할 수 있는 가장 큰 강점을 가지고 있다.
5. 또한 addAdvice()라는 메소드를 사용하기 떄문에 ProxyFactoryBean 하나만으로 여러 개의 부가 기능을 제공해주는 프록시를 만들 수 있다. → 큰 장점
6. ProxyFactoryBean은 인터페이스 자동검출 기능을 사용해 타깃 오브젝트가 구현하고 있는 인터페이스 정보를 알아낸다.

 

**포인트컷: 부가기능 적용 대상 메소드 선정 방법**

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/703f6185-4d45-48bc-8ee8-2f9fc4ce77a1/Untitled.png)

타깃 변경과 메소드 선정 알고리즘 변경 같은 확장이 필요하면 팩토리 빈 내의 프록시 생성코드를 직접 변경해야 한다.

OCP 원칙을 깔끔하게 잘 지키지 못하는 어정쩡한 구조라고 볼 수 있다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/415f0ed5-27d9-495a-8054-eb26c0e6ed42/Untitled.png)

ProxyFactoryBean방식은 두 가지 확장 기능인 부가기능과 메소드 선정 알고리즘을 활용하는 유연한 구조를 제공한다.

어드바이스: 부가기능을 제공하는 오브젝트

포인트컷: 메소드 선정 알고리즘을 담은 오브젝트

이 두가지는 모두 프록시에 DI로 주입돼서 사용되다.

→ 여러 프록시에서 공유가 가능하도록 만들어지기 때문에 스프링의 싱글톤 빈으로 등록이 가능하다.

프록시로부터 어드바이스와 포인트컷을 독립시키고 DI를 사용하게 한 것은 전형적인 전략 패턴 구조다. 

```java
@Test
public void pointcutAdvisor() {
    // 프록시 팩토리 빈 생성
    ProxyFactoryBean pfBean = new ProxyFactoryBean();
    // 타겟 객체 설정
    pfBean.setTarget(new HelloTarget());

    // 메소드 이름을 비교해서 대상을 선정하는 포인트컷 생성
    NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
    // 이름 비교 조건 설정: "sayl*"로 시작하는 모든 메소드를 선택
    pointcut.setMappedName("sayl*");

    // 포인트컷과 어드바이스를 Advisor로 묶어서 추가
    pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));

    // 프록시 생성
    Hello proxiedHello = (Hello) pfBean.getObject();

    // 테스트
    assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
    assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
    assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You"));
}

```

포인트컷을 별개의 오브젝트로 묶어서 등록해야 하는 이유

→ ProxyFactoryBean에는 여러 개의 어드바이스와 포인트컷이 추가될 수 있기 때문이다.

1. 트랜잭션은 add로 시작하는 메소드에만 적용하지만, 보안 부가기능은 모든 메소드에 적용하고, 기능검사 부가기능은 get으로 시작하는 메소드에만 적용할 수가 있다.

→ 어드바이스와 포인터컷을 묶은 오브젝트를 인터페이스 이름을 따서 어드바이저라고 부른다.

어드바이저 = 포인트컷(메소드 선정 알고리즘) + 어드바이스(부가기능)

pointcutAdvisor() 테스트에서 NameMatchPointcut은 mappedName 프로퍼티 값을 이용해 메소드의 이름을 비교하는 방식으로 대상을 선정한다.

### 6.4.2 ProxyFactoryBean 적용

TxProxyFactoryBean을 이제 스프링이 제공하는 ProxyFactoryBean을 이용하도록 수정할 것이다.

**TransactionAdvice**

JDK 다이내믹 프록시 방식으로 만든 TransactionHandler의 코드에서 타깃과 메소드 선정 부분을 제거해주면 된다.

```java
package springbook.learningtest.jdk.proxy;

/**
 * 스프링의 어드바이스 인터페이스 구현
 */
public class TransactionAdvice implements MethodInterceptor {
    private PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    /**
     * 타깃을 호출하는 기능을 가진 콜백 오브젝트를 프록시로부터 받는다.
     * 덕분에 어드바이스는 특정 타깃에 의존하지 않고 재사용 가능하다.
     */
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            // 콜백을 호출해서 타깃의 메소드를 실행한다.
            // 타깃 메소드 호출 전후로 필요한 부가 기능을 넣을 수 있다.
            Object ret = invocation.proceed();

            // 경우에 따라서 타깃이 아예 호출되지 않게 하거나 재시도를 위한 반복적인 호출도 가능하다.
            transactionManager.commit(status);
            return ret;
        } catch (RuntimeException e) {
            // JDK 다이내믹 프록시가 제공하는 Method와는 달리 스프링의 MethodInvocation을 통한 타깃 호출은 예외가 포장되지 않고 타깃에서 보낸 그대로 전달된다.
            transactionManager.rollback(status);
            throw e;
        }
    }
}

```

타깃 메소드가 던지는 예외도 InvocationTargetException로 포장돼서 오는 것이 아니기 때문에 바로 처리하면 된다.

**스프링 XML 설정파일**

1. 트랜잭션 어드바이스 빈 설정
2. 포인트컷 빈 설정
3. 어드바이저 빈 설정
4. ProxyFactoryBean 설정

이 4가지의 설정파일을 변경해준다.

**테스트**

ProxyFactoryBean도 팩토리 빈이므로 기존의 TxProxyFactoryBean과 같은 방법으로 테스트할 수 있어 간단한 수정으로 해결이 가능하다.

```java
@Test
@DirtiesContext // 컨텍스트 설정을 변경하기 때문에 여전히 필요하다.
public void upgradeAllOrNothing() {
    TestUserService testUserService = new TestUserService(users.get(3).getId());
    testUserService.setUserDao(userDao);
    testUserService.setMailSender(mailSender);

    // userService 빈은 이제 스프링의 ProxyFactoryBean이다.
    ProxyFactoryBean txProxyFactoryBean = context.getBean("&userService", ProxyFactoryBean.class);
    txProxyFactoryBean.setTarget(testUserService);

    // FactoryBean 타입이므로 동일하게 getObject()로 프록시를 가져온다.
    UserService txUserService = (UserService) txProxyFactoryBean.getObject();

}
```

**어드바이스와 포인트컷의 재사용**

ProxyFactoryBean은 스프링의 DI와 템플릿/콜백 패턴, 서비스 추상화 등의 기법이 모두 적용된 것이다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/ff568ed5-56be-4515-8a64-f6dd7bf054be/Untitled.png)

## 6.5 스프링 AOP

DI의 응용 방식을 알아 볼 것이다.

### 6.5.1 자동 프록시 생성

프록시 팩토리 빈 방식의 접근 방법의 한계라고 생각한 두 가지 문제가 있었다.

1. 부가기능이 타깃 오브젝트마다 새로 만들어지는 문제는 스프링 ProxyFactoryBean의 어드바이스를 통해 해결된다.

부가기능의 적용이 필요한 타깃 오브젝트마다 거의 비슷한 내용의 ProxyFactoryBean 빈 설정정보를 추가해주는 부분을 수정할 것이다.

빈 후처리기를 이용한 자동 프록시 생성기

BeanPostProcessor 인터페이스를 구현해서 만드는 빈 후처리기다.

빈 후처리기는 이름 그대로 스프링 빈 오브젝트로 만들어지고 난 후에, 빈 오브젝트를 다시 가공할 수 있게 해준다.

스프링이 생성하는 빈 오브젝트의 일부를 프록시로 포장하고, 프록시를 빈으로 대신 등록할 수도 있다. 이것이 자동 프록시 생성 빈 후처리기다.

1. DefaultAdvisorAutoProxyCreator 후처리기에게 빈을 보낸다.
2. 빈으로 등록된 모든 어드바이저 내의 포인트컷을 이용해 전달받은 빈이 프록시 적용 대상인지 확인한다.
3. 프록시 적용 대상이면 그때는 내장된 프록시 생성기에게 현재 빈에 대한 프록시를 만들게 한다.
4. 만들어진 프록시에 어드바이저 연결해준다.
5. 프록시가 생성되면 원래 컨테이너가 전달해준 빈 오브젝트 대신 프록시 오브젝트를 컨테이너에게 돌려준다.
6. 컨테이너는 최종적으로 빈 후처기가 돌려준 오브젝트를 빈으로 등록하고 사용한다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/1cd50430-84d9-4056-9e6c-f60de47b015f/Untitled.png)

**확장된 포인트컷**

ProxyFactoryBean에서는 굳이 클래스 레벨의 필터는 필요 없었지만, 모든 빈에 대해 프록시 자동 적용 대상을 선별해야 하는 빈 후처리기인 DefaultAdvisorAutoProxyCreator는 클래스와 메소드 선정 알고리즘을 모두 갖고 있는 포인트컷이 필요하다.


### 6.5.2 DefaultAdvisorAutoProxyCreator의 적용

포인트컷 작성에 대해 실제로 적용할 것이다.

**클래스 필터를 적용한 포인트컷 작성**

메소드 이름을 비교하는 포인트컷인 NameMatchMethodPointcut을 상속해서 클래스 이름을 비교하는 ClassFilter를 추가하도록 할 것이다.

```java
package springbook.learningtest.jdk.proxy;

public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {
    public void setMappedClassName(String mappedClassName) {
        this.setClassFilter(new SimpleClassFilter(mappedClassName));
    }

    static class SimpleClassFilter implements ClassFilter {
        String mappedName;

        private SimpleClassFilter(String mappedName) {
            this.mappedName = mappedName;
        }

        public boolean matches(Class<?> clazz) {
            return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());
        }
    }
}
```

**어드바이저를 이용하는 자동 프록시 생성기 등록**

DefaultAdvisorAutoProxyCreator는 등록된 빈 중에서 Advisor 인터페이스를 구현한 것을 모두 찾는다.

이후 생성되는 모든 빈에 대해 어드바이저의 포인트컷을 적용하고 프록시 적용 대상을 선정한다.

선정 대상이라면 프록시를 만들어 원래 빈 오브젝트와 바꿔치기한다.

원래 빈 오브젝트는 프록시 뒤에 연결돼서 프록시를 통해서만 접근 가능하게 바뀐다.

따라서 타깃 빈에 의존한다고 정의한 빈들은 프록시 오브젝트를 대신 DI 받는다.

**포인트컷 등록**

새로 만든 클래스 필터 지원 포인트컷을 빈으로 등록한다.

**어드바이스와 어드바이저**

수정할 것은 없지만 DefaultAdvisorAutoProxyCreator에 의해 자동수집되고, 프록시 대상 선정 과정에 참여하며, 자동생성된 프록시에 다이내믹하게 DI돼서 동작하는 어드바이저가 된다.

**ProxyFactoryBean 제거와 서비스 빈의 원상복구**

userServiceImpl 빈의 아이디를 userService로 돌려놓을 수 있다.

→ 더 이상 명시적인 팩토리 빈을 등록하지 않기 떄문이다.

**자동 프록시 생성기를 사용하는 테스트**

@Autowired를 통해 컨텍스트에서 가져오는 UserService 타입 오브젝트는 UserServiceImpl 오브젝트가 아니라 트랜잭션이 적용된 프록시여야 한다.

이를 검증하기 위해서는

1. TestUserService 클래스를 직접 빈으로 등록한다.

여기에는 두 가지 문제가 있다.

1. TestUserService가 UserServiceTest 클래스의 내부에 정의된 스태틱 클래스이다.
2. 포인트컷이 트랜잭션 어드바이스를 적용해주는 대상 클래스의 이름 패턴이 *ServiceImpl이라고 되어 있어서 TestUserService 클래스는 빈으로 등록을 해도 포인트컷이 프록시 적용 대상으로 선정해주지 않는다는 것이다.

→ TestUserService 스태틱 멤버 클래스를 수정하여 해결해야 한다.

클래스 이름은 포인트컷이 선정해줄 수 있도록 TestUserServiceImpl로 변경한다.

예외를 발생시킬 대상인 네 번째 사용자 아이디를 클래스에 넣는다.

```java
public class TestUserServiceImpl extends UserServiceImpl {
    private String id = "madnite1";

    protected void upgradeLevel(User user) {
        if (user.getId().equals(this.id)) {
            throw new TestUserServiceException();
        }
        super.upgradeLevel(user);
    }
}
```

이후 TestUserServiceImpl을 빈으로 등록한다,

```java
<bean id="testUserService" 
	class="springbook.user.service.UserServiceTest$TestUserServiceImpl" 
	parent="userService">
    <!-- 프로퍼티 정의를 여기에 추가 -->
    <!-- userService 빈의 설정을 상속받음 -->
</bean>
```

특이점

1. $기호는 스태틱 멤버 클래스를 지정할 때 사용한다.
2. <bean> 태그에 parent 애트리뷰트를 사용하면 다른 빈 설정의 내용을 상속받을 수 있다.
3. upgradeAllOrNothing() 테스트를 새로 추가한 testUserService 빈을 사용하도록 수정한다.

**자동생성 프록시 확인**

두 가지를 확인하고 갈 것이다.

1. 트랜잭션이 필요한 빈에 트랜잭션 부가기능이 적용됐는가이다.
2. 아무 빈에나 트랜잭션의 부가기능이 적용된 것은 아닌지 확인해야 한다.

방법

1. 포인트컷 빈의 클래스 이름 패턴을 변경해서 이번엔 testUserService 빈에 트랜잭션이 적용되지 않게 한다.
2. 자동생성된 프록시를 확인한다.

→ 두 가지중 하나를 사용해 테스트한다.

### 6.5.3 포인트컷 표현식을 이용한 포인트컷

이제 좀 더 편리한 포인트컷 작성 방법을 알아볼 것이다.

스프링은 정규식이나 JSP의 EL과 비슷한 일종의 표현식 언어를 사용해서 포인트컷을 작성할 수 있도록 하는 방법이다.

이것을 포인트컷 표현식이라고 부른다.

**포인트컷 표현식**

포인트컷 표현식을 지원하는 포인트컷을 적용하려면 AspectJExpressionPointcut 클래스를 사용하면 된다.

이 클래스는 메소드의 선정 알고리즘을 포인트컷 표현식을 이용해 한 번에 지정할 수 있게 해준다.

이처럼 선정조건도 쉽게 만들어 낼 수 있는 강력한 표현식을 지원하는 데

→ AspectJ 포인트컷 표현식이라고 한다.

**포인트컷 테스트용 클래스**

```java
package springbook.learningtest.spring.pointcut;

public class Target implements TargetInterface {
    public void hello() {}
    public void hello(String a) {}
    public int minus(int a, int b) throws RuntimeException {
        return 0;
    }
    public int plus(int a, int b) {
        return 0;
    }
    public void method() {}
}
```

여러 개의 클래스 선정 기능을 확인하기 위해 한 개의 클래스를 더 준비한다.

```java
package springbook.learningtest.spring.pointcut;

public class Bean {
    public void method() throws RuntimeException {
    }
}
```

이제 두 개의 클래스와 총 6개의 메소드를 대상으로 포인트컷 표현식을 적용할 것이다.

**포인트컷 표현식 문법**

- public
    - 접근제한자다.
    - public, protected, private 등이 올 수 있다.
- int
    - 리턴 값의 타입을 나타내는 패턴이다.
    - 반드시 하나의 타입을 지정해야 한다.
- springbook.learningtest.spring.pointcut.Target
    - 생략 가능한 타입이며 생략하면 모든 타입을 허용하겠다는 뜻이다.
- minus
    - 메소드 이름 패턴이다.
    - 모든 메소드를 다 선택하겠다면 *을 넣는다.
- (int, int)
    - 메소드 파라미터의 타입 패턴이다.
- throws java.lang.RuntimeException
    - 예외 이름에 대한 타입 패턴이다. 생략 가능하다.

AspectJExpressionPointcut 클래스의 오브젝트를 만들고 포인트컷 표현식을 expression 프로퍼티에 넣어준다.

ezecution()은 메소드를 실행에 대한 포인트컷이라는 의미다.

**포인트컷 표현식 테스트**

표현식의 필수가 아닌 항목인 접근제한자 패턴, 클래스 타입 패턴, 예외 패턴은 생략할 수 있다.

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/0da01a99-5a0d-45ba-9b0f-4138668967c6/b3a703e6-70a5-480a-968f-9a9ad7dbd5f1/Untitled.png)

→ 결과를 미리 보지 말고 각 포인트컷의 적용될 메소드가 무엇인지 생각해보면서 진행하는 것이 좋다.

**포인트컷 표현식을 이용하는 포인트컷 적용**

AspectJ 포인트컷 표현식은 메소드를 선정하는 데 편리하게 쓸 수 있는 강력한 표현식 언어다.

메소드의 시그니처를 비교하는 방식인 execution() 외에도 몇 가지 표현식 스타일을 가지고 있는데 그중에서 bean()이 있다.

특정 애노테이션이 타입, 메소드, 파라미터에 적용되어 있는 것을 보고 메소드를 선정하게 하는 포인트컷도 만들 수 있다.

포인트컷 표현식의 장점과 단점

장점: 로직이 짧은 문자열에 담기기 때문에 클래스나 코드를 추가할 필요가 없어서 코드와 설정이 모두 단순해진다.

단점: 문자열로 된 표현식이므로 런타임 시점까지 문법의 검증이나 기능 확인이 되지 않는다

**타입 패턴과 클래스 이름 패턴**

포인트컷 표현식을 적용하기 전에는 클래스 이름의 패턴을 이용해 타킷 빈을 선정하는 포인트컷을 사용했다.

TestUserServiceImpl이라고 변경했던 테스트용 클래스의 이름을 다시 TestUserService라고 바꾼다. 그럼에도 테스트는 성공한다.

이유는 포인트컷 표현식의 클래스 이름에 적용되는 패턴은 클래스 이름 패턴이 아니라 타입 패턴이기 떄문이다.

즉 TestUserService 클래스로 정의된 빈은 UserServiceImpl 타입이기도 하고, 그 때문에 ServiceImpl로 끝나는 타입 패턴의 조건을 충족하는 것이다.

TargerInterface 인터페이스를 표현식에 사용했을 때 Target 클래스의 오브젝트가 포인트컷에 의해 선정된 것이다.

포인트컷 표현식의 타입 패턴 항복을 *..UserService라고 직접 인터페이스 이름을 명시해도 두 개의 빈이 모두 선정된다.

→ 포인트컷 표현식에서 타입 패턴이라고 명시된 부분은 모두 동일한 원리가 적용된다.
