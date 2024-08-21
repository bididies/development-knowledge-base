#transactional
### Основные определения
> Транзакция – группа последовательных операций с DB, которая является логической единицей работы с данными (т.е. эти последовательные операции не делимы, и если одна из них не выполняется, то все остальные должны откатиться).

Транзакция может быть выполнена либо целиком и успешно, либо не выполнена вообще.

### Материалы
[Видео с хорошей теорией и примерами](https://www.youtube.com/watch?v=ovas9OCVfqo&t=1790s)
[Статья про уровни изоляции](https://habr.com/ru/companies/maxilect/articles/785960/)

### Тезисы
- В JDBC отказываются только операции добавления и изменения данных. (`ALTER TABLE`, `CREATE TABLE` и т.п. не откатываются).
- `auto-commit` - изменения комитятся сразу после каждого выполненного запроса.
- Можно ли повесить `@Transactional` на приватный метод? (нет)
  @Transactional – в Spring это proxy, которая оборачивает класс, а конкретно, вызов помеченного аннотацией метода.
  
  Т.о. на приватный метод мы не можем повесить @Transactional, т.к. приватный метод в proxy (в другом классе) не вызвать. Proxy в этом случае открывает транзакцию перед вызовом оборачиваемого метода в proxy и закрывает после. 

- Если вызвать метод, помеченный @Transactional, в том же классе, что и сам метод, то @Transactional не сработает. (т.к. этот метод будет вызван на `this` в самом классе, а не в прокси классе)
- @Transactional не откатит транзакцию из-за Exception, только из-за RuntimeException.
- Помечать @Transactional нужно только public методы.
- @Transactional занимает соединение с DB, поэтому нужно выносить долгую логику за пределы метода, помеченного @Transactional, т.к. время соединения с DB из-за этого увеличится.

### ACID
#### Atomicity (Атомарность)
Любая транзакция не может быть частично завершена — она либо выполнена, либо нет.

#### Consistency (Согласованность)
Согласованность гарантирует, что транзакция может перевести базу данных только из одного допустимого состояния в другое. Т.е. если деньги списались с одного счета, то они либо поступят на другой счет, либо вернутся обратно.

Согласованность данных достигается за счет атомарности.

#### Isolation (Изолированность)
Когда транзакция выполняется, параллельные транзакции не должны оказывать влияния на ее результат.

Изолированность — дорогое требование, поэтому в реальных БД существуют режимы, которые изолируют транзакцию не полностью.

Проблемы, возникающие при параллельном выполнении транзакций над одними и теме же данными:
![[Pasted image 20240815232735.png | 300]]
![[Pasted image 20240815232915.png | 300]]

>  Кейс: строится отчет на основе одних и тех же данных. И в процессе построения, данные изменяются.
>  Проблема: отчет построен на основе разных данных.

![[Pasted image 20240815233109.png | 300]]

#### Durability (Устойчивость)
Гарантируется, что после того, как транзакция была зафиксирована, она останется зафиксированной даже в случае сбоя системы (например, отключения питания или сбоя).

### Propagation Levels
- Физические транзакции: это фактические транзакции **JDBC**.
- Логические транзакции: это (потенциально вложенные) аннотированные `@Transactional` методы **Spring**.

> Параметр `propagation` отвечает за стратегию распространения транзакций.
> Это свойство определяет, что происходит, когда внутри транзакции вызывается другой метод, также помеченный как `@Transactional`.

В Spring существует семь стратегий распространения, которые определяют, будет ли создана новая транзакция, будет ли использована текущая транзакция или возможно вообще не будет транзакции.

1. **REQUIRED**
   > Это дефолтное значение
   
   Spring проверяет наличие активной транзакции, и если ничего не существует, создает новую. В противном случае бизнес-логика добавляется к текущей активной транзакции
   
2. **SUPPORTS**
> `@Transactional(propagation = Propagation.SUPPORTS)`

Если транзакция существует, то будет использоваться существующая транзакция.
Если транзакции нет, то код будет выполнен в режиме автокоммита, то есть под каждый запрос (каждый вызов метода репозитория) будет создаваться новая транзакция.

3. **MANDATORY**
Если есть активная **внешняя** транзакция, то она будет использоваться. Если активной **внешней** транзакции нет - выбросится исключение.

4. **NEVER**
   Новая транзакция не должна быть запущена внутри **внешней** транзакции.
   Выбросится исключение при наличии активной **внешней** транзакции.
``` java
class Service1 {
	@Autowired
	private final Service2 service2;

	// внешняя транзакция
	@Transactional
	public void method1() {
		service2.method2();
	}

}

class Service2 {
	// выбросится исключение из-за наличия внешней транзакции
	@Transactional(propagation = Propagation.NEVER)
	public void method2() {...}
}
```
   
5. **NOT_SUPPORTED**
   Работает всегда в режиме автокоммита, вне зависимости от того, есть транзакция уже или нет.
   Если транзакция есть -  она будет приостановлена до завершения новой транзакции. Если транзакции и так нет - то что надо.
   
6. **REQUIRES_NEW**
   Создается новая транзакция в любом случае.
   Если текущая транзакция существует, она будет приостановлена до завершения новой транзакции.
``` java
class Service1 {
	@Autowired
	private final Service2 service2;

	@Transactional
	public void method1() {
		service2.method2();
	}

}

class Service2 {
	@Transactional(propagation = Propagation.REQUIRES_NEW)
	public void method2() {...}
}
```

- для `method1()` создается транзакция (внешняя);
- в момент вызова `service2.method2()` внешняя транзакция останавливается и создается еще одна транзакция (внутренняя);
- `method2()` выполняется, внутренняя транзакция комитится и передается управление внешней транзакции.

Если во внутренней транзакции выбрасывается `unchecked Exception`, то внешняя транзакция тоже откатится, т.к. `unchecked Exception` пробросится выше.
   
7. **NESTED**
В этом случае будет создана вложенная транзакция. Если текущая транзакция существует, новая транзакция будет выполняться внутри текущей. Если же текущей транзакции нет, будет создана новая транзакция.

Если внутренняя транзакция падает с исключением, то внешняя не откатывается полность, а откатывается до точки сохранения (save point).
- `JPA` и `Hibernate` не поддерживают `NESTED`.

### Isolation
> Изоляция транзакций — это способ управления тем, как транзакции влияют друг на друга. Это свойство решает проблемы, возникающие при одновременном доступе к общим данным.
> 
> Уровень изолированности - это условный уровень консистентности данных, который может быть достигнут при выполнении параллельных транзакций. Грубо говоря, это те ограничения, на которые мы готовы пойти, выполняя параллельные запросы в базу, чтобы сохранить целостность данных.

Есть 4 основные проблемы:
- **lost updates** (потерянные данные)
  когда выполняются две параллельные транзакции и одна перетирает изменения, которые внесла другая транзакция.
- **dirty read** (грязное чтение)
  когда первая транзакция читает не закомиченные изменения второй и использует их. Но проблема заключается в том, что вторая транзакция может быть откачена.
- **non-repeatable reads**
  Когда в рамках одной транзакции делается 2 одинаковых запроса.
  Но между этими запросами, данные были изменены другой транзакцией.
  И получается что во втором запросе выбираются те же самые строки, но с измененными данными.
  `Полагаю, delete сюда же относится, т.к. является разновидностью update`.
- **phantom read**
  Так же выполняется два одинаковых запроса, в рамках одной транзакции.
  Но между запросами добавляются новые строки.
  И второй запрос уже выбирает большее кол-во строк.

> **Уровень изоляции определяет, какие из этих проблем допустимы.**
> В Spring доступно пять уровней изоляции, и каждый из них решает определенные проблемы. (пятый - `DEFAULT` - использует уровень изоляции, установленный в БД по умолчанию)

#### Уровни изоляции
![[Pasted image 20240816150417.png]]

**READ_UNCOMMITTED**
Читаются все внесенные изменения в БД.

**READ_COMMITTED**
Читаются только закомиченные изменения.

**REPEATABLE_READ**
Славится лок на строки, с которыми оперировал запрос при первом селекте в рамках транзакции.
Но это не относится к строкам, которые были добавлены, т.к. при первом селекте их еще не было и на них не ставился лок.

**SERIALIZABLE**
Выполняет транзакции последовательно.