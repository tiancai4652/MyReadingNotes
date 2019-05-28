# Sqlite && EF Code FIRST 终极解决方案 2019.5.17#
**包括根据模型自动生成数据库，初始化数据，模型改变时的自动数据迁移等**

## 目录 ##

- 01 知识汇总
- 02 在过去的经验里EF与Sqlite的使用经验
- 03 Sqlite数据库在EF Code First中遇到的困难
- 04 解决方案
- 05 解决方案的不足
- 06 与老方案对比


## 01 知识汇总 ##

### What is EF ###
Entity Framework是MS提供的一个ORM框架，它旨在为小型应用程序中数据层的快速开发提供便利。

ORM 技术是在对象和关系之间提供了一条桥梁，前台的对象型数据和数据库中的关系型的数据通过这个桥梁来相互转化,这样，我们在具体的操作实体对象的时候，就不需要再去和复杂的 SQ L 语句打交道，只需简单的操作实体对象的属性和方法。

Code First模式我们称之为“代码优先”模式，是从EF4.1开始新建加入的功能。使用Code First模式进行EF开发时开发人员只需要编写对应的数据类（其实就是领域模型的实现过程），然后自动生成数据库。这样设计的好处在于我们可以针对概念模型进行所有数据操作而不必关系数据的存储关系，使我们可以更加自然的采用面向对象的方式进行面向数据的应用程序开发。

### 遇到的讲解EF非常好的博客 ###

EF性能全面讲解:https://www.cnblogs.com/yaopengfei/p/9196962.html

EF迁移全面讲解:https://www.cnblogs.com/farb/p/DBMigration.html

### What is Sqlite ###

SQLite是一种嵌入式数据库，它的数据库就是一个文件，它占用资源非常的低。

## 02 Sqlite数据库在EF Code First 中遇到的困难 ##

 - 设计上的EF瓶颈:无法实现接口类的数据库化
 - EF的Sqlite Data Provider无法提供SQL Generation
 - Sqlite数据库自身许多语法不支持，后续模型修改的数据迁移Sql语句比较受限
 - EF瓶颈:对于视图支持不好，但有临时的解决方案
 

## 03 在过去的经验里EF与Sqlite的使用经验 ##

由于Sqlite的Data Provider不支持Sql Generation 所以EF无法从模型中自动创建数据库，所以创建数据库以及后续修改模型时需要的修改数据库工作手动去完成.

 - Model - 构建模型 - 修改模型
 - DataBase - 手写Sql构建数据库，手写Sql填充初始数据 - 手写Sql修改模型

EF COde First利用数据库中自动生成的__MigrationHistory表的信息与当前代码的模型去判断是否需要更新数据库表结构  
手写SQL语句是通过数据库某一个标识去判断需不需要更新，例如：Version列

## 04 解决方案 ##

### Sql Generation解决方法 ###

目前我发现两个实现了Sqlite SQL Generation的开源库:
[System.Data.SQLite.EF6.Migrations](https://github.com/bubibubi/db2ef6migrations)   
[SQLite.CodeFirst](https://github.com/digocesar/SQLiteCodeFirst)


目前来看的话两种库几乎一样，都是添加了关于Sql Generation的方法，两个库都可在NuGet上获取。  

需要注意的是，SQLite.CodeFirst在NuGet上的最新版1.5.2.28并不具有数据迁移功能，该Github项目并没有集成，具有数据迁移功能的版本可以点击上方链接，是另一个人在此基础上集成的。  


<font color=red>另外，最新的EF6.2在初始化数据的时候会有错误，项目建议在Add-Migration的时候降级EF到6.13，我当前的办法是使用EF6.3预览版.</font>


### 解决接口类的数据库化 ###

目前想到的方法只有一个基类去实现这个接口，然后接口的实现类去继承这个基类


    public interface ICourse: Iid
    {
        string Name { get; set; }
        ICollection<Student> Students { get; set; }
    }   

    public class CourseBase : ICourse
    {
        public string Name { get; set; }
        public virtual ICollection<Student> Students { get; set; }
        public string ID { get; set ; }
    }    

	[Table("Chinese")]
    public class Chinese : CourseBase
    {
        public Chinese()
        {
            ID = Guid.NewGuid().ToString("N");
        }
        public string Extend { get; set; }
        public string Extend1 { get; set; }
    }

然后在模型中：

    public class Student:Iid
    {
        public Student()
        {
            ID = Guid.NewGuid().ToString("N");
        }
        public string Name { get; set; }
        public Weapon.Weapon Weapon { get; set; }
        public string ID { get; set; }
        public virtual ICollection<CourseBase> Courses { get; set; }
    }

用CourseBase代替ICource

### 视图解决方案（通用） ###

EF本身是不支持View的，但是如果你的数据库已经存在View，你是可以去建Table模型一样去建立View模型并查询他。

那么我们只需要解决两个问题就可以实现视图的控制：

- 初始化数据库时如何建立View
- 修改数据模型时，Migration时屏蔽View的实体类修改的影响

目前通用的解决方案是

 - 初始化数据时手写Sql去建立View
 - 不用自动数据迁移，而用API去控制迁移
 - 如果需要修改View，手动Drop视图并再次Add New View

1 关闭自动迁移，创建初始化API Add-Migration

    AutomaticMigrationsEnabled = false;

2 在Up方法中手写Sql语句创建View

     public override void Up()
        {
            string script =
        @"
        CREATE VIEW [StudentWeaponView]
        AS SELECT p.ID AS StudentID, p.Name AS StudentName,u.ID AS WeaponID,u.Name AS WeaponName
        FROM [Students] p
        INNER JOIN [Weapon] u ON u.Id = p.Id";
            DBContext.DBContext ctx = new DBContext.DBContext();
            ctx.Database.ExecuteSqlCommand(script);
        }

3 当你想修改模型时，执行Add-Migration ChangeViewNmae

     public override void Up()
        {
            string script =
       @"
        Drop View If Exists [StudentWeaponView];";
            DBContext.DBContext ctx = new DBContext.DBContext();
            ctx.Database.ExecuteSqlCommand(script);

            string script2 =
        @"
        CREATE VIEW [StudentWeaponView]
        AS SELECT p.ID AS StudentID, p.Name AS StudentName,u.ID AS WeaponID,u.Name AS WeaponName
        FROM [Students] p
        INNER JOIN [Weapon] u ON u.Id = p.Id";
            ctx.Database.ExecuteSqlCommand(script2);
        }

## 05 当前解决方案的不足 ##
 - 需要使用EF3 Preview Or Data Migration时降级EF到6.13版本
 - 视图支持不好，当然不仅仅是针对Sqlite，EF的解决方案通病
 - EF数据迁移初始化数据方式单一

这里解释一下第三点，看一下使用方法的对比就可以了

老方案(手写SQL创建数据库）初始化数据库:

 - 脚本1:创建数据库
 - 脚本2:添加两条数据
 - 脚本3:增加一个字段
 - 脚本4:添加一条数据

EF Code First 初始化数据:

 - Migration1：创建数据库
 - Migration1：增加一个字段
 - Seed方法:添加两条数据，添加一条数据

Seed方法是每次执行完从低版本到高版本的数据迁移都会执行的方法，官方推荐用AddOrUpdate方法，就是因为每次迁移都会调用，所以会出现重复数据的情况

那么上面两种方法有什么区别呢？
如果下面Seed方法中，原封不动翻译上面的脚本2和脚本4，那么是一定会报错的，为什么？

因为在上面的脚本2执行中，Sql模型中是没有脚本三增加的字段的，而下面Seed方法执行时，脚本三的字段已经添加完毕了，即在Seed方法执行的时候，调用的是最终的数据模型，上面每次执行脚本的时候，调用的当前版本的数据模型

也就是说，如果你想实现一个版本一个版本增量的初始化数据库内容，那么在Seed方法中你要进行大量的对于版本的判断。

目前的解决办法:

数据库架构方面的版本是由EF的__MigrationHistory控制的

那么数据库初始内容上也加一个版本号

然后

    if(version>1)
	{
		//脚本1初始数据内容.....
	}

    if(version>2)
	{
		//脚本2初始数据内容.....
	}


附上[完整代码地址](https://github.com/tiancai4652/Schaco-EF)实现了Sqlite有关于EF Code First的全部功能。

Master利用的是System.Data.SQLite.EF6.Migrations库  
分支用的是SQLite.CodeFirst库



 