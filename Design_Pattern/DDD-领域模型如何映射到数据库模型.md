

DDD的建模方法是根据业务先进行领域建模，领域建模的结果包括聚合根、实体和值对象等，而聚合根实际上也是实体。一个系统在运行过程中必定要对某些数据进行持久化，那么需要持久化哪些领域对象呢？领域对象又如何转换为数据库持久化模型呢？

从持久化的角度来看，领域对象包括两种类型：

- 需要持久化的领域对象：包括实体和值对象
- 不需要持久化的领域对象：在运行过程中产生的值对象

因此，实体一定需要持久化；值对象如果是对实体的描述，也是需要进行持久化；值对象如果是在运行过程中产生的，则不需要持久化。

在领域对象转换为数据库持久化模型时，需要考虑以下几种情况领域对象间的关系：

- 一对一
- 一对多
- 多对一
- 多对多
- 继承

# 1. 实体对象的持久化

> 本节讨论的数据范围仅限于实体与实体之间的关系，实体与值对象间的关系，将在下一节讨论。

实体对象映射到数据模型时，一个实体可能对应 一 个或者多个数据库持久化对象。

## 1.1 非聚合根实体对象的持久化

**多数情况下，普通的实体（非聚合根）与持久化对象是一对一的关系，即一个实体可以转换为一个需要持久化的对象，**如一个用户实体对象User可以转换为一个持久化的用户对象UserPo。

```java
// User领域实体
public class User extends DomainEntity<String> {
  private String userName;
  private int age;
  private enum sex;
}

// User持久化对象，Po是Persistent Object的缩写
public class UserPo {
  private String id;
  private String userName;
  private int age;
  private enum sex;
}
```

| id   | user_name | age  | sex  |
| ---- | --------- | ---- | ---- |
| 1    | lishi     | 16   | Male |



## 1.2 聚合根实体对象的持久化

如果一个实体是聚合根，作为聚合的管理者，在聚合根内部直接引用实体和值对象，负责协调实体和值对象按照固定的业务规则协同完成共同的业务逻辑，因此**聚合根与持久化对象存在一对一、一对多和多对一的关系**。

### 1.2.1 一对一关系的持久化

聚合根实体对象下，一对一这种关系的持久化非常简单，只要用一个实体对象的持久化对象存储另一个实体对象的持久化对象的主键即可。比如，一个User实体对象只能有一种Role，此时，可以在User实体对象的持久化对象上存储Role持久化对象的主键，如：

```java
// User领域实体
public class User extends DomainEntity<String> implements AggregateRoot {
  private String userName;
  private int age;
  private Role role;
}

// Role领域实体
public class Role extends DomainEntity<String> {
  private String roleName;
  private List<String> permissions;
}

// User持久化对象，Po是Persistent Object的缩写
public class UserPo {
  private String id;
  private String userName;
  private int age;
  private String roleId; //持有对Role持久化对象的引用

}

// Role持久化对象
public class RolePo {
  prviate String id;
  private String roleName;
  private String permission;
}
```

| id   | user_name | age  | role_id |
| ---- | --------- | ---- | ------- |
| 1    | Lishi     | 16   | 1       |

| Id   | role_name | permission               |
| ---- | --------- | ------------------------ |
| 1    | admin     | create_user, delete_user |



### 1.2.2 一对多和多对一关系的持久化

如果聚合根实体对象中存在一对多或多对一的关系，在持久化时需要**在多的一方中存储聚合根实体对象的ID**，从而建立起一对多的关系。如在电商场景中，一个用户User可以有多个收件地址DeliveryAddress。

```java
// User领域实体
public class User extends DomainEntity<String> {
  private String userName;
  private int age;
  private enum sex;
  private List<DeliveryAddress> deliveryAddresses;
}

// DeliveryAddress
public class DeliveryAddress extends DomainEntity<String> {
  private String area;
  private String detailAddress;
  private String phoneNumber;
  private String receiver;
}

// User持久化对象
public class UserPo {
  private String id;
  private String userName;
  private int age;
  private enum sex;
}


// DeliveryAddress持久化对象
public class DeliveryAddressPo {
  private String id;
  private String area;
  private String detailAddress;
  private String phoneNumber;
  private String receiver;
  private String userId; //存储User ID
}
```

### 1.1.3 多对多关系的持久化

如果聚合根实体对象中存在多对多的关系，在持久化时通常**需要创建一个单独的持久化对象**，在该持久化对象中保存多对多关系中的双方实体的id。如一本书Book是由一个或多个作者Writer撰写的；而作者Writer可能写零本书Book或更多本书Book。

```java
// User领域实体
public class Book extends DomainEntity<String> implements AggregateRoot {
  private String bookName;
  private List<Writer> writers;
}

// Writer领域实体
public class Writer extends DomainEntity<String> {
  private String userName;
}

// Book持久化对象
public class BookPo {
  private String id;
  private String bookName;
}

// Writer持久化对象
public class WriterPo {
  prviate String id;
  private String userName;
}

// 保存Book和Writer的关系
public class UserBookRelationPo {
  private String id;
  private String bookId;
  private String writerId;
}
```

# 2. 值对象的持久化

> 本节讨论实体与值对象间的持久化关系

在领域建模中，值对象隶属于实体，一个值对象在一个实体中的数量范围可能是0个，1个或多个。因此，在实体与值对象的持久化过程中，通常需要考虑两种关系：

- 一对一
- 一对多

## 2.1 一对一关系的持久化

实体与值对象的关系如果是一对一，我们可以由两种方式持久化值对象。

**方式一：将值对象的属性嵌入到实体对象中**

如用户User和家庭住址HomeAddress之间就是一对一关系，并且家庭住址HomeAddres是值对象。

```java
// User领域实体
public class User extends DomainEntity<String> {
  private String userName;
  private int age;
  private HomeAdress homeAdress;
}

// DeliveryAddress值对象
public class HomeAddress extends ValueObject {
  private String area;
  private String detailAddress;
}

// User持久化对象
public class UserPo {
  private String id;
  private String userName;
  private int age;
  private String area; //将HomeAddress的area属性嵌入到UserPo中
  private String detailAddress //将HomeAddress的detailAddress属性嵌入到UserPo中
}
```

**方式二：将值对象序列化后存储为实体对象的一列**

依然以User和HomeAddress为例：

```java
// User领域实体
public class User extends DomainEntity<String> {
  private String userName;
  private int age;
  private HomeAdress homeAdress;
}

// DeliveryAddress值对象
public class HomeAddress extends ValueObject {
  private String area;
  private String detailAddress;
}

// User持久化对象
public class UserPo {
  private String id;
  private String userName;
  private int age;
  private String homeAddress; //如将HomeAddress序列化为json字符串嵌入到UserPo中
}
```

| Id   | user_name | age  | home_address                                      |
| ---- | --------- | ---- | ------------------------------------------------- |
| 1    | Zhangshan | 18   | {"area":"陕西西安", "detailAdress":"清河湾128号"} |

**特别提示:**  在关系型数据库中，方式二无法满足基于值对象的快速查询，会导致搜索值对象属性值变得异常困难。

## 2.2 一对多关系的持久化

如果实体对象与值对象的关系是一对多，最常用的方法是将**值对象序列化为一个大对象后存储为实体对象在数据库中的一个字段**，这样的存储结构在关系型数据中带来的问题是：**无法满足基于值对象的快速查询，会导致搜索值对象属性值变得异常困难**。

如果业务中的确需要对值对象进行快速查询，此时只能将值对象升级为实体对象。如果在业务建模过程中，存在大量的需要持久化的且是一对多的值对象，此时可以考虑采用非关系型数据库进行存储。

# 3. 继承关系的持久化

is-a关系的实体对象通常需要采用基类的方式实现公共的方法和属性，子类继承基类实现各自特有的属性和行为，这类实体在持久化时会有两种方式：

**方式一：持久化时有新增一个type字段，将子类特有的属性序列化后存储为数据表中的一列**

在通过持久化结果反向构建实体对象时通过type的值实例化具体的实体对象。如用户包括VIP用户，VIP用户是is-a User的关系，因此可以继承User基类：

```java
// User entity
public class User extends DomainEntity<String> {
  private String id;
  private String userName;
  private String phoneNumber;
}

// VIPUser Entity
public class VIPUser extends User {
  private String hobby;
  private String wechat;
  private LocalDateTime birthday;
}

// User持久化对象
public class UserPo {
  private String id;
  private String type; // type标记持久化对象的原始类型
  private String userName;
  private String phoneNumber;
  private String extendInfo;
}

```

| id   | type     | user_name | hone_number | extend_inf                                                  |
| ---- | -------- | --------- | ----------- | ----------------------------------------------------------- |
| 1    | USER     | zhangshan | 12345678901 |                                                             |
| 2    | VIP_USER | Lishi     | 09876543212 | {"hobby":"cycling","wechat":"12345678", "brithday":"09.06"} |

**方式二：将不同的子对象持久化到不同的表中**

通常采用这中持久化策略的原因是：**在关系型数据库中，需要对子类的对象进行查询操作**。（在非关系型数据库不需要采用该种方式，如MongoDB）

```java
// User持久化对象
public class UserPo {
  private String id;
  private String type; // type标记持久化对象的原始类型
  private String userName;
  private String phoneNumber;
}

public class VipUserPo {
  private String id;
  private String type; // type标记持久化对象的原始类型
  private String userName;
  private String phoneNumber;
  private String hobby;
  private String wechat;
  private LocalDateTime birthday;
}
```

