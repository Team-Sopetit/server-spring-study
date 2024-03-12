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
