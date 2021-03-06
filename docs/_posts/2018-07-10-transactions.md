---
layout: post
title: Spring - Transaction Management
---
# Transaction Management

**Why use Transactions?** _To Enforce the **ACID** Principles_

- **A**tomic - Each unit of work is an all-or-nothing operation
- **C**onsistent - Database integrity constraints are never violated
- **I**solated - Isolating transactions from each other
- **D**urable - Committed changes are permanent

- **Local Transactions** – Single Resource - Transactions managed by underlying resource
- **Global** (distributed) **Transactions** – Multiple Resources - Transaction managed by separate, dedicated transaction manager

Spring uses the same API for global vs. local.

`PlatformTransactionManager` abstraction hides implementation details.

Spring’s `PlatformTransactionManager` is the base interface for the abstraction, possible implementations:

- `DataSourceTransactionManager`
- `JmsTransactionManager`
- `JpaTransactionManager`
- `JtaTransactionManager`
- `WebLogicJtaTransactionManager`
- `WebSphereUowTransactionManager`
- and more...

Turning on by annotatiooin on configuration class `@EnableTransactionManagement`

```java
@Configuration
@EnableTransactionManagement
public class TxnConfig {
    @Bean
    // bean name is 'transactionManager' important for Spring internals
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

Adding transactional behaviour by `@Transactional` annotation (Spring package).
Spring also support `javax.transaction.Transactional` which provides fewer options (propagation, `rollbackOn`, `dontRollbackOn`)

```java
public class RewardNetworkImpl implements RewardNetwork { 
    @Transactional
    public RewardConfirmation rewardAccountFor(Dining d) {
    // atomic unit-of-work
    }
}

// interface annotation also considered
@Transactional
public class RewardNetworkImpl implements RewardNetwork {
    public RewardConfirmation rewardAccountFor(Dining d) {
        // atomic unit-of-work
    }

    @Transactional(timeout=45)
    // only for override default/class settings
    public RewardConfirmation updateConfirmation(RewardConfirmantion rc) {
        // atomic unit-of-work
    }
}
```

- Target object wrapped in a proxy. Uses an Around advice
- Proxy implements the following behavior
  - Transaction started before entering the method
  - Commit at the end of the method
  - Rollback by default if method throws a RuntimeException (can be overridden)
- Transaction context bound to current thread. 
- All controlled by configuration

```java
public class RewardNetworkImpl implements RewardNetwork { @Transactional
    public RewardConfirmation rewardAccountFor(Dining d) {

        // exception triggers rolback
        throw new RuntimeException();
    }

    // checked exceptions needs to be explicite given
    // safe (no rollback) exceptions can be also provided
    @Transactional(rollbackFor=MyCheckedException.class, noRollbackFor={JmxException.class, MailException.class})
    public RewardConfirmation rewardAccountFor(Dining d) throws Exception {
        // ...
    }
}
```

## Transaction isolation

4 isolation levels can be used:

- `READ_UNCOMMITTED`
  - Lowest isolation level – allows dirty reads
  - Current transaction can see the results of another uncommitted unit-of-work
  - Typically used for large, intrusive read-only transactions And/or where the data is constantly changing
  - `@Transactional (isolation=Isolation.READ_UNCOMMITTED)`
- `READ_COMMITTED`
  - Does not allow dirty reads only committed information can be accessed
  - Default strategy for most databases
  - `@Transactional (isolation=Isolation.READ_COMMITTED)`
- `REPEATABLE_READ`
  - Does not allow dirty reads
  - Non-repeatable reads are prevented. If a row is read twice in the same transaction, result will always be the same
  - Might result in locking depending on the DBMS
  - `@Transactional (isolation=Isolation.REPEATABLE_READ)`
- `SERIALIZABLE`
  - Prevents non-repeatable reads and dirty-reads
  - Prevents phantom reads
  - `@Transactional (isolation=Isolation.SERIALIZABLE)`

## Transaction propagation

```java
public class ClientServiceImpl implements ClientService {
    @Autowired
    private AccountService accountService;

    @Transactional
    public void updateClient(Client c)
    { 
    // …
        this.accountService.update(c.getAccounts());
    }
}

public class AccountServiceImpl implements AccountService
{
    @Transactional
    public void update(List <Account> l)
    {
        // …
    }
}
```

`@Transactional(propagation=Propagation.REQUIRES_NEW)`

Propagation Type | If NO current transaction (txn) exists | If there IS a current transaction (txn)
--- | --- | ---
MANDATORY | Throw exception | Use current txn
NEVER | Don't create a txn, run method without a txn | Throw exception
NOT_SUPPORTED | Don't create a txn, run method without a txn | Suspend current txn, run method without a txn
SUPPORTS | Don't create a txn, run method without a txn | Use current txn
REQUIRED (default) | Create a new txn | Use current txn
REQUIRES_NEW | Create a new txn | Suspend current txn, create a new independent txn
NESTED | Create a new txn | Create a new nested txn

## Programmatic Transactions  - `TransactionTemplate`

```java
public interface TransactionCallback<T> {
    public T doInTransaction(TransactionStatus status) throws Exception;
}
```

```java
// no @Transactional
public RewardConfirmation rewardAccountFor(Dining dining) {
    // ...
    return new TransactionTemplate(txManager).execute( (status) -> {
        try {
            // ...
            accountRepository.updateBeneficiaries(account);
            confirmation = rewardRepository.confirmReward(contribution, dining);
        }
        catch (RewardException e) {
            // manual rollback request 
            status.setRollbackOnly();
            confirmation = new RewardFailure();
        }
        return confirmation;
        }
    );
}
```

## Read only transactions

```java
public void rewardAccount1() {
    // two connections ???
    jdbcTemplate.queryForList("...");
    jdbcTemplate.queryForInt("...");
}

@Transactional(readOnly=true)
public void rewardAccount2() {
    // single connection
    jdbcTemplate.queryForList("...");
    jdbcTemplate.queryForInt("...");
}

// With a high isolation level, a read-only transaction prevents
// data from being modified until the transaction commits
@Transactional(readOnly=true, isolation=Isolation.REPEATABLE_READ)
public void myAccounts(long userId) {
    List accounts = jdbcTemplate.queryForList ("SELECT * FROM Accounts WHERE user = ?", userId);
    process(accounts);
    int nAccounts = jdbcTemplate.queryForInt ("SELECT count(*) FROM Accounts WHERE user = ?", userId);
    assert accounts.size() == nAccounts;
}
```

## Multiple transaction managers

**Important**: Separate transaction managers = separate transactions!

```java
@Bean
public PlatformTransactionManager myOtherTransactionManager() {
    return new DataSourceTransactionManager(dataSource1());
}

@Bean
@Primary
public PlatformTransactionManager transactionManager() {
    return new DataSourceTransactionManager(dataSource2());
}


@Transactional("myOtherTransactionManager")
public void rewardAccount1() {
    jdbcTemplate.queryForList("...");
    jdbcTemplate.queryForInt("...");
}

// uses primary manager
@Transactional
public void rewardAccount2() {
    jdbcTemplate.queryForList("...");
    jdbcTemplate.queryForInt("...");
}
```

## Global transactions

- Also called distributed transactions
- Involve multiple dissimilar resources
- Global transactions typically require JTA and specific drivers (XA drivers)
  - Two-phase commit protocol
- Many possible strategies
  - Spring allows you to switch easily from a non-JTA to a JTA transaction policy
  - Just change the type of the transaction manager
