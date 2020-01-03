---
title: 查询结果处理
group: base
type: guide
order: 4
---

POJO是一个简单的Java对象，主要是包含一些属性和 `getter/setter` 方法，在业务中常用到的是用于传输数据以及作为参数传递。 在Web应用的场景中，也通常用来和前端做数据交互。

jOOQ的代码生成器能够帮我们根据表结构生成对应的POJO，能很大程度上减少我们自己创建POJO的工作量，当然，此功能也是大部分ORM框架的必备功能。本章主要讲解各种方式将数据结果转换为我们想要的格式。

## 读取接口
查询操作通常以fetch API 作为结束API，例如常用的有 
- 基础读取
  - `fetch`         读取集合
  - `fetchSet`      读取并返回一个Set集合
  - `fetchArray`    读取并返回一个数组
  - `fetchOne`      读取单条记录，如果记录超过一条会报错
  - `fetchAny`      读取单条记录，如果有多条，会取第一条数据
  - `fetchSingle`   读取单条记录，如果记录为空或者记录超过一条会报错

- 扩展
  - `fetchMap`      读取并返回一个Map
  - `fetchGroups`   读取并返回一个分组Map

### `fetch`, `fetchOne`
基础读取的方法，这里用 `fetch` 和 `fetchOne` 方法做演示。其他几个方法的使用方式都类似，掌握一个其他的都可以很快掌握。

如果 `fetchOne` 方法的结果集查询超出一条，会抛出异常 `org.jooq.exception.TooManyRowsException`

 `fetchSet, fetchArray`  方法和 `fetch` 方法一样，都是返回多条数据，只是返回的格式不同，fetch通常返回List或者jOOQ的Result对象

接下来介绍一下几个方法重载的返回值

- `fetch()`, `fetchOne()` 
    此方法无参调用时，直接返回一个通用`Record`对象，如果是`fetch`方法，返回的是一个`Result<Record>`结果集对象
```java
    Record record = dslContext.select().from(S1_USER)
            .where(S1_USER.ID.eq(1)).fetchOne();
    Result<Record> records = dslContext.select().from(S1_USER).fetch();
```

- `fetch(RecordMapper mapper)`, `fetchOne(RecordMapper mapper)`
    `RecordMapper`接口的提供`map`方法，用于来返回数据。`map` 方法传入一个 `Record` 对象。常见的用法是使用lambda表达式将 `Record` 对象转换成一个POJO，进行多表查询的时候，如果多个表的字段查询，并且名称一样的时候，会以最后一个字段值为准。
```java
S1UserPojo userPojo = dslContext.select()
            .from(S1_USER)
            .where(S1_USER.ID.eq(1))
            .fetchOne(r -> r.into(S1UserPojo.class));
List<S1UserPojo> userPojoList = dslContext.select()
            .from(S1_USER)
            .where(S1_USER.ID.eq(1))
            .fetch(r -> r.into(S1UserPojo.class));

```
    多表查询时，字段系统时，直接用into方法将结果集转换为POJO时，相同字段名称的方法会以最后一个字段值为准。这时候，我们可以现将结果集通过 `into(Table table)` 方法将结果集转换为指定表的`Record`对象，然后再`into`进指定的POJO类中。
```java 
// 多表关联查询，查询s2_user_message.id = 2的数据，直接into的结果getId()却是1
// 这是因为同时关联查询了s1_user表，该表的id字段值为1
S2UserMessage userMessage = dslContext.select().from(S2_USER_MESSAGE)
        .leftJoin(S1_USER).on(S1_USER.ID.eq(S2_USER_MESSAGE.USER_ID))
        .where(S2_USER_MESSAGE.ID.eq(2))
        .fetchOne(r -> r.into(S2UserMessage.class));
// userMessage.getId() == 1

// 将结果集into进指定的表描述中，然后在into至指定的POJO类
S2UserMessage userMessage2 = dslContext.select().from(S2_USER_MESSAGE)
        .leftJoin(S1_USER).on(S1_USER.ID.eq(S2_USER_MESSAGE.USER_ID))
        .where(S2_USER_MESSAGE.ID.eq(2))
        .fetchOne(r -> {
            S2UserMessage fetchUserMessage = r.into(S2_USER_MESSAGE).into(S2UserMessage.class);
            fetchUserMessage.setUsername(r.get(S1_USER.USERNAME));
            return fetchUserMessage;
        });
// userMessage.getId() == 2
```

- `fetch(Field field)`, `fetchOne(Field field)`
    `Field`是一个接口，代码生成器生成的表字段常量例如 `S1_USER.ID`, 都实现了 `Field` 接口，这个重载可以直接取出指定表字段，会自动根据传入的字段对象推测字段类型
```java
Integer id = dslContext.select().from(S1_USER).where(S1_USER.ID.eq(1))
        .fetchOne(S1_USER.ID);

List<Integer> id = dslContext.select().from(S1_USER).where(S1_USER.ID.eq(1))
        .fetch(S1_USER.ID);
```

- `fetch(String fieldName, Class<?> type)`, `fetchOne(String fieldName, Class<?> type)`
    可以直接通过字段名称字符串获取指定字段值，可以通过第二个参数指定返回值，如果不指定，返回Object
```java
Integer id = dslContext.select().from(S1_USER).where(S1_USER.ID.eq(1))
        .fetchOne("id", Integer.class);

List<Integer> idList = dslContext.select().from(S1_USER).where(S1_USER.ID.eq(1))
        .fetch("id", Integer.class); 
```

- `fetch(int fieldIndex, Class<?> type)`, `fetchOne(int fieldIndex, Class<?> type)`
    可以通过查询字段下标顺序进行查询指定字段，可以通过第二个参数指定返回值，如果不指定，返回Object
```java
Object id = dslContext.select(S1_USER.ID, S1_USER.USERNAME).from(S1_USER)
        .where(S1_USER.ID.eq(1)).fetchOne(0);

List<Object> idList = dslContext.select(S1_USER.ID, S1_USER.USERNAME)
        .from(S1_USER).where(S1_USER.ID.eq(1)).fetch(0);
```

### fetchAny
此方法大体用法和 fetchOne 
-- TODO 

### fetchSingle

### fetchArray

### fetchSet 
-- TODO


### `fetchMap`

此方法可以将结果集处理为一个Map格式，此方法有很多重载，这里介绍几个常用的，注意，此方法作为Key的字段必须确定是在当前结果集中是唯一的，如果出现重复Key，此方法会抛出异常

- `fetchMap(Field<K> field, Class<V> type)`
以表字段值为key，返回一个 `K:V` 的Map对象
```java
Map<Integer, S1UserPojo> idUserPojoMap = dslContext.select().from(S1_USER)
                .fetchMap(S1_USER.ID, S1UserPojo.class);

```

- `fetchMap(Feild<K> field, Field<V> field)`
以表字段值为key，返回一个 `K:V` 的Map对象
```java
Map<Integer, String> idUserNameMap = dslContext.select().from(S1_USER)
                .fetchMap(S1_USER.ID, S1_USER.USERNAME);
```

### `fetchGroups`

此方法可以将结果集处理为一个Map格式，和`fetchMap`类似，只不过这里的值为一个指定类型的集合，通常在处理一对多数据时会用到

- `fetchGroups(Field<K> field, Class<V> type)`
以表字段值为Key，返回一个`K:List<V>` 的Map对象
```java
Map<Integer, List<S2UserMessage>> userIdUserMessageMap = dslContext.select().from(S2_USER_MESSAGE)
                .fetchGroups(S2_USER_MESSAGE.USER_ID, S2UserMessage.class);
```

- `fetchGroups(Field<K> keyField, Field<V> valueField)`
- 以表字段值为Key，返回一个`K:List<V>` 的Map对象
```java
Map<Integer, List<Integer>> userIdUserMessageIdMap = dslContext.select().from(S2_USER_MESSAGE)
                .fetchGroups(S2_USER_MESSAGE.USER_ID, S2_USER_MESSAGE.ID);
```

## 内容总结
本章源码: [https://github.com/k55k32/learn-jooq/tree/master/section-3](https://github.com/k55k32/learn-jooq/tree/master/section-3)

文章主要讲解的是各种类型的读取API，掌握好这些API对于jOOQ的使用很有帮助。 本章要特别注意以下几个接口
- `Field<T>` 接口
    由代码生成器生成的所有表字段常量，都是此接口的实现类，包含了字段信息和字段类型，所有的读取接口基本都有基于此字段参数的重载

- `RecordMapper<? super R, E> mapper` 接口
    该接口很简单，主要是提供一个 `map` 方法给大家自由实现，因为此接口只有一个方法，所以可以通过lambda表达式快速实现该接口。同样的，所有的读取接口基本都有基于此接口参数的重载
```java
public interface RecordMapper<R extends Record, E> {
    E map(R record);
}
```