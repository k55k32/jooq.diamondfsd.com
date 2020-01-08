---
title: 性能优化和DAO的生成
group: base
type: guide
order: 6
---

大部分ORM框架在将数据映射成对象时，都是通过反射的方式进行的

jOOQ对象转换主要是`Record API`来完成，最常用的 `from`/`into` 方法也是使用反射方式来进行类型间转换。


jOOQ

| 类型/次数   | 1    | 10   | 100  |    1000  |   1w  |   10w  |
|   ---     |  ---  | ---  |  ---  |   ---   |   ---  |   ---  |
| 实现from  |  ~0ms |  ~1ms |  ~5ms |   ~5ms  |  ~5ms  | ~20ms  |
| 反射方式  |  ~0ms | ~3ms  | ~60ms |   ~100ms | ~340ms |~2200ms |
