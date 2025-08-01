# [如何根据项目需求自定义 ASP.NET Core Identity 与 EF Core](https://antondevtips.com/blog/how-to-customize-aspnet-core-identity-with-efcore-for-your-project-needs)

安全和身份验证是任何应用程序中最重要的方面之一。理解和应用经过验证的工具对于防止未经授权的访问和数据泄露等常见漏洞至关重要。

ASP.NET Core Identity 为开发者提供了一种强大的方式来管理用户、角色、声明，并为 Web 应用执行用户认证。Identity 提供了开箱即用的解决方案和 API。但通常你需要进行调整以适应你的特定需求或要求。

今天我想逐步展示如何实际地定制 ASP.NET Identity 的方法。

我们将探讨：

- 如何将内置的 Identity 表适配到你的数据库架构
- 如何使用 JWT 令牌注册和登录用户
- 如何使用 Identity 更新用户角色和声明
- 如何使用 Identity 初始化角色和声明。

让我们开始吧！

## 开始使用 ASP.NET Identity

ASP.NET Core Identity 是一组工具，用于为 ASP.NET Core 应用程序添加登录功能。它处理诸如创建新用户、哈希密码、验证用户凭据以及管理角色或声明等任务。

将以下包添加到你的项目中，以开始使用 ASP.NET Core Identity：

```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

以下是你如何配置默认身份验证的方法：

```csharp
builder.Services.AddDefaultIdentity<IdentityUser>(options => {})
    .AddEntityFrameworkStores<ApplicationDbContext>();

builder.Services.Configure<IdentityOptions>(options =>
{
    // 密码设置
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequireUppercase = true;
    options.Password.RequiredLength = 6;
    options.Password.RequiredUniqueChars = 1;

    // 锁定设置
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;

    // 用户设置
    options.User.AllowedUserNameCharacters =
    "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-._@+";
    options.User.RequireUniqueEmail = false;
});
```

以下是 Identity 提供的可用实体列表：

- User  用户
- Role  角色
- Claim  声明
- UserClaim  用户声明
- UserRole  用户角色
- RoleClaim  角色声明
- UserToken  用户令牌
- UserLogin  用户登录

然而，在许多应用程序中，你可能需要进行以下自定义：

- 将自定义字段添加到 **User** 实体
- 将自定义字段添加到 **Role** 实体
- 在使用 **IdentityDbContext** 时引用用户和角色子实体
- 使用 JWT 令牌而不是 Bearer 令牌（是的，当使用现成的 Web API 时，ASP.NET Core Identity 会返回 Bearer 令牌）

让我们探讨如何为 Identity 使用 EF Core 自定义数据库架构。

## 自定义 ASP.NET Identity 与 EF Core

ASP.NET Core Identity 的一个伟大特性是，你可以自定义实体及其对应的数据库表。

你可以创建自己的类并从 **IdentityUser** 继承，以添加新字段（FullName 和 JobTitle）：

```csharp
public class User : IdentityUser
{
    public string FullName { get; set; }
    
    public string JobTitle { get; set; }
}
```

接下来，你需要从 **IdentityDbContext** 继承并指定你的自定义用户类的类型：

```csharp
public class BooksDbContext : IdentityDbContext<User>
{
}
```

你可以在其他实体中像往常一样引用用户实体：

```csharp
public class Author
{
    public required Guid Id { get; set; }

    public required string Name { get; set; }

    public List<Book> Books { get; set; } = [];

    public string? UserId { get; set; }

    public User? User { get; set; }
}

public class AuthorConfiguration : IEntityTypeConfiguration<Author>
{
    public void Configure(EntityTypeBuilder<Author> builder)
    {
        builder.ToTable("authors");
        builder.HasKey(x => x.Id);

        // ...

        builder.HasOne(x => x.User)
            .WithOne()
            .HasForeignKey<Author>(x => x.UserId);
    }
}
```

让我们进一步扩展 **User** 实体，以便能够导航到用户的声明、角色、登录和令牌：

```csharp
public class User : IdentityUser
{
    public ICollection<UserClaim> Claims { get; set; }

    public ICollection<UserRole> UserRoles { get; set; }

    public ICollection<UserLogin> UserLogins { get; set; }

    public ICollection<UserToken> UserTokens { get; set; }
}
```

我们需要重写 **User** 实体的映射以指定关系：

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        // 每个用户可以有多个 UserClaims
        builder.HasMany(e => e.Claims)
            .WithOne(e => e.User)
            .HasForeignKey(uc => uc.UserId)
            .IsRequired();

        // 每个用户可以有多个 UserLogins
        builder.HasMany(e => e.UserLogins)
            .WithOne(e => e.User)
            .HasForeignKey(ul => ul.UserId)
            .IsRequired();

        // 每个用户可以有多个 UserTokens
        builder.HasMany(e => e.UserTokens)
            .WithOne(e => e.User)
            .HasForeignKey(ut => ut.UserId)
            .IsRequired();

        // 每个用户可以有多个条目在 UserRole 连接表中
        builder.HasMany(e => e.UserRoles)
            .WithOne(e => e.User)
            .HasForeignKey(ur => ur.UserId)
            .IsRequired();
    }
}
```

如果你想要拥有所有 Identity 实体之间的关系，你还需要扩展其他类：

```csharp
public class Role : IdentityRole
{
    public ICollection<UserRole> UserRoles { get; set; }
    public ICollection<RoleClaim> RoleClaims { get; set; }
}

public class RoleClaim : IdentityRoleClaim<string>
{
    public Role Role { get; set; }
}

public class UserRole : IdentityUserRole<string>
{
    public User User { get; set; }
    public Role Role { get; set; }
}

public class UserClaim : IdentityUserClaim<string>
{
    public User User { get; set; }
}

public class UserLogin : IdentityUserLogin<string>
{
    public User User { get; set; }
}

public class UserToken : IdentityUserToken<string>
{
    public User User { get; set; }
}
```

以下是实体的映射：

```csharp
public class RoleConfiguration : IEntityTypeConfiguration<Role>
{
    public void Configure(EntityTypeBuilder<Role> builder)
    {
        builder.ToTable("roles");

        // 每个角色可以在 UserRole 连接表中有多个条目
        builder.HasMany(e => e.UserRoles)
            .WithOne(e => e.Role)
            .HasForeignKey(ur => ur.RoleId)
            .IsRequired();

        // 每个角色可以有多个关联的 RoleClaims
        builder.HasMany(e => e.RoleClaims)
            .WithOne(e => e.Role)
            .HasForeignKey(rc => rc.RoleId)
            .IsRequired();
    }
}

public class RoleClaimConfiguration : IEntityTypeConfiguration<RoleClaim>
{
    public void Configure(EntityTypeBuilder<RoleClaim> builder)
    {
        builder.ToTable("role_claims");
    }
}

public class UserRoleConfiguration : IEntityTypeConfiguration<UserRole>
{
    public void Configure(EntityTypeBuilder<UserRole> builder)
    {
        builder.HasKey(x => new { x.UserId, x.RoleId });
    }
}

public class UserClaimConfiguration : IEntityTypeConfiguration<UserClaim>
{
    public void Configure(EntityTypeBuilder<UserClaim> builder)
    {
        builder.ToTable("user_claims");
    }
}

public class UserLoginConfiguration : IEntityTypeConfiguration<UserLogin>
{
    public void Configure(EntityTypeBuilder<UserLogin> builder)
    {
        builder.ToTable("user_logins");
    }
}

public class UserTokenConfiguration : IEntityTypeConfiguration<UserToken>
{
    public void Configure(EntityTypeBuilder<UserToken> builder)
    {
        builder.ToTable("user_tokens");
    }
}
```

看起来可能很多，但你需要一次性完成这些，然后在每个项目中使用它。

以下是 BooksDbContext 的变化方式：

```csharp
public class BooksDbContext : IdentityDbContext<User, Role, string,
    UserClaim, UserRole, UserLogin,
    RoleClaim, UserToken>
{
}
```

最后，你需要在依赖注入中注册 Identity：

```csharp
services.AddDbContext<BooksDbContext>((provider, options) =>
{
    options
        .UseNpgsql(connectionString, npgsqlOptions =>
        {
            npgsqlOptions.MigrationsHistoryTable(DatabaseConsts.MigrationTableName,
                DatabaseConsts.Schema);
        })
        .UseSnakeCaseNamingConvention();
});

services
    .AddIdentity<User, Role>(options =>
    {
        options.Password.RequireDigit = true;
        options.Password.RequireLowercase = true;
        options.Password.RequireUppercase = true;
        options.Password.RequireNonAlphanumeric = true;
        options.Password.RequiredLength = 8;
    })
    .AddEntityFrameworkStores<BooksDbContext>()
    .AddSignInManager()
    .AddDefaultTokenProviders();
```

默认情况下，数据库中的所有表和列都将使用 PascalCase 命名。如果你更喜欢一致且自动的命名模式，请考虑使用 EFCore.NamingConventions 包：

```bash
dotnet add package EFCore.NamingConventions
```

一旦安装完成，你只需将命名约定支持插入到你的 DbContext 配置中。在我上面的 DbContext 注册中，我使用了 UseSnakeCaseNamingConvention() 用于我的 Postgres 数据库。因此，表名和列名将看起来像： user_claims 、 user_id 等。

## 使用 Identity 注册用户

现在 Identity 已经配置好了，你可以暴露一个端点让新用户注册。以下是一个最小 API 示例：

```csharp
public record RegisterUserRequest(string Email, string Password);

app.MapPost("/api/register", async (
    [FromBody] RegisterUserRequest request,
    UserManager<User> userManager) =>
{
    var existingUser = await userManager.FindByEmailAsync(request.Email);
    if (existingUser != null)
    {
        return Results.BadRequest("User already exists.");
    }

    var user = new User
    {
        UserName = request.Email,
        Email = request.Email
    };

    var result = await userManager.CreateAsync(user, request.Password);
    if (!result.Succeeded)
    {
        return Results.BadRequest(result.Errors);
    }
    
    result = await userManager.AddToRoleAsync(user, "DefaultRole");
    
    if (!result.Succeeded)
    {
        return Results.BadRequest(result.Errors);
    }

    var response = new UserResponse(user.Id, user.Email);
    return Results.Created($"/api/users/{user.Id}", response);
});
```

你可以使用 **UserManager\<User\>** 类来管理用户。

注册涉及以下步骤：

1. 你需要通过调用 **FindByEmailAsync** 方法来检查用户是否已存在。
2. 如果用户不存在，你可以通过调用 **CreateAsync** 方法来创建一个新用户。
3. 你可以通过调用 **AddToRoleAsync** 方法将用户添加到角色中。
4. 在注册过程中出现错误时 - 你可以返回一个带有来自 Identity 的错误信息的 **BadRequest** 响应。

## 如何使用 Identity 登录用户

一旦用户创建成功，他们就可以进行身份验证。以下是如何使用 Identity 实现身份验证的方法：

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;

public record LoginUserRequest(string Email, string Password);

app.MapPost("/api/login", async (
    [FromBody] LoginUserRequest request,
    IOptions<AuthConfiguration> authOptions,
    UserManager<User> userManager,
    SignInManager<User> signInManager,
    RoleManager<Role> roleManager) =>
{
    var user = await userManager.FindByEmailAsync(request.Email);
    if (user is null)
    {
        return Results.NotFound("User not found");
    }

    var result = await signInManager.CheckPasswordSignInAsync(user, request.Password, false);
    if (!result.Succeeded)
    {
        return Results.Unauthorized();
    }

    var roles = await userManager.GetRolesAsync(user);
    var userRole = roles.FirstOrDefault() ?? "user";

    var role = await roleManager.FindByNameAsync(userRole);
    var roleClaims = role is not null ? await roleManager.GetClaimsAsync(role) : [];

    var token = GenerateJwtToken(user, authOptions.Value, userRole, roleClaims);
    return Results.Ok(new { Token = token });
});
```

认证过程包括以下步骤：

- 你需要通过调用 **FindByEmailAsync** 方法来检查用户是否存在。
- 你可以通过调用 **CheckPasswordSignInAsync** 方法从 **SignInManager\<User\>** 检查用户的密码。
- 你可以通过调用 **GetRolesAsync** 方法从 **UserManager\<User\>** 获取用户的角色。
- 你可以通过调用 **GetClaimsAsync** 方法从 **RoleManager\<Role\>** 获取角色的声明。
- 在登录过程中出现错误时，你可以返回一个包含来自 Identity 的错误信息的 **BadRequest** 响应。

你可以在成功登录后发行一个 JWT 令牌：

```csharp
private static string GenerateJwtToken(User user,
    AuthConfiguration authConfiguration,
    string userRole,
    IList<Claim> roleClaims)
{
    var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(authConfiguration.Key));
    var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);

    List<Claim> claims = [
        new(JwtRegisteredClaimNames.Sub, user.Email!),
        new("userid", user.Id),
        new("role", userRole)
    ];

    foreach (var roleClaim in roleClaims)
    {
        claims.Add(new Claim(roleClaim.Type, roleClaim.Value));
    }

    var token = new JwtSecurityToken(
        issuer: authConfiguration.Issuer,
        audience: authConfiguration.Audience,
        claims: claims,
        expires: DateTime.Now.AddMinutes(30),
        signingCredentials: credentials
    );

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

我的应用程序中的每个角色都有一组声明，我将这些声明添加到 JWT 令牌中。以下是发行的 JWT 令牌可能的样子：

```json
{
  "sub": "admin@test.com",
  "userid": "dc233fac-bace-4719-9a4f-853e199300d5",
  "role": "Admin",
  "users:create": "true",
  "users:update": "true",
  "users:delete": "true",
  "books:create": "true",
  "books:update": "true",
  "books:delete": "true",
  "exp": 1739481834,
  "iss": "DevTips",
  "aud": "DevTips"
}
```

你可以使用声明来限制对端点的访问，例如：

```csharp
app.MapPost("/api/books", Handle)
        .RequireAuthorization("books:create");
        
    app.MapDelete("/api/books/{id}", Handle)
        .RequireAuthorization("books:delete");
        
    app.MapPost("/api/users", Handle)
        .RequireAuthorization("users:create");
        
    app.MapDelete("/api/users/{id}", Handle)
        .RequireAuthorization("users:delete");
```

## 如何填充身份数据：初始化角色和声明

当你希望设置一个带有默认角色和声明的应用程序时，种子数据非常有用。常见的方法是在应用程序启动时一次性填充数据：

```csharp
var app = builder.Build();

// 注册中间件

// 创建并填充数据库
using (var scope = app.Services.CreateScope())
{
    var dbContext = scope.ServiceProvider.GetRequiredService<BooksDbContext>();
    var userManager = scope.ServiceProvider.GetRequiredService<UserManager<User>>();
    var roleManager = scope.ServiceProvider.GetRequiredService<RoleManager<Role>>();

    await DatabaseSeedService.SeedAsync(dbContext, userManager, roleManager);
}

await app.RunAsync();
```

```csharp
public static class DatabaseSeedService
{
    public static async Task SeedAsync(BooksDbContext dbContext, UserManager<User> userManager,
        RoleManager<Role> roleManager)
    {
        await dbContext.Database.MigrateAsync();

        if (await dbContext.Users.AnyAsync())
        {
            return;
        }

        // 填充角色和声明

        await dbContext.SaveChangesAsync();
    }
}
```

你可以使用 RoleManager\<Role\> 来管理角色及其声明：

```csharp
var adminRole = new Role { Name = "Admin" };
var authorRole = new Role { Name = "Author" };

var result = await roleManager.CreateAsync(adminRole);
result = await roleManager.CreateAsync(authorRole);

result = await roleManager.AddClaimAsync(adminRole, new Claim("users:create", "true"));
result = await roleManager.AddClaimAsync(adminRole, new Claim("users:update", "true"));
result = await roleManager.AddClaimAsync(adminRole, new Claim("users:delete", "true"));

result = await roleManager.AddClaimAsync(adminRole, new Claim("books:create", "true"));
result = await roleManager.AddClaimAsync(adminRole, new Claim("books:update", "true"));
result = await roleManager.AddClaimAsync(adminRole, new Claim("books:delete", "true"));

result = await roleManager.AddClaimAsync(authorRole, new Claim("books:create", "true"));
result = await roleManager.AddClaimAsync(authorRole, new Claim("books:update", "true"));
result = await roleManager.AddClaimAsync(authorRole, new Claim("books:delete", "true"));
```

以下是你如何在应用程序中创建默认用户的方法：

```csharp
var adminUser = new User
{
    Id = Guid.NewGuid().ToString(),
    Email = "admin@test.com",
    UserName = "admin@test.com"
};

result = await userManager.CreateAsync(adminUser, "Test1234!");
result = await userManager.AddToRoleAsync(adminUser, "Admin");
```

首次成功登录后，更改默认密码非常重要。

今天就到这里。期待与你再次相见。
