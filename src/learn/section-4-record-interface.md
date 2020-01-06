---
title: Record 详解
group: base
type: guide
order: 5
---

在之前的章节中，基本每章都有出现的 `Record` 接口，也做过一些简单的讲解和一些代码示例。作为jOOQ中最重要的一个接口，本章将详细讲解 `Record` 的各种API和基础用法。

## 介绍
`Record` 是jOOQ定义的用于储存数据库结果记录的一个接口，其主要是将一个表字段的列表和值的列表使用相同的顺序储存在一起，可以看做是一个用于储存列/值的映射的对象。

`Record` 通常有以下几种形式

### 表记录

来自具体的数据库表，包含主键。在进行查询操作的时候，jOOQ会将结果集包装为一个`TableRecord` 对象。 在使用代码生成器的时候，会生成更详细的表记录类，包含表的每个字段操作等，通常以表名为开头 `XxxxRecord`。

### UDT 记录   
    
通常用于 Oracle 等支持用户自定义数据类型的数据库记录，这里接触较少，不作讲解

### 明确数据的记录   

通用记录类型的一种，当你的字段不超过22个时，会根据字段个数反射成 `Record1`, `Record2` ... `Record22` 类的对象。这些对象需要的泛型个数和后面的数字一致，类型和字段类型一致。jOOQ自动生成的`Record`对象里，如果字段个数不超过 22 个，会同时实现 `Record[N]` 接口

举个例子

- `s1_user` 这张表有6个字段，其生成的 `Record` 对象，继承了 `UpdatableRecordImpl` 并且实现了 `Record6` 接口。`Record6`接口的泛型参数和表字段类型一一对应，类型顺序和数据库内字段顺序一致
```Java
class S1UserRecord extends UpdatableRecordImpl<S1UserRecord> 
    implements Record6<Integer, String, String, String, Timestamp, Timestamp>
```

- `s4_columen_gt22` 这张表来用于演示，该表的字段因为超过了22个，所以不会去实现`Record[N]` 接口，只继承了`UpdatableRecordImpl`
```java
class S4ColumenGt22Record extends UpdatableRecordImpl<S4ColumenGt22Record>
```

再看看 `Record[N]` 的接口定义，此接口主要是提供了获取字段，获取值，设置值的方法。由泛型决定字段/值类型和顺序，N 决定字段/值的个数。此接口的目的其实也很简单，就是为了更快速的操作指定位置的字段/值。
```java
interface Record[N]<T1, ... T[N]> {
    // 获取字段
    Field<T1> field1();
    ...
    Field<T[N]> field[N]();

    // 获取值
    T1 value1();
    ...
    T[N] valueN();

    // 设置值
    Record[1]<T1> value1(T1 value)
    ... 
    Record[N]<TN> value1(T[N] value)
}
```

## 创建 Record 对象
这里说的创建`Record`对象，指的是在jOOQ已经生成了对应表的`Record`类的情况下，创建Record进行。使用 `Record` 提供的方法，我们可以方便的做一些针对表数据的操作。创建对象的方式有以下几种

### `new`
由jOOQ生成的`Record`对象，可以通过 `new` 的方式直接创建一个实例。 通过直接 `new` 的方式创建对象，由于没有连接相关信息，无法直接进行 `insert`, `update`, `delete` 方法的调用。但是可以通过 `DSLContext` 的API进行操作数据操作，通过这种方式创建的`Record`对象可以理解为一个单纯的数据储存对象
```java
S1UserRecord s1UserRecord = new S1UserRecord();
s1UserRecord.setUsername("new1");
s1UserRecord.setEmail("diamondfsd@gmail.com");
// insert会报错，因为没有连接配置
s1UserRecord.insert();

// 不需要获取主键，直接执行insert
int row = dslContext.insertInto(S1_USER).set(s1UserRecord)
        .execute()

// 执行insert并返回相应字段
Integer id = dslContext.insertInto(S1_USER).set(s1UserRecord)
                .returning(S1_USER.ID)
                .fetchOne().getId();
```

### `newRecord`
在获取到 `DSLContext` 实例后，可以使用 `dslContext.newRecord(Table<?> table)` 方法来创建一个指定表的`Record`对象，这也是比较常用的方法。通过此方法创建的对象包含了`dslContext`内的数据连接配置，可以直接进行 `insert`, `update`, `delete` 等操作。
```java
S1UserRecord s1UserRecord = dslContext.newRecord(S1_USER);
s1UserRecord.setUsername("newRecord1");
s1UserRecord.setEmail("diamondfsd@gmail.com");
s1UserRecord.insert();
```

### `fetch`
通过`fetch*`方法读取到结果 `Record` 对象，同样带有数据库连接相关配置，并且带有查询结果的数据。可以直接进行数据操作
```java
S1UserRecord s1UserRecord = dslContext.selectFrom(S1_USER).where(S1_USER.ID.eq(1))
        .fetchOne();
s1UserRecord.setEmail("hello email");
int row = s1UserRecord.update();

//

S1UserRecord fetchIntoUserRecord = dslContext.select().from(S1_USER)
        .where(S1_USER.ID.eq(1))
        .fetchOneInto(S1UserRecord.class);
fetchIntoUserRecord.setEmail("hello email2");
int row2 = fetchIntoUserRecord.update();
```