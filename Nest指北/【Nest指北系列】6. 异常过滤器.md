Nest 有一个内置的**异常层**，对应用程序运行过程中的所有未处理异常进行捕获，交给相应的**异常过滤器**处理，处理后返回给用户更友好的响应。

# 异常类

异常过滤器可以捕获各种类型的异常，所以，我们先来了解一下 Nest 中的异常类。

## HttpException 基础异常类

Nest 提供了一个内置的 HttpException 类，可以从 `@nestjs/common` 导入，用于发送标准的 http 响应对象。

### 参数

两个参数：`response` 和 `status`。

`response`：定义响应体，可以是 string 或 object。

`status`：定义 http 状态码。

### 响应对象

调用 HttpException 类返回的响应对象默认情况下包含 `statusCode` 和 `message` 两个属性，也可以自定义，具体和传入的参数相关。

- 如果 `response` 参数是 string，仅覆盖响应对象的 `message` 属性，`statusCode` 属性为 `status` 参数的值。

```ts
@Get()
async findAll() {
    throw new HttpException("Forbidden", HttpStatus.FORBIDDEN);
}
```

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

- 如果 `response` 参数是 object，则可以覆盖整个响应体，即可完全自定义。

```ts
@Get()
async findAll() {
    throw new HttpException({
        status: HttpStatus.FORBIDDEN,
        error: "custom message",
    }, HttpStatus.FORBIDDEN);
}
```

```json
{
  "status": 403,
  "error": "custom message"
}
```

## HttpException 的常见子类

Nest 还内置了一系列继承自基础异常类 HttpException 的子类，都可以从 `@nestjs/common` 导入，使用后会自动修改状态码。比如：

### BadRequestException (400)

通常用于表示客户端发送的请求有错误，服务器无法处理。比如参数缺失、无效输入等情况。

### UnauthorizedException (401)

通常表示请求没有经过身份验证或认证失败。

### ForbiddenException (403)

通常表示客户端身份认证成功，但没有权限访问请求的资源。

等等。

## 自定义异常类

Nest 中除了内置的异常类，还可以创建自定义异常类。自定义异常类继承自 HttpException 基类，对其进行扩展以创建更具有描述性和更可控的错误。

```ts
export class ForbiddenException extends HttpException {
  constructor() {
    super("Forbidden", HttpStatus.FORBIDDEN);
  }
}
```

```ts
@Get()
async findAll() {
    throw new ForbiddenException();
}
```

# 内置异常过滤器

Nest 内置全局异常过滤器，处理 HttpException 以及其子类的异常。比如，默认情况下访问不存在的路径，返回的报错如下，这个报错就是 Nest 默认的异常过滤器处理后返回的。

```json
{
  "message": "Cannot GET /api/user",
  "error": "Not Found",
  "statusCode": 404
}
```

当异常无法被识别时（即异常既不是 HttpException 也不是它的子类），内置异常过滤器会生成以下默认的响应对象：

```json
{
  "message": "Internal server error",
  "statusCode": 500
}
```

# 自定义异常过滤器

内置的异常过滤器已经可以自动处理大多数情况，但有时候我们希望对异常层有更多的控制。比如，希望返回自定义的 json 结构，或者希望添加一些日志。这时，就需要自定义异常过滤器。

## 创建

### nest cli 创建异常过滤器

可以通过 nest cli 快速创建异常过滤器：

```
nest g filter httpException
```

执行这行命令后，在 src/http-exception 文件夹中创建 http-exception.filter.ts 文件。

```ts
import { ArgumentsHost, Catch, ExceptionFilter } from "@nestjs/common";

@Catch()
export class HttpExceptionFilter<T> implements ExceptionFilter {
  catch(exception: T, host: ArgumentsHost) {}
}
```

### 实现 catch 方法

可以看到自定义的异常过滤器需要实现 `ExceptionFilter` 接口，这个接口只有一个 `catch` 方法。

#### 参数

`catch` 方法有两个参数：

- `exception`：表示当前正在处理的异常对象。

- `host`：一个 `ArgumentsHost` 上下文对象，可以用于获取 `Request` 和 `Response` 对象的引用。

```ts
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

#### 返回响应

当异常被捕获到过滤器中后，必须使用 `response` 对象返回数据给客户端，否则请求方将永远收不到任何响应，导致超时异常。

```ts
catch(exception: HttpException, host: ArgumentsHost) {
  const ctx = host.switchToHttp();
  const response = ctx.getResponse<Response>();

  console.log('异常捕获:', exception.message);
  // ❌ 没有 response 处理，客户端将一直等待，最终超时
}
```

正确的写法：

```ts
catch(exception: HttpException, host: ArgumentsHost) {
  const ctx = host.switchToHttp();
  const response = ctx.getResponse<Response>();
  const status = exception.getStatus ? exception.getStatus() : 500;

  response.status(status).json({
    statusCode: status,
    message: exception.message || '服务器内部错误',
    timestamp: new Date().toISOString(),
  });
}
```

### 定义 @Catch 装饰器

`@Catch` 装饰器用于指定要捕获的异常类，可以传入单个参数或逗号分隔的参数列表。

#### 捕获单个异常

`@Catch(HttpException)` ：只捕获 `HttpException` 及其子类。

#### 捕获多个异常

`@Catch(HttpException, TypeError)`：同时捕获 `HttpException` 和 `TypeError`

#### 捕获所有异常

如果 `@Catch` 不传参数，则该过滤器会捕获所有异常。通常用于全局异常处处理。

```ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    console.log("捕获到未处理异常:", exception);
  }
}
```

## 继承

自定义异常过滤器可以像上面那样完全自己创建，也可以继承自某个已实现的过滤器，仅对其做一些功能扩展。这样可以复用逻辑，提高代码复用性。

比如下面的例子中，先创建一个 `BaseExceptionFilter` ，用于返回约定的响应格式：

```ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpStatus,
} from "@nestjs/common";
import { Response, Request } from "express";

@Catch() // 捕获所有异常
export class BaseExceptionFilter implements ExceptionFilter {
  catch(exception: any, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status = HttpStatus.INTERNAL_SERVER_ERROR;

    response.status(status).json({
      statusCode: status,
      message: "服务器发生未知错误",
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

然后继承 BaseExceptionFilter ，创建 `LoggingExceptionFilter`，其中使用 `super.catch()` 调用基类的异常处理逻辑，并在此基础上扩展日志记录功能。

```ts
@Catch(HttpException)
export class LoggingExceptionFilter extends BaseExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    console.log(`⚠️ 捕获 HTTP 异常: ${exception.message}`);

    // 调用基类的通用异常处理逻辑
    super.catch(exception, host);
  }
}
```

## 绑定

创建完异常过滤器后，我们需要将其绑定使用。异常过滤器的绑定范围有：方法范围、控制器范围、全局范围。其中方法范围和控制器范围都使用 `@UseFilter()` 装饰器进行绑定。

### @UseFilter() 装饰器

- 可以传入实例；也可以直接传入类，将实例化的过程交给 Nest 框架进行。
  更推荐使用类的方式，这样可以在多个模块中复用同一个实例。

```ts
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}

@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

- 可以接受单个参数或逗号分隔的参数列表。

```ts
@UseFilters(HttpExceptionFilter, LoggingExceptionFilter)
```

### 方法范围

```ts
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

### 控制器范围

```ts
@UseFilters(new HttpExceptionFilter())
export class UserController {}
```

### 全局范围

- 全局范围过滤器通过 `app.useGlobalFilters` 绑定。

- 全局过滤器用于整个应用程序、每个控制器和每个路由处理程序。

- 被方法过滤器或控制器过滤器拦截后，异常不会再进入全局过滤器。

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

使用 `app.useGlobalFilters` 方式绑定的全局过滤器无法使用依赖注入，为了解决这个问题，可以使用下面的写法：

```ts
import { Module } from "@nestjs/common";
import { APP_FILTER } from "@nestjs/core";
@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

# 异常过滤器的使用场景

- **统一异常处理**

使用全局异常过滤器，确保所有错误（比如 http 相关错误、数据库相关错误、运行时错误等）响应格式一致。

- **优化报错信息**

- **日志记录**

- 等等

# 总结

Nest 中异常过滤器用于捕获异常，提供更友好的响应，在应用开发中有着重要的作用。本章介绍了 Nest 中的异常类、内置异常过滤器和自定义异常过滤器，以及它们的使用场景。
