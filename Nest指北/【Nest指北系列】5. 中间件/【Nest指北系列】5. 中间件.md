Nest 中的**中间件**本质上是在路由处理程序之前调用的**函数**。允许开发者对请求或响应对象进行操作。

![image.png](1.webp)

由于 Nest 底层默认使用 Express 作为 HTTP 服务器，因此 Nest 的中间件默认情况下等价于 express 的中间件。所以，我们先来了解一下 express 的中间件。

# Express 中间件

Express 的中间件基于**洋葱模型**，`use` 方法使用 `req`、`res`、`next` 作为参数，依赖调用 `next()` 来执行下一个中间件。

```ts
const express = require("express");
const app = express();

app.use((req, res, next) => {
  console.log("中间件 1");
  next();
});

app.use((req, res, next) => {
  console.log("中间件 2");
  next();
});

// 处理请求
app.get("/", (req, res) => {
  res.send("Hello Express");
});

app.listen(3000);
```

# Nest 中间件的创建和使用

Nest 的中间件分为类和函数两种形式。

## 类中间件

### 创建

#### 创建 NestMiddleware 类

使用 nest cli 快速创建

    nest g middleware test

执行命令后，在 src/test 下创建 test.middleware.ts 文件：

```ts
import { Injectable, NestMiddleware } from "@nestjs/common";

@Injectable()
export class TestMiddleware implements NestMiddleware {
  use(req: any, res: any, next: () => void) {
    next();
  }
}
```

可以看出，Nest 中间件类需要实现  `NestMiddleware`  接口，且被  `@Injectable()`  装饰。

#### 与 express 类型对应

因为 nest cli 不知道我们在 Nest 底层使用 express 还是 fastify，所以 `req`、`res` 参数的类型都是 `any`。我们修改一下它们的类型：

```ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

@Injectable()
export class TestMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    next();
  }
}
```

这时 Nest 中间件和 Express 中间件完全等价了：

- 实现 `use` 方法。
- `use` 方法的 `req`、`res`、`next` 参数完全来自于 Express API。
- `use` 方法中可以执行任何代码，比如对请求、响应对象进行操作等。
- 必须调用 `next()` 将控制权传递给下一个中间件函数，否则请求将会被挂起。

#### 依赖注入

Nest 中间件通过 `@Injectable`  装饰器声明这个类被 IoC 容器接管，可以通过依赖注入使用其他 service。

```ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { Request, Response, NextFunction } from "express";

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  constructor(private configService: ConfigService) {}

  use(req: Request, res: Response, next: NextFunction) {
    const logLevel = this.configService.get<string>("LOG_LEVEL") || "info";
    console.log(`[${logLevel}] Request: ${req.method} ${req.url}`);
    next();
  }
}
```

### 注册

注册中间件的模块需要实现 `NestModule` 接口，然后使用  `configure()`  方法来进行注册。

```ts
import { Module, NestModule, MiddlewareConsumer } from "@nestjs/common";

@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(TestMiddleware).forRoutes("*");
  }
}
```

`MiddlewareConsumer` 是一个辅助类，它提供了几种内置方法来管理中间件，这些方法都可以使用**链式调用**的方式组织在一起。比如 `apply` 、 `forRoutes` 、 `exclude` 。

#### apply

`apply()` 方法用于注册中间件，可以接受单个中间件，也可以接受多个中间件。

```ts
consumer.apply(TestMiddleware).forRoutes("*");
```

```ts
consumer.apply(TestMiddleware, TestMiddleware2).forRoutes("*");
```

#### forRoutes

`forRoutes()` 方法用于定义中间件对哪些路由生效，可以接受的参数类型如下：

##### 接受字符串（单个或多个）

表示路由路径。

```ts
// 表示 TestMiddleware 仅适用于 /user 路由
app.use(TestMiddleware).forRoutes("user");
```

```ts
// 表示 TestMiddleware 适用于 `/user` 和 `/order`
app.use(TestMiddleware).forRoutes("user", "order");
```

##### 接受 RouteInfo 对象

用于精确指定路由，包括方法和路径。

```ts
// 表示 TestMiddleware 适用于 `GET /user` 和 `POST /order`
app
  .use(TestMiddleware)
  .forRoutes(
    { path: "user", method: RequestMethod.GET },
    { path: "order", method: RequestMethod.POST }
  );
```

##### 接受控制器类（单个或多个）

表示应用到某个或多个控制器上的所有路由。

```ts
app.use(TestMiddleware).forRoutes(UserController);
```

```ts
app.use(TestMiddleware).forRoutes(UserController, OrderController);
```

##### 路由通配符

`forRoutes()` 方法传入的字符串和 `RouteInfo` 对象的 `path` 属性上都可以使用路由通配符，比如：

```ts
// 匹配所有路由
app.use(TestMiddleware).forRoutes("*");
```

```ts
// 匹配 abcd、ab_cd、abecd 等等。
app.use(TestMiddleware).forRoutes({ path: "ab*cd", method: RequestMethod.ALL });
```

字符 ""、"+"、"\*"、"()" 都可以在路径中使用，它们是对应于正则表达式的子集。

#### exclude

`exclude()` 方法用于排除特定的路由不使用中间件。

```ts
// TestMiddleware 适用于 /user 及其子路由，但不会应用于 /user/login
app.use(TestMiddleware).exclude("user/login").forRoutes("user");
```

`exclude()` 方法和 `forRoutes()` 一样也可以接受 `RouteInfo` 对象。

```ts
app
  .use(TestMiddleware)
  .exclude({ path: "user/login", method: RequestMethod.POST })
  .forRoutes("user");
```

## 函数中间件

### 创建

也可以使用普通函数作为中间件，函数的参数和类中间件 `use` 方法的参数一致，接受 `req`、`res`、`next`  参数。

```ts
import { Request, Response, NextFunction } from "express";

export function testMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  console.log("Request");
  next();
}
```

### 注册

函数中间件的注册方式和类中间件一样，在 `configure` 方法中使用 `apply` 注册即可。

```ts
import { Module, NestModule, MiddlewareConsumer } from "@nestjs/common";

@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(TestMiddleware).forRoutes("*");
  }
}
```

## 如何选择类和函数形式

上面介绍了 Nest 中间件的两种呢形式，那么应该如何选择呢？

一般来说，类中间件使用更多。因为类中间件中支持依赖注入，使中间件更具有扩展性，更适合大型项目，也与 Nest 的核心思想更加相符。

## 路由中间件和全局中间件

Nest 中间件的应用范围可以是特定的路由也可以是全局。

特定路由的中间件使用 `MiddlewareConsumer` 上的 `apply` 方法注册， `forRoutes` 方法定义适用路由，上面已经有过相关例子。

全局范围的中间件即针对所有路由生效，通过在 main.ts 文件中使用 app.use() 方法注册。

```ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { TestMiddleware } from "./middleware/testt.middleware";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 注册全局中间件
  app.use(TestMiddleware);

  await app.listen(3000);
}
bootstrap();
```

# Nest 中间件的使用场景

Nest 中间件用于在路由处理程序之前执行某些逻辑，以及处理请求和响应。常见的使用场景如下：

## 日志

比如记录请求信息等。

```ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
    next();
  }
}
```

## 身份验证

在请求到达路由处理程序之前，验证用户身份或权限，并记录用户信息

```ts
import {
  Injectable,
  NestMiddleware,
  UnauthorizedException,
} from "@nestjs/common";
import { Request, Response, NextFunction } from "express";
import * as jwt from "jsonwebtoken";

@Injectable()
export class AuthMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const token = req.headers.authorization?.split(" ")[1];
    if (!token) {
      throw new UnauthorizedException("No token provided");
    }

    try {
      const decoded = jwt.verify(token, "your-secret-key");
      req.user = decoded;
      next();
    } catch (err) {
      throw new UnauthorizedException("Invalid token");
    }
  }
}
```

## 请求修改

比如在请求上下文上增加额外信息。

```ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";
import { v4 as uuidv4 } from "uuid";

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    req.requestId = uuidv4(); // 为每个请求生成唯一的 ID
    next();
  }
}
```

## 响应修改

比如缓存响应数据，有缓存则直接返回。

```ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

const cache = new Map<string, any>();

@Injectable()
export class CacheMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const cacheKey = req.originalUrl;

    if (cache.has(cacheKey)) {
      return res.json(cache.get(cacheKey));
    }

    next();
  }
}
```

还有其他很多场景，这里就不一一举例了。

# 总结

- Nest 中间件的本质是一个函数。

- Nest 的中间件和 Express 的中间件默认情况下是等价的，都实现了 `use` 方法，接受  `req`、`res`、`next`  参数，且通过调用  `next()`  执行下一个中间件函数。二者最大的区别是，Nest 中间件支持依赖注入。

- Nest 中间件分为 class 形式和 function 形式。二者最大的区别也是是否支持依赖注入。

- Nest 注册中间件时，可以针对全局，也可以针对特定路由。针对特定路由时，可以通过字符串、`RouteInfo` 对象或控制器指定，也可以使用通配符。

- Nest 中间件有多种使用场景，包括日志、身份验证、请求响应的修改等。
