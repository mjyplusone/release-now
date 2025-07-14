Nest 中的**守卫**（**Guard**）用于在请求到达控制器之前执行一些逻辑，以决定请求是否交给路由处理程序进行处理。在业务逻辑前统一检查的逻辑，都可以抽象成守卫，比如常见的权限控制和身份验证等场景。

# 创建

## nest cli 创建守卫

可以通过 nest cli 快速创建守卫：

```
nest g guard user
```

执行这行命令后，在 src/user 文件夹中创建 user.guard.ts 文件：

```ts
import { CanActivate, ExecutionContext, Injectable } from "@nestjs/common";
import { Observable } from "rxjs";

@Injectable()
export class UserGuard implements CanActivate {
  canActivate(
    context: ExecutionContext
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```

可以看到守卫是具有 `@Injectable()` 装饰器的类，即声明这个类被 Nest IoC 容器接管，可以通过 `constructor` 注入依赖。它需要实现 `CanActive` 接口，这个接口有一个 `canActive` 方法。

## 实现 canActive 方法

### 参数

`canActive` 方法有一个 context 参数，它是一个 `ExecutionContext` 对象。`ExecutionContext` 基于 `ArgumentsHost` 对象（可用于获取请求响应对象，[异常过滤器 `catch` 方法的参数](https://juejin.cn/post/7487815126302457856#heading-13)中有用到）扩展了两个方法：

```ts
export interface ExecutionContext extends ArgumentsHost {
  getClass<T>(): Type<T>;
  getHandler(): Function;
}
```

- `getClass()`：返回要调用的请求处理程序所属的控制器类。
- `getHandler()`：返回要调用的请求处理程序的引用。

比如：一个请求，在 `UserController` 中绑定 `create()` 方法，`getHandler()` 返回 `create()` 方法，`getClass()` 返回 `UserController` 类。

可以使用这两个方法，配合 `Reflector` 类，读取附加在路由方法或控制器上的元数据。这个后面我们会给出具体例子。

### 返回

`canActive` 方法可以同步或异步的返回响应，即返回布尔值或 `Promise` 或 `Observable` 流。如果返回 `true` 则进入请求处理程序，返回 `false` 则框架抛出 `ForbiddenException` 异常。如果想返回不同的错误响应，也可以抛出自己声明的异常。

```ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from "@nestjs/common";
import { Observable } from "rxjs";

@Injectable()
export class UserGuard implements CanActivate {
  canActivate(
    context: ExecutionContext
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.body.user;

    if (user) {
      return true;
    }

    throw new UnauthorizedException("unauth");
  }
}
```

# 使用

创建完守卫后，我们需要将其绑定使用。守卫的绑定范围有：路由方法范围、控制器范围、全局范围。其中路由方法范围和控制器范围都使用  `@UseGuards()`  装饰器进行绑定。

## @UseGuards 装饰器

- 可以传入实例；也可以直接传入类，将实例化的过程交给 Nest 框架进行。更推荐使用类的方式，这样可以在多个模块中复用同一个实例。

```ts
@Controller("user")
@UseGuards(RolesGuard)
export class UserController {}

@Controller("user")
@UseGuards(new RolesGuard())
export class UserController {}
```

- 可以接受单个参数或逗号分隔的参数列表。

```ts
@Controller("user")
@UseGuards(RolesGuard, AuthGuard)
export class UserController {}
```

## 控制器守卫

```ts
@Controller("user")
@UseGuards(RolesGuard)
export class UserController {}
```

## 路由方法守卫

```ts
@Controller("user")
export class UsersController {
  @Get("private")
  @UseGuards(RolesGuard)
  getPrivateData() {}
}
```

## 全局守卫

通过 `app.useGlobalGuards` 绑定

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

## 执行顺序

- 当 `@UseGuards` 接受多个守卫时，执行顺序由输入顺序而定，其中任何一个守卫返回 `false` ，后面的守卫都会终止执行。

- 不同绑定范围的守卫的执行顺序是：全局 -> 控制器 -> 方法。

# 动态守卫

动态守卫指的是可以在运行时根据参数动态创建的守卫。这种守卫通常使用工厂函数或者类的静态方法来创建。

## 创建

创建动态守卫类，`constructor` 接受参数。

```ts
import { CanActivate, ExecutionContext, Injectable } from "@nestjs/common";

@Injectable()
export class DynamicAuthGuard implements CanActivate {
  constructor(private readonly requiredPermission: string) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    return user?.permissions?.includes(this.requiredPermission);
  }
}
```

创建工厂函数，接受参数传入动态守卫类，返回一个新的守卫实例。

```ts
import { DynamicAuthGuard } from "./dynamic-auth.guard";

export const PermissionGuard = (permission: string) =>
  new DynamicAuthGuard(permission);
```

## 使用

将参数传入工厂函数，动态创建守卫，传入 `@UseGuards()`  装饰器使用。

```ts
@Controller("resources")
export class ResourcesController {
  @Get("view")
  @UseGuards(PermissionGuard("view")) // 需要 `view` 权限
  viewResource() {
    return { message: "You can view this resource" };
  }

  @Get("edit")
  @UseGuards(PermissionGuard("edit")) // 需要 `edit` 权限
  editResource() {
    return { message: "You can edit this resource" };
  }
}
```

# 守卫的使用场景

守卫有两个常见的使用场景：

- 身份验证：确定用户是否已登录。
- 权限控制：检查用户是否有权限执行特定操作。

## JWT 身份验证

首先来看使用 JWT 配合守卫进行身份验证的场景。

### JWT 简介

JWT（Json Web Token），本质是一个字符串，是一个常用于身份验证的轻量级令牌格式。

#### 组成部分

由三个部分组成：

```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEyMywicm9sZSI6ImFkbWluIiwiaWF0IjoxNjg2MDAwMDAwLCJleHAiOjE2ODYwMDYwMDB9.qBf8oqKf0OOGROGRO6CqN3TxN3p4s6F2G8HQtkFJ0w4
```

- `header`：头部，包含令牌元数据，比如类型和签名算法

```json
{
  "alg": "HS256", // 签名算法（HMAC SHA-256）
  "typ": "JWT" // 令牌类型
}
```

- `payload`：负载，存储用户信息，比如：

```json
{
  "userId": 123,
  "role": "admin"
}
```

- `signature`：签名，用于保证令牌不会被伪造和篡改。计算方式是：按照 `header` 中固定的签名算法，指定一个密钥，对 `header` 和 `payload` 进行加密。

```ts
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secretKey);
```

#### 常见工作流

1. 用户登录 -> 服务器验证身份并签发 JWT。

2. 客户端存储 JWT（通常存放在 `localStorage` 或 `HttpOnly Cookie`）。

3. 客户端在请求中带上 JWT（通常在请求头增加 `Authorization: Bearer <token>`）。

4. 服务器验证 JWT，如果有效则返回资源，否则返回 `401 Unauthorized`。

### JWT 守卫实现

#### 安装 @nest/jwt

```
pnpm i @nest/jwt
```

#### 注入 JwtModule

从 `@nest/jwt` 导入 `JwtModule`，注入到 `AuthModule` 使用。

```ts
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { UsersModule } from "../users/users.module";
import { JwtModule } from "@nestjs/jwt";
import { AuthController } from "./auth.controller";
import { jwtConstants } from "./constants";
@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: "60s" },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

#### 签发 JWT

从 `@nest/jwt` 导入 `JwtService`，验证用户身份后，使用 `jwtService.signAsync` 签发 JWT 返回。

```ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { UsersService } from "../users/users.service";
import { JwtService } from "@nestjs/jwt";
@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}
  async signIn(username, pass) {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { sub: user.userId, username: user.username };
    return {
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
```

#### 实现身份验证守卫

创建 `AuthGuard`，`canActive` 方法中，从 header 中解析 token，使用 `jwtService.verifyAsync` 进行验证，验证成功返回 `true`，否则抛出错误。

```ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { jwtConstants } from "./constants";
import { Request } from "express";

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      const payload = await this.jwtService.verifyAsync(token, {
        secret: jwtConstants.secret,
      });
      request["user"] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(" ") ?? [];
    return type === "Bearer" ? token : undefined;
  }
}
```

## 基于角色的访问控制

基于角色的访问控制，一般基于 **元信息** + **装饰器** + **守卫** 实现。

### 元信息的设置和读取

#### SetMetaData

`SetMetadata()` 是 Nest 中用于设置函数、类的元数据的方法，可以从 `@nestjs/common` 导入，通常将其与装饰器结合使用。

`SetMetadata()` 有两个参数：

- 元数据 key
- 元数据 value 值

```ts
import { SetMetadata } from "@nestjs/common";
export const ROLES_KEY = "roles";
export const Roles = (...roles) => SetMetadata(ROLES_KEY, roles);
```

```ts
@Controller("roles")
@Roles(Role.Admin)
export class RoleController {
  @Get()
  @Roles(Role.User)
  findAll() {}
}
```

#### Reflector

用于读取 `setMetaData` 定义的元数据。

**1. 从单个上下文读取元数据**

使用 `Reflector` 类的 `get` 方法，有两个参数：

- 元数据 key
- 通过 `ExecutionContext` 对象获取的控制器类或路由处理方法

如果元数据附加在路由处理方法上，使用 `context.getHandler()` 方法：

```ts
const roles = this.reflector.get<string[]>("roles", context.getHandler());
```

如果元数据附加在 Controller 上，使用 `context.getClass()` 获取控制器类：

```ts
const roles = this.reflector.get<string[]>("roles", context.getClass());
```

**2. 从多个上下文获取元数据**

比如上面的例子中元数据即被附加在 Controller 上，又被附加在路由处理方法上：

```ts
@Controller("roles")
@Roles(Role.Admin)
export class RoleController {
  @Get()
  @Roles(Role.User)
  findAll() {}
}
```

**覆盖数据**：使用 `getAllAndOverride` ，以下代码结果为 `[Role.User]` 。

```ts
const roles = this.reflector.getAllAndOverride<string[]>("roles", [
  context.getHandler(),
  context.getClass(),
]);
```

**合并数据**：使用 `getAllAndMerge` ，以下代码结果为 `[Role.Admin, Role.User]` 。

```ts
const roles = this.reflector.getAllAndMerge<string[]>("roles", [
  context.getHandler(),
  context.getClass(),
]);
```

### 实现

#### 通过装饰器设置元信息

使用 `setMetaData` 方法，结合装饰器，将元数据附加到路由处理程序。

```ts
import { SetMetadata } from "@nestjs/common";
export const ROLES_KEY = "roles";
export const Roles = (...roles) => SetMetadata(ROLES_KEY, roles);
```

```ts
@Controller("roles")
@Roles(Role.Admin)
export class RoleController {
  @Get()
  @Roles(Role.User)
  findAll() {}
}
```

#### 实现访问控制守卫

创建守卫，其中通过 `Reflector` 类读取装饰器定义的元数据，即进入控制器或路由处理方法需要的角色，与当前用户拥有的角色进行对比，判断是否能够进入处理程序。

```ts
import { CanActivate, ExecutionContext, Injectable } from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { Observable } from "rxjs";
import { ROLES_KEY } from "src/decorators/roles.decorator";
import { Role } from "src/enum/roles.enum";
import { UserService } from "src/user/user.service";
@Injectable()
export class RoleGuard implements CanActivate {
  constructor(private reflector: Reflector, private userService: UserService) {}
  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 获取meta中的角色信息
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(), // 从路由上读
      context.getClass(), // 从controller上读
    ]);
    if (!requiredRoles) {
      return true;
    }
    // 与用户信息对比校验
    const req = context.switchToHttp().getRequest();
    const user = await this.userService.findOneByUserName(req.user.username);
    const roleIds = user.roles.map((role) => role.id);
    return requiredRoles.some((role) => roleIds.includes(role));
  }
}
```

#### 使用守卫

```ts
@Controller("roles")
@Roles(Role.Admin)
@UseGuards(RoleGuard)
export class RoleController {}
```

# 总结

本章介绍了 Nest 中守卫的创建和使用，并通过具体实例介绍了守卫的两个经典使用场景：身份验证和访问控制。
