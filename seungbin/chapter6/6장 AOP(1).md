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
