Nest 中的**拦截器**（**Interceptor**）用于在请求处理函数之前和之后进行逻辑处理。有点类似于 axios 中的请求拦截器和响应拦截器。常见的应用场景有转换返回结果、缓存拦截器、超时拦截器等。

Nest 中拦截器的核心逻辑（对返回数据的流式操作）依赖于 RxJS 库，所以我们先简单介绍一下 RxJS。

# RxJS

RxJS 是一个用于处理异步数据流的库，它提供了很多操作符，用于简化异步逻辑的编写。

## 核心概念

先来了解 RxJS 的几个核心概念：

### Observable 可观察对象

表示一个随着时间推移可以发出多个值的数据流。

1. 使用 `new Observable` 可以创建一个 Observable 对象。

2. `new Observable` 回调函数中的逻辑为 Observable 执行逻辑，它是惰性运算的，只有在观察者订阅后才会执行。

3. Observable 执行可以推送三种类型的通知：

- `next()` 发送数据
- `error()` 发送错误
- `complete()` 结束数据流
  `error()` 和 `complete()` 只能在 Observable 执行期间发生一次，且只会执行其中一个。

```ts
import { Observable } from "rxjs";

const observable = new Observable((observer) => {
  observer.next("Hello");
  observer.next("World");
  observer.complete();
});
```

### Observer 观察者

监听 Observable 发出的值。

1. Observer 是一组回调的集合，每个回调函数对应一种 Observable 推送的通知类型。

2. 要使用观察者，需要把它传递给 Observable 对象的 `subsrcibe` 方法。订阅建立起 Observable 和 Observer 之间的联系，订阅后观察者可以接收 Observable 流发出的数据。

```ts
const observer = {
  next: (value) => console.log("Next:", value),
  error: (err) => console.error("Error:", err),
  complete: () => console.log("Complete!"),
};

observable.subscribe(observer);

// 输出
// Next: Hello
// Next: World
// Complete!
```

### Operators 操作符

RxJS 提供了大量的操作符来转换、过滤、组合、合并数据流。即**由数据源（Observable）产生的数据，经过一系列 Operators 处理，最后传给 Observer**。这些操作符在 Nest 拦截器对返回数据的处理中也起着重要的作用。

## 常见操作符

下面介绍一些 RxJS 中常见的操作符。

### 创建操作符

#### of

创建包含特定值的 Observable 流。

```ts
import { of } from "rxjs";
const observable = of(1, 2, 3);
```

#### from

将数组、promise、可迭代对象转为 Observable 流。

```ts
import { from } from "rxjs";
const observable = from([1, 2, 3]);
```

### 转换操作符

#### map

对每个值进行转换。

```ts
import { of } from "rxjs";
import { map } from "rxjs/operators";

of(1, 2, 3)
  .pipe(map((x) => x * 2))
  .subscribe(console.log); // 2 4 6
```

#### scan

类似 reduce，累计值并发出中间结果。

```ts
import { of } from "rxjs";
import { scan } from "rxjs/operators";
of(1, 2, 3)
  .pipe(scan((acc, x) => acc + x, 0))
  .subscribe(console.log); // 1 3 6
```

### 错误处理操作符

#### catchError

捕获发出的错误并返回新的 Observable。

```ts
import { of, throwError } from "rxjs";
import { catchError } from "rxjs/operators";
const source = throwError(() => new Error("Something went wrong!")); // 模拟错误
source
  .pipe(
    catchError((err) => {
      console.error("Error caught:", err.message);
      return of("Fallback value"); // 返回一个新的流作为替代
    })
  )
  .subscribe({
    next: (value) => console.log("Next:", value),
    error: (err) => console.error("Error:", err),
    complete: () => console.log("Complete!"),
  });

// 输出：
// Error caught: Something went wrong!
// Next: Fallback value
// Complete!
```

### 工具操作符

#### tap

对每个值执行副作用，但不会修改流的值。

```ts
import { of } from "rxjs";
import { tap } from "rxjs/operators";
of(1, 2, 3)
  .pipe(
    tap((value) => console.log(`Value passed through: ${value}`)) // 记录值
  )
  .subscribe((value) => console.log("Subscriber received:", value));

// 输出：
// Value passed through: 1
// Subscriber received: 1
// Value passed through: 2
// Subscriber received: 2
// Value passed through: 3
// Subscriber received: 3
```

#### timeout

在超时时间内未发出值时抛出错误。

```ts
import { interval } from "rxjs";
import { timeout } from "rxjs/operators";
interval(3000) // 每 3 秒发出一个值
  .pipe(timeout(2000)) // 超时设置为 2 秒
  .subscribe({
    next: (value) => console.log("Next:", value),
    error: (err) => console.error("Error:", err.message),
  });

// 输出：
// Error: Timeout has occurred
```

# 创建拦截器

下面我们来尝试创建一个 Nest 拦截器。

## nest cli 创建拦截器

```
nest g interceptor interceptor/transform
```

执行这行命令后，在 src/interceptor/transform 文件夹中创建 transform.interceptors.ts 文件：

```ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable } from "rxjs";

@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle();
  }
}
```

可以看出拦截器是具有 `@Injectable()` 装饰器的类，即声明这个类被 Nest IoC 容器接管，可以通过 `constructor` 注入依赖。同时，它需要实现  `NestInterceptor`  接口，这个接口有一个 `intercept` 方法。

## 实现 intercept 方法

### 参数

`intercept` 方法接收两个参数：

- `context`：一个 `ExecutionContext` 对象，与[【Nest 指北系列】守卫](https://juejin.cn/post/7493007413734752275#heading-2)中的 `canActive` 方法的参数相同。有 `getClass` 和 `getHandler` 两个方法，可以配合 `Reflector` 获取元数据。
- `next`: 接口定义如下，`next.handle()` 函数即路由处理函数，返回值为 Observable 对象包装的路由处理函数的返回值。后续可以通过 RxJS 操作符对这个 Observable 数据流进行处理，后续的应用场景一节中有具体的示例。

```ts
export interface CallHandler<T = any> {
  handle(): Observable<T>;
}
```

比如：访问 /user 时，路由处理函数返回 `[]` ，在应用拦截器的情况下，调用 `next.handle()` 方法的返回值就是 `Observable<[]>` 对象。

```ts
@Controller("user")
export class UserController {
  @Get()
  list() {
    return [];
  }
}
```

**注意：需要在拦截器中调用 next.handle() 方法才会执行对应路由处理函数，不调用就不会执行。**

# 使用拦截器

拦截器分为：方法拦截器、控制器拦截器、全局拦截器，其中方法拦截器和控制器拦截器都适用 `@UseInterceptors()` 装饰器绑定。

## @UseInterceptors 装饰器

- 可以传入实例；也可以直接传入类，将实例化的过程交给 Nest 框架进行。更推荐使用类的方式，这样可以在多个模块中复用同一个实例。

```ts
@UseInterceptors(TransformInterceptor)
export class UserController {}

@UseInterceptors(new TransformInterceptor())
export class UserController {}
```

## 控制器拦截器

```ts
@UseInterceptors(TransformInterceptor)
export class UserController {}
```

## 路由方法拦截器

```ts
export class UserController {
  @Get("")
  @UseInterceptors(TransformInterceptor)
  getUsers() {}
}
```

## 全局拦截器

通过  `app.useGlobalInterceptors`  绑定

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new TransformInterceptor());
```

## 执行顺序

拦截器的执行顺序类似**洋葱模型**，如果 TransformInterceptor 中在进入请求前和返回响应后分别打 log，执行顺序如下：

```ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable } from "rxjs";

@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log("Before");
    return next.handle().pipe(tap(() => console.log("After")));
  }
}
```

```ts
global Before
class Before
method Before
method After
class After
global After
```

# 拦截器应用场景

拦截器配合 RxJS 运算符可以实现多个应用场景。

## 在路由处理程序前后添加额外逻辑

实现计算路由处理程序执行时间并输出：

- 由于执行顺序为 拦截器 -> 路由处理程序 -> 拦截器，所以可以通过前后打点计算路由执行时间。
- **tap 运算符**：对 Observable 流执行副作用，控制台输出执行时间。

```ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from "@nestjs/common";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log("Before...");

    const controller = context.getClass(); // 获取当前调用的控制器
    const handler = context.getHandler(); // 获取当前调用的处理器

    // 保存路由执行前的时间
    const now = Date.now();
    return next.handle().pipe(
      // 计算出这个路由的执行时间
      tap(() => console.log(`After... ${Date.now() - now}ms`))
    );
  }
}
```

### 对比中间件

- 中间件只能在路由处理程序之前添加额外逻辑，而拦截器可以在之前和之后分别添加逻辑，且拦截器可以使用 RxJS 操作符对响应流进行处理。

- 拦截器可以从 context 拿到目标的 class 和 handler，进而配合 Reflector 获取元信息，而中间件不行。

## 转换路由处理程序的返回结果

实现统一的响应返回：

- **map 运算符**：对响应结果 Observable 流的每个值进行转换，得到标准输出格式。

```ts
import { Injectable, NestInterceptor, CallHandler } from "@nestjs/common";
import { map } from "rxjs/operators";
import { Observable } from "rxjs";

interface data<T> {
  data: T;
}

@Injectable()
export class Response<T = any> implements NestInterceptor {
  intercept(context, next: CallHandler): Observable<data<T>> {
    return next.handle().pipe(
      map((data) => {
        return {
          data,
          status: 0,
          success: true,
          message: "成功",
        };
      })
    );
  }
}
```

## 转换路由处理程序抛出的异常

- **catchError 运算符**：捕获错误并进行转换。

```ts
import { EntityNoFoundException } from "@common/exception/common.exception";
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable, throwError } from "rxjs";
import { catchError } from "rxjs/operators";

@Injectable()
export class TypeOrmExceptionInterceptor implements NestInterceptor {
  public intercept(
    context: ExecutionContext,
    next: CallHandler
  ): Observable<any> {
    return next.handle().pipe(
      catchError((err) => {
        if (err.name === "EntityNotFound") {
          return throwError(new EntityNoFoundException());
        }
        return throwError(err);
      })
    );
  }
}
```

## 扩展路由处理程序的行为

实现在用户调用某些接口后，对用户执行一些额外操作，比如添加角色：

- **tap 运算符**：在接口调用后执行副作用，给用户绑定上角色。

```ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { User } from "@src/auth/user/user.entity";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";
import { getConnection } from "typeorm";

@Injectable()
export class BindRoleToUserInterceptor implements NestInterceptor {
  public intercept(
    context: ExecutionContext,
    next: CallHandler
  ): Observable<any> {
    return next.handle().pipe(
      tap(async () => {
        const req = context.switchToHttp().getRequest();
        await this.bindRoleToUser(req.roleId, req.user.id);
      })
    );
  }
}
```

## 根据条件重写路由处理程序

实现缓存拦截器：

- 当命中缓存时，通过 **of 运算符**创建新的 Observable 流并返回，而不调用 `next.handle()`，即不走原来的路由处理程序。

```ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from "@nestjs/common";
import { Observable, of } from "rxjs";

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

## 超时拦截器

- **timeout 操作符**：在特定时间内没收到消息的时候抛出 `TimeoutError` 错误。
- **catchError 操作符**：捕获错误，判断如果是 `TimeoutError`，就返回 `RequestTimeoutException`，有内置的异常处理器对应这个异常，会返回标准的响应格式。

```ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  RequestTimeoutException,
} from "@nestjs/common";
import { Observable, throwError, TimeoutError } from "rxjs";
import { catchError, timeout } from "rxjs/operators";
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError((err) => {
        if (err instanceof TimeoutError) {
          return throwError(new RequestTimeoutException());
        }
        return throwError(err);
      })
    );
  }
}
```

# 总结

本章介绍了 Nest 中的拦截器，包括它的创建和使用。自定义拦截器时需要实现 `NestInterceptor` 类的 `intercept` 方法，其 `next` 参数的 `handle` 方法返回 RxJS Observable 对象，即路由处理函数的返回值，对其进行操作时需要用到 RxJS 库中的各种操作符。
