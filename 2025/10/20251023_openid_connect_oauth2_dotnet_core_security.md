# [为什么 OpenID Connect 并不复杂](https://medium.com/c-sharp-programming/openid-connect-oauth2-dotnet-core-security-b9ab4f2647dd)

> 在 ASP.NET Core 中学习 OpenID Connect 和 OAuth 2.0。设置、JWT、安全最佳实践和故障排除详解

## 引言

OAuth 2.0 和 OpenID Connect 是现代 Web 身份验证和授权的基础协议。本指南全面介绍了如何在 ASP.NET Core 应用程序中实现这些协议。

## 你将学到什么

- OAuth 2.0 授权流程
- OpenID Connect 认证层
- ASP.NET Core 认证中间件
- 令牌处理和验证
- 安全最佳实践

## 理解 OAuth 2.0

### 什么是 OAuth 2.0？

OAuth 2.0 是一个授权框架，它使应用程序能够获得对用户账户的有限访问权限。它通过将用户身份验证委托给托管用户账户的服务，并授权第三方应用程序访问用户账户来工作。

### 关键组件

- 资源所有者：授权应用程序访问其账户的用户
- 客户端：想要访问用户账户的应用程序
- 资源服务器：托管受保护资源的服务器
- 授权服务器：验证用户身份并颁发访问令牌的服务器

### 授权码流程

授权码流程是 Web 应用程序最安全的 OAuth 2.0 流程：

1. 授权请求：客户端将用户重定向到授权服务器
2. 用户认证：用户通过授权服务器进行身份验证
3. 授权授予：授权服务器重定向并返回授权码
4. 令牌请求：客户端使用授权码向授权服务器请求访问令牌
5. 访问受保护资源：客户端使用访问令牌访问受保护的资源

```csharp
// ASP.NET Core OAuth 2.0 Configuration
services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = "OAuth2";
})
.AddCookie()
.AddOAuth("OAuth2", options =>
{
    options.ClientId = "your-client-id";
    options.ClientSecret = "your-client-secret";
    options.AuthorizationEndpoint = "https://provider.com/oauth/authorize";
    options.TokenEndpoint = "https://provider.com/oauth/token";
});
```

## 理解 OpenID Connect

### 什么是 OpenID Connect？

OpenID Connect (OIDC) 是构建在 OAuth 2.0 之上的身份层。虽然 OAuth 2.0 是为授权设计的，但 OpenID Connect 添加了身份验证功能，允许客户端验证用户的身份。

### 与 OAuth 2.0 的主要区别

OAuth 2.0OpenID Connect 授权框架身份验证协议访问令牌 ID 令牌 + 访问令牌资源访问用户身份验证没有标准用户信息标准化用户声明

### ID 令牌

OpenID Connect 引入了 ID 令牌，这些是包含用户身份信息的 JSON Web 令牌（JWT）：

```json
{
  "sub": "248289761001",
  "name": "Jane Doe",
  "email": "jane.doe@example.com",
  "iat": 1516239022,
  "exp": 1516242622,
  "aud": "your-client-id",
  "iss": "https://your-provider.com"
}
```

### 标准范围

- openid ：表示 OIDC 请求的必需范围
- profile : 访问用户的个人资料信息
- email : 访问用户的电子邮件地址
- address : 访问用户的地址信息
- phone : 访问用户的电话号码

### ASP.NET Core 中的实现

基本配置

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(options =>
    {
        options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
    })
    .AddCookie(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddOpenIdConnect(OpenIdConnectDefaults.AuthenticationScheme, options =>
    {
        options.Authority = "https://your-identity-provider.com";
        options.ClientId = "your-client-id";
        options.ClientSecret = "your-client-secret";
        options.ResponseType = "code";
        options.SaveTokens = true;
        
        options.Scope.Clear();
        options.Scope.Add("openid");
        options.Scope.Add("profile");
        options.Scope.Add("email");
    });
}
```

中间件配置

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseAuthentication();
    app.UseAuthorization();
    
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

身份验证控制器

```csharp
[ApiController]
[Route("[controller]")]
public class AuthController : ControllerBase
{
    [HttpGet("login")]
    public IActionResult Login(string returnUrl = "/")
    {
        return Challenge(new AuthenticationProperties
        {
            RedirectUri = returnUrl
        });
    }

    [HttpGet("logout")]
    public async Task<IActionResult> Logout()
    {
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
        await HttpContext.SignOutAsync(OpenIdConnectDefaults.AuthenticationScheme);
        return Redirect("/");
    }
}
```

访问用户信息

```csharp
[Authorize]
public class ProfileController : ControllerBase
{
    [HttpGet]
    public IActionResult GetProfile()
    {
        var userId = User.FindFirst("sub")?.Value;
        var userName = User.FindFirst("name")?.Value;
        var email = User.FindFirst("email")?.Value;
        
        return Ok(new
        {
            Id = userId,
            Name = userName,
            Email = email,
            Claims = User.Claims.Select(c => new { c.Type, c.Value })
        });
    }
}
```

令牌管理

```csharp
[Authorize]
public class TokenController : ControllerBase
{
    [HttpGet("access-token")]
    public async Task<IActionResult> GetAccessToken()
    {
        var accessToken = await HttpContext.GetTokenAsync("access_token");
        var refreshToken = await HttpContext.GetTokenAsync("refresh_token");
        var idToken = await HttpContext.GetTokenAsync("id_token");
        
        return Ok(new
        {
            AccessToken = accessToken,
            RefreshToken = refreshToken,
            IdToken = idToken
        });
    }
}
```

## 安全最佳实践

### 全面使用 HTTPS

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddHttpsRedirection(options =>
    {
        options.RedirectStatusCode = StatusCodes.Status307TemporaryRedirect;
        options.HttpsPort = 443;
    });
}
```

### 正确验证令牌

```csharp
services.AddOpenIdConnect(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ClockSkew = TimeSpan.FromMinutes(5)
    };
});
```

### 实现 PKCE（代码交换密钥证明）

```csharp
services.AddOpenIdConnect(options =>
{
    options.UsePkce = true; 
});
```

### 配置安全 Cookie

```csharp
services.AddCookie(options =>
{
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.ExpireTimeSpan = TimeSpan.FromHours(1);
    options.SlidingExpiration = true;
});
```

### 处理令牌刷新

```csharp
services.AddOpenIdConnect(options =>
{
    options.Events = new OpenIdConnectEvents
    {
        OnTokenValidated = async context =>
        {
            // 将令牌妥善保存
            var tokens = context.Properties.GetTokens();
            // 实现令牌刷新逻辑
        }
    };
});
```

## 高级场景

### 自定义声明转换

```csharp
public class CustomClaimsTransformation : IClaimsTransformation
{
    public Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        var identity = (ClaimsIdentity)principal.Identity;
        
        // 添加自定义声明
        if (principal.HasClaim("email", "admin@company.com"))
        {
            identity.AddClaim(new Claim("role", "Administrator"));
        }
        
        return Task.FromResult(principal);
    }
}

// 在服务中注册
services.AddTransient<IClaimsTransformation, CustomClaimsTransformation>();
```

### 多个身份提供程序

```csharp
services.AddAuthentication()
    .AddOpenIdConnect("Google", options =>
    {
        options.Authority = "https://accounts.google.com";
        options.ClientId = "google-client-id";
        options.ClientSecret = "google-client-secret";
    })
    .AddOpenIdConnect("Microsoft", options =>
    {
        options.Authority = "https://login.microsoftonline.com/common/v2.0";
        options.ClientId = "microsoft-client-id";
        options.ClientSecret = "microsoft-client-secret";
    });
```

### 使用 JWT Bearer 保护 API

```csharp
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://your-identity-server.com";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = false
        };
    });

services.AddAuthorization(options =>
{
    options.AddPolicy("ApiScope", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("scope", "api1");
    });
});
```

## 故障排除技巧

### 常见问题与解决方案

重定向 URI 不匹配

问题： redirect_uri_mismatch 错误 解决方案：确保客户端和服务器配置中的重定向 URI 完全匹配

```csharp
options.CallbackPath = "/signin-oidc";
```

令牌验证失败

问题：令牌签名验证失败 解决方案：验证颁发者和受众配置

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidIssuer = "https://your-identity-server.com",
    ValidAudience = "your-client-id"
};
```

CORS 问题

问题：跨域请求被阻止 解决方案：正确配置 CORS

```csharp
services.AddCors(options =>
{
    options.AddPolicy("AllowSpecificOrigin", builder =>
    {
        builder.WithOrigins("https://your-client-app.com")
               .AllowAnyMethod()
               .AllowAnyHeader()
               .AllowCredentials();
    });
});
```

Cookie 大小限制

问题：Cookie 对于头部来说太大 解决方案：使用会话存储或外部令牌存储

```csharp
services.AddSession();
services.AddOpenIdConnect(options =>
{
    options.SaveTokens = false; // 不将令牌保存在 cookie 中
    options.Events = new OpenIdConnectEvents
    {
        OnTokenValidated = context =>
        {
            // 将令牌存储在会话中
            context.HttpContext.Session.SetString("access_token", 
                context.TokenEndpointResponse.AccessToken);
            return Task.CompletedTask;
        }
    };
});
```

## 调试技巧

启用详细日志：

```csharp
builder.Logging.AddFilter("Microsoft.AspNetCore.Authentication", LogLevel.Debug);
```

检查声明：

```csharp
[HttpGet("debug")]
public IActionResult Debug()
{
    return Ok(User.Claims.Select(c => new { c.Type, c.Value }));
}
```

验证配置：

```csharp
services.PostConfigure<OpenIdConnectOptions>(OpenIdConnectDefaults.AuthenticationScheme, options =>
{
    // 校验配置
    if (string.IsNullOrEmpty(options.ClientId))
        throw new InvalidOperationException("ClientId is required");
});
```

## 结论

OAuth 2.0 和 OpenID Connect 为现代 Web 应用程序中的身份验证和授权提供了强大、标准化的方法。ASP.NET Core 的内置支持使实现变得简单，同时保持了安全最佳实践。

关键要点：

- 使用 OpenID Connect 进行身份验证，使用 OAuth 2.0 进行授权
- 在生产环境中始终使用 HTTPS
- 实现适当的令牌验证和刷新逻辑
- 遵循 cookie 和会话管理的安全最佳实践
- 使用不同的身份提供者进行彻底测试

这一基础将使你能够构建安全、可扩展的身份验证系统，这些系统能与各种身份提供者集成，并有效保护你的应用程序资源。

今天就到这里。希望这些内容对你有帮助。

PS. 点击下方【阅读原文】获取源代码
