---
title: Hibernate4 JPA注解详解（上）
date: 2018-4-20 14:15:03
tags: Hibernate
---

#### 前言

本期我们对Hibernate JPA注解的配置进行说明，这些注解如何使用，如何配置等。

#### @Table

Table 用来定义 entity 主表的 name，catalog，schema 等属性。

属性说明：

* name：表名
* catalog：对应数据库的catalog
* schema：对应关系数据库中的schema
* UniqueConstraints：定义一个 UniqueConstraint 数组，指定需要建唯一约束的列

```java
@Entity
@Table(name="CUST")
public class Customer { ... }
```

#### @SecondaryTable

一个 entity class 可以映射到多表，SecondaryTable 用来定义单个从表的名字，主键名字等属性。
属性说明：

<!-- more -->

* name：表名
* catalog：对应关系数据库中的 catalog
* pkJoin：定义一个 PrimaryKeyJoinColumn 数组，指定从表的主键列
* UniqueConstraints：定义一个 UniqueConstraint 数组，指定需要建唯一约束的列

下面的代码说明 Customer 类映射到两个表，主表名是 CUSTOMER，从表名是CUST_DETAIL，从表的主键列和主表的主键列类型相同，列名为 CUST_ID。

```java
@Entity
@Table(name="CUSTOMER")
@SecondaryTable(name="CUST_DETAIL",pkJoin=@PrimaryKeyJoinColumn(name="CUST_ID"))
public class Customer { ... }
```

#### @SecondaryTables

当一个 entity class 映射到一个主表和多个从表时，用 SecondaryTables 来定义各个从表的属性。
属性说明：

* value：定义一个 SecondaryTable 数组，指定每个从表的属性。

```java
@Table(name = "CUSTOMER")
@SecondaryTables( value = {
        @SecondaryTable(name = "CUST_NAME", pkJoin = { @PrimaryKeyJoinColumn(name = "STMO_ID", referencedColumnName = "id") }),
        @SecondaryTable(name = "CUST_ADDRESS", pkJoin = { @PrimaryKeyJoinColumn(name = "STMO_ID", referencedColumnName = "id") }) })
public class Customer {}
```

#### @UniqueConstraint

UniqueConstraint 定义在 Table 或 SecondaryTable 元数据里，用来指定建表时需要建唯一约束的列。
属性说明：

* columnNames：定义一个字符串数组，指定要建唯一约束的列名。

```java
@Entity
@Table(name="EMPLOYEE",uniqueConstraints={@UniqueConstraint(columnNames={"EMP_ID","EMP_NAME"})})
public class Employee { ... }
```

#### @Column

Column 元数据定义了映射到数据库的列的所有属性：列名，是否唯一，是否允许为空，是否允许更新等。
属性说明：

* unique：是否唯一
* nullable：是否允许为空
* insertable：是否允许插入
* updatable：是否允许更新
* columnDefinition：定义建表时创建此列的 DDL
* secondaryTable：从表名。如果此列不建在主表上（默认建在主表），该属性定义该列所在从表的名字。

```java
public class Person {
    @Column(name = "PERSONNAME", unique = true, nullable = false, updatable = true)
    private String name;

    @Column(name = "PHOTO", columnDefinition = "BLOB NOT NULL", secondaryTable="PER_PHOTO")
    private byte[] picture;
}
```

#### @JoinColumn

如果在 entity class 的 field 上定义了关系（one2one 或 one2many 等），我们通过 JoinColumn来定义关系的属性。JoinColumn 的大部分属性和 Column 类似。
属性说明：

* name：另一个表指向本表的外键


* unique：是否唯一
* referencedColumnName：该列指向列的列名（建表时该列作为外键列指向关系另一端的指定列）
* nullable：是否允许为空
* insertable：是否允许插入
* updatable：是否允许更新
* columnDefinition：定义建表时创建此列的 DDL
* secondaryTable：从表名。如果此列不建在主表上（默认建在主表），该属性定义该
  列所在从表的名字。

下面的代码说明 Custom 和 Order 是一对一关系。在 Order 对应的映射表建一个名为CUST_ID 的列，该列作为外键指向 Custom 对应表中名为 ID 的列。

```java
public class Custom {
    @OneToOne
    @JoinColumn(name = "CUST_ID", referencedColumnName = "ID", unique = true, nullable = true, updatable = true)
    public Order getOrder() {
        return order;
    }
}
```

#### @JoinColumns

如果在 entity class 的 field 上定义了关系（one2one 或 one2many 等），并且关系存在多个 JoinColumn，用 JoinColumns 定义多个 JoinColumn 的属性。
属性说明：

* value：定义 JoinColumn 数组，指定每个 JoinColumn 的属性。

下面的代码说明 Custom 和 Order 是一对一关系。在 Order 对应的映射表建两列，一列名为CUST_ID，该列作为外键指向Custom对应表中名为ID的列,另一列名为CUST_NAME，该列作为外键指向 Custom 对应表中名为 NAME 的列。

```java
public class Custom {
    @OneToOne
    @JoinColumns({
            @JoinColumn(name = "CUST_ID", referencedColumnName = "ID"),
            @JoinColumn(name = "CUST_NAME", referencedColumnName = "NAME")
    })
    public Order getOrder() {
        return order;
    }
}
```

#### @Id

声明当前 field 为映射表中的主键列。id 值的获取方式有五种：TABLE, SEQUENCE,IDENTITY, AUTO, NONE。Oracle 和 DB2 支持 SEQUENCE，SQL Server 和 Sybase 支持 IDENTITY,mysql 支持 AUTO。所有的数据库都可以指定为 AUTO，我们会根据不同数据库做转换。NONE(默认)需要用户自己指定 Id 的值。
属性说明：

* generate：主键值的获取类型
* generator：TableGenerator 的名字（当 generate=GeneratorType.TABLE 才需要指定
  该属性）

下面的代码声明 Task 的主键列 id 是自动增长的。(Oracle 和 DB2 从默认的 SEQUENCE取值，SQL Server 和 Sybase 该列建成 IDENTITY，mysql 该列建成 auto increment。)

```java
@Entity
@Table(name = "OTASK")
public class Task {
    @Id(generate = GeneratorType.AUTO)
    public Integer getId() {
        return id;
    }
}
```

#### @IdClass

当 entity class 使用[复合主键](https://blog.csdn.net/u011781521/article/details/71083112)时，需要定义一个类作为 id class。id class 必须符合以下要求:类必须声明为 public，并提供一个声明为 public 的空构造函数。必须实现 Serializable 接，覆写 equals()和 hashCode（）方法。entity class 的所有 id field 在 id class 都要定义，且类型一样。
属性说明：

* value：id class 的类名

下面的代码声明 Task 的主键列 id 是自动增长的。(Oracle 和 DB2 从默认的 SEQUENCE取值，SQL Server 和 Sybase 该列建成 IDENTITY，mysql 该列建成 auto increment。)

```java
public class EmployeePK implements java.io.Serializable {
    String empName;
    Integer empAge;

    public EmployeePK() {
    }

    public boolean equals(Object obj) { ......}

    public int hashCode() {......}
}
```

```java
@IdClass(value = com.acme.EmployeePK.class)
@Entity(access = FIELD)
public class Employee {
    @Id
    String empName;
    @Id
    Integer empAge;
}
```

#### @MapKey

在一对多，多对多关系中，我们可以用 Map 来保存集合对象。默认用主键值做 key，如果使用复合主键，则用 id class 的实例做 key，如果指定了 name 属性，就用指定的 field 的值做 key。
属性说明：

* name：用来做 key 的 field 名字

下面的代码说明 Person 和 Book 之间是一对多关系。Person 的 books 字段是 Map 类型，用 Book 的 isbn 字段的值作为 Map 的 key。

```java
@Table(name = "PERSON")
public class Person {
    @OneToMany(targetEntity = Book.class, cascade = CascadeType.ALL, mappedy = "person")
    @MapKey(name = "isbn")
    private Map books = new HashMap();
}
```

#### @MappedSuperclass

使用@MappedSuperclass 指定一个实体类从中继承持久字段的超类。当多个实体类共享通用的持久字段或属性时，这将是一个方便的模式。

您可以像对实体那样使用任何直接和关系映射批注（如 @Basic 和 @ManyToMany）对该超类的字段和属性进行批注，但由于没有针对该超类本身的表存在，因此这些映射只适用于它的子类。继承的持久字段或属性属于子类的表。

可以在子类中使用@AttributeOverride 或@AssociationOverride 来覆盖超类的映射配置。

@MappedSuperclass 没有属性。

```java
//如何将 Employee 指定为映射超类
@MappedSuperclass
public class Employee {
    @Id
    protected Integer empId;
    @Version
    protected Integer version;
    @ManyToOne
    @JoinColumn(name = "ADDR")
    protected Address address;
}

//如何在实体类中使用@AttributeOverride 以覆盖超类中设置的配置。
@Entity
@AttributeOverride(name = "address", column = @Column(name = "ADDR_ID"))
public class PartTimeEmployee extends Employee {
    @Column(name = "WAGE")
    protected Float hourlyWage;
}
```

#### @PrimaryKeyJoinColumn

在三种情况下会用到@PrimaryKeyJoinColumn

* 继承。
* entity class 映射到一个或多个从表。从表根据主表的主键列（列名为referencedColumnName 值的列），建立一个类型一样的主键列，列名由 name 属性定义。
* one2one 关系，关系维护端的主键作为外键指向关系被维护端的主键，不再新建一个外键列。

属性说明：

* name：列名。
* referencedColumnName：该列引用列的列名
* columnDefinition：定义建表时创建此列的 DDL

下面的代码说明 Customer 映射到两个表，主表 CUSTOMER,从表 CUST_DETAIL，从表需要建立主键列 CUST_ID，该列和主表的主键列 id 除了列名不同，其他定义一样。

```java
@Entity
@Table(name = "CUSTOMER")
@SecondaryTable(name = "CUST_DETAIL", pkJoin = @PrimaryKeyJoinColumn(name = "CUST_ID"，referencedColumnName = " id"))
public class Customer {
    @Id(generate = GeneratorType.AUTO)
    public Integer getId() {
        return id;
    }
}
```

下面的代码说明 Employee 和 EmployeeInfo 是一对一关系，Employee 的主键列 id 作为外键指向 EmployeeInfo 的主键列 INFO_ID。

```java
@Table(name = "Employee")
public class Employee {
    @OneToOne
    @PrimaryKeyJoinColumn(name = "id", referencedColumnName = "INFO_ID")
    EmployeeInfo info;
}
```

#### @PrimaryKeyJoinColumns

如果 entity class 使用了复合主键，指定单个 PrimaryKeyJoinColumn 不能满足要求时，可
以用 PrimaryKeyJoinColumns 来定义多个 PrimaryKeyJoinColumn
属性说明：

* value： 一个 PrimaryKeyJoinColumn 数组，包含所有 PrimaryKeyJoinColumn下面的代码说明了 Employee 和 EmployeeInfo 是一对一关系。他们都使用复合主键，建表时需要在 Employee 表建立一个外键，从 Employee 的主键列 id,name 指向 EmployeeInfo的主键列 INFO_ID 和 INFO_NAME

```java
@Entity
@IdClass(EmpPK.class)
@Table(name = "EMPLOYEE")
public class Employee {
    private int id;
    private String name;
    private String address;
    @OneToOne(cascade = CascadeType.ALL)
    @PrimaryKeyJoinColumns
    @PrimaryKeyJoinColumn(name = "id", referencedColumnName = "INFO_ID")
    @PrimaryKeyJoinColumn(name = "name", referencedColumnName = "INFO_NAME") })
        EmployeeInfo info;
        }

@Entity
@IdClass(EmpPK.class)
@Table(name = "EMPLOYEE_INFO")
public class EmployeeInfo {
    @Id
    @Column(name = "INFO_ID")
    private int id;
    @Id
    @Column(name = "INFO_NAME")
    private String name;
}
```

#### @Transient

Transient 用来注释 entity 的属性，指定的这些属性不会被持久化，也不会为这些属性建表

```java
@Transient
private String name;
```

#### @Version

Version 指定实体类在乐观事务中的 version 属性。在实体类重新由 EntityManager 管理并
且加入到乐观事务中时，保证完整性。每一个类只能有一个属性被指定为 version，version
属性应该映射到实体类的主表上。
属性说明：

* value： 一个 PrimaryKeyJoinColumn 数组，包含所有 PrimaryKeyJoinColumn下面的代码说明 versionNum 属性作为这个类的 version，映射到数据库中主表的列名是OPTLOCK

```java
@Version
@Column("OPTLOCK")
protected int getVersionNum() { return versionNum; }
```

----

我的博客：[https://zhaojun0193.github.io](https://zhaojun0193.github.io)

本文代码地址：[https://github.com/zhaojun0193/shiro-example](https://github.com/zhaojun0193/shiro-example)