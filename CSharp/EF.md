## EntityFrameworkCore(EF)

### SQL Server

##### Program.cs配置DBContext

```C#
builder.Services.AddDbContext<EfexampleContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("EFExample")));
```

##### 将写好的实体类自动生成数据库

```C#
Add-Migration /*CustomName*/
Update-Database
```

##### 将建立好的数据库自动生成实体类

```C#
Scaffold-DbContext "Server=[服务器名称];Database=[数据库名称];UserId=[用户名];Password=[密码]; TrustServerCertificate=true" Microsoft.EntityFrameworkCore.SqlServer                                 -OutputDir [实体类文件夹路径] (Models) -Context [DbContext名称] (Data) -Namespace [命名空间] -Force
```

### ***MySql***

##### 导入包

```c#
using Microsoft.EntityFrameworkCore;
using System.Configuration;
using Microsoft.Extensions.Configuration;
using Pomelo.EntityFrameworkCore.MySql.Infrastructure;
using Pomelo.EntityFrameworkCore.MySql.Internal;
```

##### Program.cs配置

```C#
var config = bder.Build();
string connectionSting = config.GetConnectionString("Connections") ?? "Default";

string mySqlVersion = "8.0.32";
var serverVersion = new MySqlServerVersion(new Version(mySqlVersion));
builder.Services.AddDbContextPool<AppDbContext>(opt =>
    opt.UseMySql(builder.Configuration.GetConnectionString("Connections"), new MySqlServerVersion(serverVersion)));
```

如果导入Mysql连接包为 Pomelo.EntityFrameworkCore.MySql 则调用 opt.UseMySql                                                                                       若为 MySql.EntityFrameworkCore 则调用 opt.UseMySQL （Mysql.Data.EntityFramworkCore已过期，建议使用前者）                                                                                                          两者之间的区别在于大小写

- `UseMySql()` 是区分大小写的，用于在 Entity Framework Core 中配置连接到 MySQL 数据库的选项。
- `UseMySQL()` 是不区分大小写的，也用于在 Entity Framework Core 中配置连接到 MySQL 数据库的选项。

##### DB First 自动生成实体类命令：

```
Scaffold-DbContext "Server=[服务器名称];Database=[数据库名称];UserId=[用户名];Password=[密码]" MySql.Data.EntityFrameworkCore -OutputDir [实体类文件夹路径] -Context [DbContext名称] 
-Namespace [命名空间] -Force
```

其中，[服务器名称]、[数据库名称]、[用户名] 和 [密码] 分别是您的 MySQL 数据库连接字符串的相关信息。[实体类文件夹路径] 是自动生成实体类文件的存储路径。[DbContext名称] 则是您自己定义的上下文类名称。-Force 参数可强制重新生成平台非关键字实体类型 (例如 Audit)，请谨慎使用该参数。

在命令执行后，会在指定的文件夹中生成多个 C# 实体类文件。每个实体类对应数据库中的一张表格，其中包含了表中的列信息。此外，命令还会生成一个包含所有实体类的 DbContext 类文件。

##### Code First(别问，问就是还没用过，到时再说):

xxxxxxxxxxxxxxxxxxxx

```C#
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 可能报错

dotnet : Could not execute because the specified command or file was not found.

```shell
dotnet tool install --global dotnet-ef
```

nuget : The term 'nuget' is not recognized as the name of a cmdlet, function, script file, or operable program. Check the spelling of the name, or if a path was included, verify that the path is correct and try again.（无法访问Nuget）

```
Install-Package NuGet.CommandLine
```

