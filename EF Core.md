# EF Core

1. DbContext

   - 方法OnModelCreating：配置实体映射

     - Fluent API配置

     ```c#
     protected override void OnModelCreating(ModelBuilder modelBuilder)
         {
             var allTypes = Assembly.GetExecutingAssembly().GetTypes()
                 .Where(t => t.IsClass && typeof(IEntity).IsAssignableFrom(t))
                 .ToList();
     
             foreach (var type in allTypes.Where(type => modelBuilder.Model.FindEntityType(type) == null))
             {
                 modelBuilder.Model.AddEntityType(type);
             }
         }
     ```

     

     - Data Annotations配置

     ```c#
     [Table("Customers")]
     public class Customer
     {
         [Key]
         public int Id { get; set; }
     }
     ```

   - 使用数据注解进行表字段的映射

   ```c#
   [Table("product")]
   public class Product : IEntity
   {
       [Column("id")]
       public Guid Id { get; set; }
   
       [Column("name")]
       public string Name { get; set; }
   
       [Column("price")]
       public double Price { get; set; }
   }
   ```

2. IRepository.cs：封装了一系列对数据库操作的方法，相当于封装了一个mybatis

   ```c#
   public interface IRepository
   {
       ValueTask<TEntity?> GetByIdAsync<TEntity>(object id, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task<List<TEntity>> GetAllAsync<TEntity>(CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task<List<TEntity>> ToListAsync<TEntity>(Expression<Func<TEntity, bool>> predicate,
           CancellationToken cancellationToken = default) where TEntity : class, IEntity;
   
       Task InsertAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task InsertAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task UpdateAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task UpdateAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task DeleteAsync<TEntity>(Expression<Func<TEntity, bool>> predicate,
           CancellationToken cancellationToken = default) where TEntity : class, IEntity;
       
       Task DeleteAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task DeleteAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task<int> CountAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task<TEntity?> SingleOrDefaultAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task<TEntity?> FirstOrDefaultAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task<bool> AnyAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default)
           where TEntity : class, IEntity;
   
       Task<List<T>> SqlQueryAsync<T>(string sql, params object[] parameters)
           where T : class, IEntity;
   
       IQueryable<TEntity> Query<TEntity>(Expression<Func<TEntity, bool>>? predicate = null)
           where TEntity : class, IEntity;
   
       IQueryable<TEntity> QueryNoTracking<TEntity>(Expression<Func<TEntity, bool>>? predicate = null)
           where TEntity : class, IEntity;
   
       DatabaseFacade Database { get; }
       Task BatchInsertAsync<TEntity>(IList<TEntity> entities) where TEntity : class, IEntity;
       Task BatchUpdateAsync<TEntity>(IList<TEntity> entities) where TEntity : class, IEntity;
       Task BatchDeleteAsync<TEntity>(Expression<Func<TEntity, bool>> predicate) where TEntity : class, IEntity;
   }
   ```



- 数据库连接（MySQL）：

  1. 在appsettings.json编写连接语句：

     ```json
       "ConnectionStrings": {
         "Default": "server=[server];userid=[userid];password=[password];database=[database];Allow User Variables=True;"
       }
     ```

     

  2. 在实现了DbContext的类中，重写OnConfiguring方法：

     ```c#
     public PractiseForMasonDBContext(IConfiguration configuration)
     {
       // 读取配置文件中的连接语句
     	var section = configuration.GetSection("ConnectionStrings:Default");
             _dbConnectionString = section.Value;
     }
         
     protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
     {
     	optionsBuilder.UseMySql(_dbConnectionString, new MySqlServerVersion(new Version(8, 0, 38)));
     }
     ```

  3. 在autofac的配置Model文件中注册Dbcontext：

     ```c#
     private void RegisterDbContext(ContainerBuilder builder)
         {
             builder.RegisterType<PractiseForMasonDBContext>()
                 .AsSelf()
                 .As<DbContext>()
                 .AsImplementedInterfaces()
                 .WithParameter((info, context) => info.ParameterType == typeof(ConnectionString),
                     (info, context) => context.Resolve<ConnectionString>())
                 .InstancePerLifetimeScope();
     
             builder.RegisterType<EFRepository>().As<IRepository>().InstancePerLifetimeScope();
         }
     ```

     

- EFRepository: IRepository 的实现，封装了基本的通用crud

  ```c#
  using System.Linq.Expressions;
  using EFCore.BulkExtensions;
  using Microsoft.EntityFrameworkCore;
  using Microsoft.EntityFrameworkCore.Infrastructure;
  using PractiseForMason.Core.Domain;
  
  namespace PractiseForMason.Core.Data;
  
  public class EFRepository : IRepository
  {
      private readonly PractiseForMasonDBContext _dbContext;
  
      public EFRepository(PractiseForMasonDBContext dbContext)
      {
          _dbContext = dbContext;
      }
      
      public ValueTask<TEntity?> GetByIdAsync<TEntity>(object id, CancellationToken cancellationToken = default) 
          where TEntity : class, IEntity
      {
          return _dbContext.FindAsync<TEntity>(new[] { id }, cancellationToken);
      }
  
      public Task<List<TEntity>> GetAllAsync<TEntity>(CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          return _dbContext.Set<TEntity>().ToListAsync(cancellationToken);
      }
  
      public Task<List<TEntity>> ToListAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          return _dbContext.Set<TEntity>().Where(predicate).ToListAsync(cancellationToken);
      }
  
      public async Task InsertAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          await _dbContext.AddAsync(entity, cancellationToken).ConfigureAwait(false);
          _dbContext.ShouldSaveChanges = true;
      }
  
      public async Task InsertAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          await _dbContext.AddRangeAsync(entities, cancellationToken).ConfigureAwait(false);
          _dbContext.ShouldSaveChanges = true;
      }
  
      public Task UpdateAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          _dbContext.Update(entity);
          _dbContext.ShouldSaveChanges = true;
          return Task.CompletedTask;
      }
  
      public Task UpdateAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          _dbContext.UpdateRange(entities);
          _dbContext.ShouldSaveChanges = true;
          return Task.CompletedTask;
      }
  
      public Task DeleteAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          _dbContext.Set<TEntity>().Where(predicate).ExecuteDeleteAsync(cancellationToken).ConfigureAwait(false);
          _dbContext.ShouldSaveChanges = true;
          return Task.CompletedTask;
      }
  
      public Task DeleteAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          _dbContext.Remove(entity);
          _dbContext.ShouldSaveChanges = true;
          return Task.CompletedTask;
      }
  
      public Task DeleteAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          _dbContext.RemoveRange(entities);
          _dbContext.ShouldSaveChanges = true;
          return Task.CompletedTask;
      }
  
      public Task<int> CountAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          return _dbContext.Set<TEntity>().CountAsync(predicate, cancellationToken);
      }
  
      public Task<TEntity?> SingleOrDefaultAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          return _dbContext.Set<TEntity>().SingleOrDefaultAsync(predicate, cancellationToken);
      }
  
      public Task<TEntity?> FirstOrDefaultAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          return _dbContext.Set<TEntity>().FirstOrDefaultAsync(predicate, cancellationToken);
      }
  
      public Task<bool> AnyAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity
      {
          return _dbContext.Set<TEntity>().AnyAsync(predicate, cancellationToken);
      }
  
      public async Task<List<TEntity>> SqlQueryAsync<TEntity>(string sql, params object[] parameters) where TEntity : class, IEntity
      {
          return await _dbContext.Set<TEntity>().FromSqlRaw(sql, parameters).ToListAsync();
      }
  
      public IQueryable<TEntity> Query<TEntity>(Expression<Func<TEntity, bool>>? predicate = null) where TEntity : class, IEntity
      {
          return predicate == null ? _dbContext.Set<TEntity>() : _dbContext.Set<TEntity>().Where(predicate);
      }
  
      public IQueryable<TEntity> QueryNoTracking<TEntity>(Expression<Func<TEntity, bool>>? predicate = null) where TEntity : class, IEntity
      {
          return predicate == null
              ? _dbContext.Set<TEntity>().AsNoTracking()
              : _dbContext.Set<TEntity>().AsNoTracking().Where(predicate);
      }
  
      public DatabaseFacade Database => _dbContext.Database;
      
      public async Task BatchInsertAsync<TEntity>(IList<TEntity> entities) where TEntity : class, IEntity
      {
          await _dbContext.BulkInsertAsync<TEntity>(entities).ConfigureAwait(false);
      }
  
      public async Task BatchUpdateAsync<TEntity>(IList<TEntity> entities) where TEntity : class, IEntity
      {
          await _dbContext.BulkUpdateAsync<TEntity>(entities).ConfigureAwait(false);
      }
  
      public async Task BatchDeleteAsync<TEntity>(Expression<Func<TEntity, bool>> predicate) where TEntity : class, IEntity
      {
          await _dbContext.Set<TEntity>().Where(predicate).ExecuteDeleteAsync().ConfigureAwait(false);
      }
  }
  ```

- 定义并注册ContectionString，并在DBContext中替换原有的手写字符串的连接

  ```c#
  public class ConnectionString : IConfigurationSetting
  {
      public ConnectionString(IConfiguration configuration)
      {
          Value = configuration.GetConnectionString("Default");
      }
  
      public string Value { get; set; }
  }
  
  private void RegisterSettings(ContainerBuilder builder)
  {
      var settingTypes = _assemblies.SelectMany(a => a.GetTypes())
          .Where(t => t.IsClass && typeof(IConfigurationSetting).IsAssignableFrom(t))
          .ToArray();
  
      builder.RegisterTypes(settingTypes).AsSelf().SingleInstance();
  }
  ```




## 加载相关数据

### 预加载

#### 基本用法

你可以使用 `Include` 方法指定要在查询结果中包含的相关数据。以下示例中，返回的博客（Blogs）将会包含其相关的帖子（Posts）属性。

```c#
using (var context = new BloggingContext())
{
    var blogs = context.Blogs
        .Include(blog => blog.Posts)
        .ToList();
}
```

Entity Framework Core 会自动修正上下文实例中以前加载的任何其他实体的导航属性。所以即使你没有显式地包含导航属性的数据，如果以前加载了部分或全部相关实体，属性仍可能会被填充。

#### 多重关系

可以在单个查询中包含来自多个关系的数据。

```c#
using (var context = new BloggingContext())
{
    var blogs = context.Blogs
        .Include(blog => blog.Posts)
        .Include(blog => blog.Owner)
        .ToList();
}
```

#### 多层级包含

使用 `ThenInclude` 方法可以通过关系向下钻取，包含多个层级的相关数据。例如，加载所有博客、相关的帖子及每个帖子的作者：

```c#
using (var context = new BloggingContext())
{
    var blogs = context.Blogs
        .Include(blog => blog.Posts)
        .ThenInclude(post => post.Author)
        .ToList();
}
```

你可以链式调用多个 `ThenInclude` 来继续包含进一步层级的相关数据。

```c#
using (var context = new BloggingContext())
{
    var blogs = context.Blogs
        .Include(blog => blog.Posts)
        .ThenInclude(post => post.Author)
        .ThenInclude(author => author.Photo)
        .ToList();
}
```

你可以在同一查询中结合多个层级和多个根的相关数据。

```c#
using (var context = new BloggingContext())
{
    var blogs = context.Blogs
        .Include(blog => blog.Posts)
        .ThenInclude(post => post.Author)
        .ThenInclude(author => author.Photo)
        .Include(blog => blog.Owner)
        .ThenInclude(owner => owner.Photo)
        .ToList();
}
```

#### 过滤包含

在应用 `Include` 以加载相关数据时，可以对包含的集合导航添加某些可枚举操作，如过滤和排序。

支持的操作有：`Where`、`OrderBy`、`OrderByDescending`、`ThenBy`、`ThenByDescending`、`Skip` 和 `Take`。

这些操作应应用于传递给 `Include` 方法的 lambda 表达式中的集合导航。

```c#
using (var context = new BloggingContext())
{
    var filteredBlogs = context.Blogs
        .Include(
            blog => blog.Posts
                .Where(post => post.BlogId == 1)
                .OrderByDescending(post => post.Title)
                .Take(5))
        .ToList();
}
```

#### 派生类型上的包含

你可以使用 `Include` 和 `ThenInclude` 来包含仅在派生类型上定义的导航数据。

例如，给定以下模型：

```c#
public class SchoolContext : DbContext
{
    public DbSet<Person> People { get; set; }
    public DbSet<School> Schools { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<School>().HasMany(s => s.Students).WithOne(s => s.School);
    }
}

public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
}

public class Student : Person
{
    public School School { get; set; }
}

public class School
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<Student> Students { get; set; }
}
```

可以使用多种模式来急加载所有学生的学校（School）导航属性：

- 使用类型转换

```c#
context.People.Include(person => ((Student)person).School).ToList();
```

- 使用 `as` 操作符

```c#
context.People.Include(person => (person as Student).School).ToList();
```

- 使用字符串参数的 `Include` 重载

```c#
context.People.Include("School").ToList();
```

#### 自动包含导航配置

你可以配置模型中的导航，每次从数据库加载实体时都包含该导航，使用 `AutoInclude` 方法。它的效果与在每个返回该实体类型结果的查询中指定 `Include` 相同。

```c#
modelBuilder.Entity<Theme>().Navigation(e => e.ColorScheme).AutoInclude();
```

这样配置后，运行如下查询将会加载所有主题的 `ColorScheme` 导航属性。

```c#
using (var context = new BloggingContext())
{
    var themes = context.Themes.ToList();
}
```

如果在某个查询中不想加载通过模型级别配置为自动包含的相关数据，可以使用 `IgnoreAutoIncludes` 方法。

```c#
using (var context = new BloggingContext())
{
    var themes = context.Themes.IgnoreAutoIncludes().ToList();
}
```

请注意，拥有类型的导航也会被默认配置为自动包含，使用 `IgnoreAutoIncludes` API 并不会阻止它们被包含，它们仍然会出现在查询结果中。

### 显式加载

#### 使用 DbContext.Entry(...) API 显式加载导航属性

可以通过 `DbContext.Entry(...)` API 显式加载导航属性。

```c#
using (var context = new BloggingContext())
{
    var blog = context.Blogs
        .Single(b => b.BlogId == 1);

    context.Entry(blog)
        .Collection(b => b.Posts)
        .Load();

    context.Entry(blog)
        .Reference(b => b.Owner)
        .Load();
}
```

#### 通过单独查询显式加载导航属性

还可以通过执行返回相关实体的单独查询来显式加载导航属性。如果启用更改跟踪，则当查询具体化实体时，EF Core 将自动设置新加载实体的导航属性以引用任何已加载的实体，并将已加载实体的导航属性设置为引用新加载的实体。

#### 查询相关实体

可以获得表示导航属性内容的 LINQ 查询。这允许你在查询上应用其他操作。例如，对相关实体应用聚合操作而不将它们加载到内存中。

```c#
using (var context = new BloggingContext())
{
    var blog = context.Blogs
        .Single(b => b.BlogId == 1);

    var postCount = context.Entry(blog)
        .Collection(b => b.Posts)
        .Query()
        .Count();
}
```

还可以过滤哪些相关实体加载到内存中。

```c#
using (var context = new BloggingContext())
{
    var blog = context.Blogs
        .Single(b => b.BlogId == 1);

    var goodPosts = context.Entry(blog)
        .Collection(b => b.Posts)
        .Query()
        .Where(p => p.Rating > 3)
        .ToList();
}
```

### 延迟加载

#### 使用代理延迟加载 (Lazy Loading with Proxies)

使用延迟加载的最简单方法是安装 `Microsoft.EntityFrameworkCore.Proxies` 包，并通过调用 `UseLazyLoadingProxies` 来启用它。例如：

```c#
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder
        .UseLazyLoadingProxies()
        .UseSqlServer(myConnectionString);
```

或者当使用 `AddDbContext` 时：

```c#
.AddDbContext<BloggingContext>(
    b => b.UseLazyLoadingProxies()
          .UseSqlServer(myConnectionString));
```

然后，EF Core 将为任何可重写的导航属性启用延迟加载 - 也就是说，它必须是 `virtual` 并且位于可继承的类上。例如，在以下实体中，`Post.Blog` 和 `Blog.Posts` 导航属性将被延迟加载。

```c#
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }

    public virtual ICollection<Post> Posts { get; set; }
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }

    public virtual Blog Blog { get; set; }
}
```

**警告**：延迟加载可能会导致不必要的额外数据库往返（所谓的 N+1 问题），应注意避免这种情况。有关更多详细信息，请参阅性能部分。

#### 无代理延迟加载 (Lazy Loading without Proxies)

无代理的延迟加载通过将 `ILazyLoader` 服务注入实体来工作，如实体类型构造函数中所述。例如：

```c#
public class Blog
{
    private ICollection<Post> _posts;

    public Blog()
    {
    }

    private Blog(ILazyLoader lazyLoader)
    {
        LazyLoader = lazyLoader;
    }

    private ILazyLoader LazyLoader { get; set; }

    public int Id { get; set; }
    public string Name { get; set; }

    public ICollection<Post> Posts
    {
        get => LazyLoader.Load(this, ref _posts);
        set => _posts = value;
    }
}

public class Post
{
    private Blog _blog;

    public Post()
    {
    }

    private Post(ILazyLoader lazyLoader)
    {
        LazyLoader = lazyLoader;
    }

    private ILazyLoader LazyLoader { get; set; }

    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }

    public Blog Blog
    {
        get => LazyLoader.Load(this, ref _blog);
        set => _blog = value;
    }
}
```

此方法不需要继承实体类型，也不要求导航属性是 `virtual`，并且允许使用 `new` 创建的实体实例在附加到上下文后进行延迟加载。但是，它需要对 `ILazyLoader` 服务的引用，该服务在 `Microsoft.EntityFrameworkCore.Abstractions` 包中定义。该包包含最少的类型集，因此依赖它的影响很小。但是，为了完全避免依赖实体类型中的任何 EF Core 包，可以将 `ILazyLoader.Load` 方法作为委托注入。例如：

```c#
public class Blog
{
    private ICollection<Post> _posts;

    public Blog()
    {
    }

    private Blog(Action<object, string> lazyLoader)
    {
        LazyLoader = lazyLoader;
    }

    private Action<object, string> LazyLoader { get; set; }

    public int Id { get; set; }
    public string Name { get; set; }

    public ICollection<Post> Posts
    {
        get => LazyLoader.Load(this, ref _posts);
        set => _posts = value;
    }
}

public class Post
{
    private Blog _blog;

    public Post()
    {
    }

    private Post(Action<object, string> lazyLoader)
    {
        LazyLoader = lazyLoader;
    }

    private Action<object, string> LazyLoader { get; set; }

    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }

    public Blog Blog
    {
        get => LazyLoader.Load(this, ref _blog);
        set => _blog = value;
    }
}
```

上面的代码使用 `Load` 扩展方法来使委托的使用更加简洁：

```c#
public static class PocoLoadingExtensions
{
    public static TRelated Load<TRelated>(
        this Action<object, string> loader,
        object entity,
        ref TRelated navigationField,
        [CallerMemberName] string navigationName = null)
        where TRelated : class
    {
        loader?.Invoke(entity, navigationName);

        return navigationField;
    }
}
```

**注意**：延迟加载委托的构造函数参数必须称为 `lazyLoader`。



## 跟踪查询与非跟踪查询

在 Entity Framework Core (EF Core) 中，跟踪行为控制 EF Core 是否在其更改跟踪器中保留实体实例的信息。如果一个实体被跟踪，那么在 `SaveChanges` 期间检测到的任何更改都会被持久化到数据库中。EF Core 还会在跟踪查询结果中的实体和更改跟踪器中的实体之间修正导航属性。

### 跟踪查询

默认情况下，返回实体类型的查询是跟踪查询。这意味着对实体实例的任何更改都会通过 `SaveChanges` 持久化。以下示例展示了对博客评级的更改在 `SaveChanges` 期间被检测到并持久化到数据库中：

```csharp
var blog = context.Blogs.SingleOrDefault(b => b.BlogId == 1);
blog.Rating = 5;
context.SaveChanges();
```

在返回跟踪查询的结果时，EF Core 会检查实体是否已经在上下文中。如果找到现有实体，则返回相同的实例，这可能比 No-Tracking 查询使用更少的内存并且更快。EF Core 不会用数据库值覆盖实体条目中的当前值和原始值。如果在上下文中未找到实体，EF Core 会创建一个新实体实例并将其附加到上下文。查询结果中不会包含任何已添加到上下文但尚未保存到数据库的实体。

### 非跟踪查询

非跟踪查询在只读场景中非常有用，因为无需设置更改跟踪信息，因此它们通常执行得更快。如果从数据库检索的实体不需要更新，那么应该使用 非跟踪 查询。可以将单个查询设置为 非跟踪。非跟踪 查询还会根据数据库中的内容返回结果，而忽略任何本地更改或添加的实体。

```csharp
var blogs = context.Blogs
    .AsNoTracking()
    .ToList();
```

可以在上下文实例级别更改默认跟踪行为：

```csharp
context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
var blogs = context.Blogs.ToList();
```

### 身份解析

由于跟踪查询使用更改跟踪器，EF Core 在跟踪查询中执行身份解析。当实例化一个实体时，如果实体已经被跟踪，EF Core 会从更改跟踪器返回相同的实体实例。如果结果包含多次出现的相同实体，则每次出现都会返回相同的实例。而 非跟踪 查询：

- 不使用更改跟踪器，也不执行身份解析。
- 返回一个新的实体实例，即使结果中包含多次相同的实体。

可以在同一个查询中组合跟踪和 非跟踪。例如，可以有一个 非跟踪 查询，但对结果进行身份解析。就像 `AsNoTracking` 查询操作符一样，还有一个 `AsNoTrackingWithIdentityResolution<TEntity>(IQueryable<TEntity>)` 操作符。当配置查询使用身份解析但不跟踪时，在生成查询结果时在后台使用一个独立的更改跟踪器，因此每个实例只会实例化一次。由于这个更改跟踪器与上下文中的不同，因此上下文不会跟踪这些结果。在查询完全枚举后，更改跟踪器会超出作用域并根据需要进行垃圾回收。

```csharp
var blogs = context.Blogs
    .AsNoTrackingWithIdentityResolution()
    .ToList();
```

### 配置默认跟踪行为

如果发现需要为许多查询更改跟踪行为，可以更改默认设置：

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=EFQuerying.Tracking;Trusted_Connection=True;ConnectRetryCount=0")
        .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
}
```

这样可以使所有查询默认不跟踪。仍然可以添加 `AsTracking` 使特定查询跟踪。

### 跟踪和自定义投影

即使查询结果类型不是实体类型，EF Core 仍然默认跟踪结果中包含的实体类型。在下面的查询中，返回一个匿名类型，但结果集中 `Blog` 的实例将被跟踪。

```csharp
var blog = context.Blogs
    .Select(
        b => new { Blog = b, PostCount = b.Posts.Count() });
```

如果结果集中包含来自 LINQ 组合的实体类型，EF Core 会跟踪它们。

```csharp
var blog = context.Blogs
    .Select(
        b => new { Blog = b, Post = b.Posts.OrderBy(p => p.Rating).LastOrDefault() });
```

如果结果集中不包含任何实体类型，则不进行跟踪。在下面的查询中，我们返回一个包含实体值但不包含实际实体类型实例的匿名类型。查询结果中没有被跟踪的实体。

```csharp
var blog = context.Blogs
    .Select(
        b => new { Id = b.BlogId, b.Url });
```

EF Core 支持在顶级投影中进行客户端评估。如果 EF Core 为客户端评估实例化实体实例，它将跟踪这些实体实例。以下示例中，我们将 `Blog` 实体传递给客户端方法 `StandardizeURL`，因此 EF Core 也会跟踪这些 `Blog` 实例。

```csharp
var blogs = context.Blogs
    .OrderByDescending(blog => blog.Rating)
    .Select(
        blog => new { Id = blog.BlogId, Url = StandardizeUrl(blog) })
    .ToList();

public static string StandardizeUrl(Blog blog)
{
    var url = blog.Url.ToLower();

    if (!url.StartsWith("http://"))
    {
        url = string.Concat("http://", url);
    }

    return url;
}
```

EF Core 不跟踪结果中包含的无键实体实例，但根据上述规则跟踪所有具有键的实体类型实例。



## 逻辑删除

在 EF Core 中，逻辑删除的实现主要有两种方案：

1. **通过自定义方法**：直接更新逻辑删除标识字段以标记删除状态。这种方法通常适用于单个实体的删除操作。
2. **在 `UnitOfWork` 中处理**：在调用 `SaveChanges` 之前，将实体的状态从 `Deleted` 修改为 `Modified`，并更新逻辑删除标识字段。这种方法可以实现全局的逻辑删除功能，更加统一和简洁。



## SQL 注入问题

防止 SQL 注入在 EF Core 中的主要方案包括：

1. **使用参数化查询**：通过使用参数来代替字符串拼接，从而防止 SQL 注入。
2. **使用 LINQ 表达式**：更推荐使用 LINQ，因为它将查询编译为表达式树，并生成参数化的 SQL 查询，从而提高安全性。



# DbContext

1. 项目中将DbContext生命周期注册为Scoped，即在同一个Scoped内只初始化一个实例，也就是一个Http请求会创建一个实例。
   - 在同一个请求中，对DbContext的操作都会获得相同实例
   - 请求完成后，DbContext自动释放。
   - DbContext本质上不是线程安全的，在scope生命周期中，每个请求都有自己的DbContext实例，避免多线程下的并发问题。
   - 同一个请求中的DbContext是同一个实例的话，可以将所有数据库操作都共享同一个事务。可以有效控制好事务。
   - 如果执行后台任务时间过长，则需要手动管理生命周期，避免DbContext被释放。

2. 其他生命周期适用场景：
   - 单例（Singleton）：需要共享状态的全局服务，或一些不频繁改变的配置数据
   - 瞬时（Transient）：短期操作或任务，如临时的数据库查询或操作



## OnModelCreating

```c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        var allTypes = Assembly.GetExecutingAssembly().GetTypes()
            .Where(t => t.IsClass && typeof(IEntity).IsAssignableFrom(t))
            .ToList();

        foreach (var type in allTypes.Where(type => modelBuilder.Model.FindEntityType(type) == null))
        {
            var entityTypeBuilder = modelBuilder.Entity(type);
            
            if (type.GetCustomAttributes(typeof(TableAttribute), true).FirstOrDefault() is TableAttribute tableAttribute)
            {
                entityTypeBuilder.ToTable(tableAttribute.Name);
            }

            foreach (var property in type.GetProperties())
            {
                if (property.GetCustomAttributes(typeof(ColumnAttribute), true).FirstOrDefault() is ColumnAttribute columnAttribute)
                {
                    entityTypeBuilder.Property(property.Name).HasColumnName(columnAttribute.Name);
                }
            }
            
        }
    }
```



- `OnModelCreating` 的另一种映射方法，通过使用反射获取实现 `IEntity` 接口的所有类。
- 使用`modelBuilder.Entity(type)`将实体添加到modelBuilder中。
- 利用特性 `TableAttribute` 和 `ColumnAttribute` 进行自定义表和列的映射配置。



## DbContext生存期

- 创建DbContext实例
- 根据上下文追踪实体实例。实体将在以下情况被跟踪：
  - 正在从查询返回
  - 正在从添加或附加到上下文
- 根据需要对所跟踪的实体进行更改以实现业务规则
- 调用SaveChanges或SaveChangesAsync。EF Core检测所做修改，并将其写入数据库。
- 释放DbContext实例。



## 补充

- 关于在Task中使用IServiceProvider问题

  ​	如果在没有使用await情况下，使用Task.Run来执行代码，且在内部使用IServiceProvider实例。可能会导致Task方法体还没执行完毕就释放了该实例。因为每次请求单独创建的IServiceProvider生命周期为Scope，且Task.Run内的逻辑何时执行由系统CPU调度本身决定。

  - 如何解决该问题？
    - 使用`await`
    
    - 使用 `IServiceScopeFactory` 创建 `IServiceScope` 实例，因为 `IServiceScopeFactory` 是从根容器中获取的，其创建的子容器（`IServiceScope`）不会因为请求的结束而被自动释放。
    
      

- EF Core中的ExecuteDeleteAsync是进行批量删除的方法，且该方法执行会自动调用SaveChanges进行保存

- 在使用异步方法时要确保使用了await和async关键字，如果没有则会导致执行sql失败，并且没有控制台不会抛出异常的日志。

  **原因：**在 C# 中，异步方法通常返回 `Task` 或 `Task<T>`。如果在调用这些方法时没有使用 `await`，这些操作会以非阻塞的方式开始执行，但不会等待其完成。这样可能导致程序继续执行后续代码，而没有等待前面的数据库操作完成，这在某些情况下可能会导致未完成的数据库事务或未保存的更改。
