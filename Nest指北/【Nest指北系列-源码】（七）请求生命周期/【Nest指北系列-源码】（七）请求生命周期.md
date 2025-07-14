上一篇[【Nest 指北系列-源码】（六）创建应用实例和初始化应用](https://juejin.cn/post/7520467788377833472) 讲到 Nest 中路由注册流程，通过 `Reflect.getMetadata()` 获取控制器中 `@Get()` 、`@Post()` 等装饰器定义的**路径**、**请求方法**、**请求处理函数**等元数据，然后绑定到 HTTP 服务器的路由系统中。

在 Nest 中，一个请求从进入应用到最终响应，除了执行请求处理函数外，还会依次经过很多处理步骤，完整链路如下：

`Middleware` -> `Guards` -> `Interceptors(before)` -> `Pipes` -> `Controller handler` -> `Interceptors(after)` -> `Exception Filters`

本文将会分析这条执行链是如何构建的。

---

先回到 Nest 项目的入口文件：

```ts
// main.ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

其中，`NestFactory.create()` 创建 `app` 实例，然后通过 `app.listen()` 监听端口。[listen](https://juejin.cn/post/7520467788377833472#heading-2) 方法中先调用 [init](https://juejin.cn/post/7520467788377833472#heading-3) 方法初始化，再调用底层 HTTP 服务器的 `listen` 方法。比如底层服务器为 Express 的话，则进入：

```ts
@Injectable()
export class ExpressAdapter extends AbstractHttpAdapter {
  listen() { this.httpServer.listen(...) }
}
```

之后对于每个请求，会调用其路径对应的 **Nest 路由代理**。

# Nest 路由代理

Nest 路由代理是什么呢？让我们来看创建它，并将它绑定到底层 HTTP 框架的源码。

源码路径：`packages/core/router/router-explorer.ts`。

上一篇[【Nest 指北系列-源码】（六）创建应用实例和初始化应用](url)中提到了 `applyPathsToRouterProxy()` 方法，其中对于每一个路由定义调用 `applyCallbackToRouter()` 方法。

## applyCallbackToRouter

桥梁函数，负责创建 Nest 路由代理，并将它注册到底层框架中。

精简后的源码如下：

```ts
private applyCallbackToRouter<T extends HttpServer>(
  router: T,
  routeDefinition: RouteDefinition,
  instanceWrapper: InstanceWrapper,
  moduleKey: string,
  routePathMetadata: RoutePathMetadata,
  host: string | RegExp | Array<string | RegExp>,
) {
  const {
    path: paths,
    requestMethod,
    targetCallback,
    methodName,
  } = routeDefinition;

  const { instance } = instanceWrapper;
  const routerMethodRef = this.routerMethodFactory
    .get(router, requestMethod)
    .bind(router);

  // ......
  const proxy = this.createCallbackProxy(
    instance,
    targetCallback,
    methodName,
    moduleKey,
    requestMethod,
  );

  // ......
  let routeHandler = this.applyHostFilter(host, proxy);

  paths.forEach(path => {
    // ......

    routePathMetadata.methodPath = path;
    const pathsToRegister = this.routePathFactory.create(
      routePathMetadata,
      requestMethod,
    );
    pathsToRegister.forEach(path => {
      // ......

      const normalizedPath = router.normalizePath
        ? router.normalizePath(path)
        : path;
      routerMethodRef(normalizedPath, routeHandler);

      // ......
    });

    // ......
  });
}
```

**参数**：

- `router`：http 服务器适配器实例，比如 Express 或 Fastify 的包装类。

- `routeDefinition`：路由定义。

- `InstanceWrapper`：控制器的 `InstanceWrapper` 实例。

- `moduleKey`：模块唯一标识。

- `routePathMetadata`: 路由 path 元数据，包含模块路径、控制器路径。

**核心流程**：

**1. 通过 `routerMethodFactory.get()` 获取 http adapter 中的对应处理方法**

可以看下 `RouterMethodFactory` 类的实现：

```ts
export const REQUEST_METHOD_MAP = {
  [RequestMethod.GET]: "get",
  [RequestMethod.POST]: "post",
  [RequestMethod.PUT]: "put",
  [RequestMethod.DELETE]: "delete",
  [RequestMethod.PATCH]: "patch",
  [RequestMethod.ALL]: "all",
  // ......
} as const satisfies Record<RequestMethod, keyof HttpServer>;

export class RouterMethodFactory {
  public get(target: HttpServer, requestMethod: RequestMethod): Function {
    const methodName = REQUEST_METHOD_MAP[requestMethod];
    const method = target[methodName];
    if (!method) {
      return target.use;
    }
    return method;
  }
}
```

通过一个映射表把 Nest 的 `RequestMethod` 枚举转换为字符串方法名，然后在 http adapter 上获取对应处理方法。

**2. 创建 Nest 路由代理**

Nest 不直接调用控制器原始方法，而是调用 `createCallbackProxy` 方法封装一个代理。它的具体实现下面会讲到。

**3. 将路由代理注册到底层框架中**

- `routePathFactory.create(...)` 负责合并模块、控制器和方法路径。

- `normalizePath()` 会根据 Express/Fastify 规范化路径。

- 最后调用 `routerMethodRef(path, handler)` 即 `router.get(path, handler)` 之类的方法完成注册。

## createCallbackProxy

创建路由代理，其中处理 `Guards`、`Interceptors`、`Pipes`、`Exception Filters`。

```ts
private createCallbackProxy(
  instance: Controller,
  callback: RouterProxyCallback,
  methodName: string,
  moduleRef: string,
  requestMethod: RequestMethod,
  contextId = STATIC_CONTEXT,
  inquirerId?: string,
) {
  const executionContext = this.executionContextCreator.create(
    instance,
    callback,
    methodName,
    moduleRef,
    requestMethod,
    contextId,
    inquirerId,
  );
  const exceptionFilter = this.exceptionsFilter.create(
    instance,
    callback,
    moduleRef,
    contextId,
    inquirerId,
  );
  return this.routerProxy.createProxy(executionContext, exceptionFilter);
}
```

**参数**：

- `instance`: 控制器实例。

- `callback`: 控制器上的 handler 方法。

- `methodName`: 路由方法名。

- `moduleRef`: 模块唯一标识。

- `requestMethod`: 请求方法类型（GET / POST 等）。

**核心流程**：

1. 调用 `executionContextCreator.create(...)` 创建一个 `executionContext` 对象，即最终的路由处理函数，内部集成了 handler 函数、守卫、拦截器、管道、响应头、状态码处理。可以理解为，它负责生成一个**完整的执行链上下文**。

2. 调用 `exceptionsFilter.create(...)` 创建一个 `exceptionFilter` 对象，即异常处理函数，内部包含全局默认异常过滤器以及自定义的异常过滤器。用于包装上一步构建的路由处理函数，当它抛出异常时，统一由异常处理函数处理。

3. 调用 `routerProxy.createProxy(...)` 将路由处理链和异常处理链包装成最终的路由代理。

### executionContext

构建路由处理链。

**核心流程**：

**1. 通过 `Reflecet.getMetadata()` 获取控制器 handler 方法的元数据**

比如：

- http 状态码（`@HttpCode`）。
- 参数列表长度（`argsLength`）。
- 参数类型数组（用于管道验证）。
- 参数装饰器 `@Body`、`@Param` 等的映射。
- 响应头信息。

```ts
const {
  argsLength,
  fnHandleResponse,
  paramtypes,
  getParamsMetadata,
  httpStatusCode,
  responseHeaders,
  hasCustomHeaders,
} = this.getMetadata(
  instance,
  callback,
  methodName,
  moduleKey,
  requestMethod,
  contextType
);
```

**2. 将参数元数据和参数类型数组合并为 `paramOptions` 结构，供管道使用**

```ts
const paramsOptions = this.contextUtils.mergeParamsMetatypes(...);
```

**3. 构建组件链**

```ts
const pipes = this.pipesContextCreator.create(...);
const guards = this.guardsContextCreator.create(...);
const interceptors = this.interceptorsContextCreator.create(...);
```

- 根据控制器 handler 方法上的装饰器（如 `@UsePipes()` 等）创建执行链。这些 `pipes`、`guards`、`interceptors` 都是实例化后的，每次请求都会执行。

**4. 构建逻辑函数**

```ts
const fnCanActivate = this.createGuardsFn(...);
const fnApplyPipes = this.createPipesFn(...);
```

将上一步获取的 `guards`、`pipes` 数组封装成函数，比如会取 `guard` 上的 `canActive` 方法。

**5. 构建 handler 包裹器**

```ts
const handler = (args, req, res, next) => async () => {
  fnApplyPipes && (await fnApplyPipes(args, req, res, next));
  return callback.apply(instance, args);
};
```

对控制器 handler 方法的封装，会**先执行管道处理函数，再执行控制器方法**。

**6. 生成最终执行函数**

```ts
return async (req, res, next) => {
  const args = this.contextUtils.createNullArray(argsLength);

  fnCanActivate && (await fnCanActivate([req, res, next])); // 守卫判断

  this.responseController.setStatus(res, httpStatusCode);
  if (hasCustomHeaders) {
    this.responseController.setHeaders(res, responseHeaders);
  }

  const result = await this.interceptorsConsumer.intercept(
    interceptors,
    [req, res, next],
    instance,
    callback,
    handler(args, req, res, next), // 包含管道逻辑的实际 handler
    contextType
  );

  await (fnHandleResponse as HandlerResponseBasicFn)(result, res, req); // 最终返回
};
```

其中按顺序：

- 执行**守卫**判断。
- 设置状态码与响应头。
- 执行**拦截器**链。
- 执行包含**管道**逻辑的实际控制器 **handler** 方法。
- 最终将返回值通过响应控制器输出到 res。

> 可以看出，`executionContext.create(...)` 返回了最终的路由处理函数，内部集成了守卫、拦截器、管道、响应头、状态码处理，整体执行顺序为：`guards` -> `interceptors` -> `pipes` -> `handler`。

### exceptionFilter

构建异常处理器链。

**核心流程**：

**1. 创建默认异常处理器**

```ts
const exceptionHandler = new ExceptionsHandler(this.applicationRef);
```

- `applicationRef` 是对 Express / Fastify Response API 的抽象封装。

- `ExceptionsHandler` 是 Nest 默认的异常处理器类，内部提供 `next.handle(exception, host)` 方法。

**2. 读取装饰器元数据构建上下文链**

```ts
const filters = this.createContext(
  instance,
  callback,
  EXCEPTION_FILTERS_METADATA,
  contextId,
  inquirerId
);
```

- 通过 `ContextCreator` 类的方法，根据 `@UseFilters()` 装饰器获取控制器类和方法上注册的异常过滤器，创建上下文链。

**3. 如果没有注册自定义过滤器，返回默认异常处理器**

```ts
if (isEmpty(filters)) {
  return exceptionHandler;
}
```

- 如果没有自定义的 `@UseFilters()`，就返回默认 handler。

**4. 注册自定义过滤器链**

```ts
exceptionHandler.setCustomFilters(filters.reverse());
```

- 使用 `.reverse()` 让后定义的过滤器 **先执行**（类似中间件执行顺序）。

- 注册后，异常处理器在处理异常时会按顺序调用这些 `filters` 的 `catch(exception, host)` 方法。

**5. 返回最终异常处理器 handler**

```ts
return exceptionHandler;
```

- 返回的 handler 将被用于包装前面构建的路由处理链的执行过程（如果执行失败，会进入 `exceptionHandler`）。

### createProxy

将目标函数（路由 handler 函数）包装成带有异常处理的路由代理函数，用于最终绑定到底层框架中。

```ts
public createProxy(
  targetCallback: RouterProxyCallback,
  exceptionsHandler: ExceptionsHandler,
) {
  return async <TRequest, TResponse>(
    req: TRequest,
    res: TResponse,
    next: () => void,
  ) => {
    try {
      await targetCallback(req, res, next);
    } catch (e) {
      const host = new ExecutionContextHost([req, res, next]);
      exceptionsHandler.next(e, host);
      return res;
    }
  };
}
```

**参数**：

- `targetCallback`：路由处理函数（含守卫/拦截器/管道等）。

- `exceptionsHandler`：异常处理函数。

**核心逻辑**：

1. `try..catch...` 包裹路由处理函数，`catch` 捕获异常，执行异常处理链。

2. `ExecutionContextHost` 是一个包装器，将 `[req, res, next]` 提供给异常处理器，供其调用 `getRequest()`、`getResponse()` 等 API。

3. 调用 `exceptionsHandler.next(...)` 会遍历注册的异常过滤器，尝试依次调用其 `catch(exception, host)` 方法。

---

分析完 `applyCallbackToRouter` 和 `createCallbackProxy` 方法，可以理解 Nest 中每个路由回调是如下形式：

```ts
app.get("/users", async (req, res, next) => {
  try {
    // 守卫 → 拦截器 → 管道 → 控制器方法
  } catch (e) {
    // 自定义异常过滤器
  }
});
```

整条链路中还差一个**中间件**，它是在什么阶段处理的呢？我们继续来看。

## 中间件处理

关于中间件的注册，我们要回到 `NestApplication` 应用实例的 [init](url) 方法。其中调用 [`registerModules`](url) -> `this.middlewareModule.register` -> `resolveMiddleware`。使用 `expressApp.use(...)` 将中间件注册到底层框架。

```ts
public async register(
  middlewareContainer: MiddlewareContainer,
  container: NestContainer,
  config: ApplicationConfig,
  injector: Injector,
  httpAdapter: HttpServer,
  graphInspector: GraphInspector,
  options: TAppOptions,
) {
  // ......

  const modules = container.getModules();
  await this.resolveMiddleware(middlewareContainer, modules);
}

public async resolveMiddleware(
  middlewareContainer: MiddlewareContainer,
  modules: Map<string, Module>,
) {
  const moduleEntries = [...modules.entries()];
  const loadMiddlewareConfiguration = async ([moduleName, moduleRef]: [
    string,
    Module,
  ]) => {
    await this.loadConfiguration(middlewareContainer, moduleRef, moduleName);
    await this.resolver.resolveInstances(moduleRef, moduleName);
  };
  await Promise.all(moduleEntries.map(loadMiddlewareConfiguration));
}
```

于是，每次请求进入之后会先执行中间件。中间件不在 Nest 创建的路由代理中。

---

# 总结

最终，Nest 请求的整个生命周期是这样的：

```ts
                            Incoming HTTP Request
                                     ｜
                        Http Adapter (Express/Fastify)
                                     ｜
                                 Middleware
                                     ｜
                                handler proxy
                                     ｜
                      ---------------------------------——
                     ｜                                  ｜
 ┌─────────────────────────────────────────┐   ┌──────────────────┐
 │ 路由处理链                                |   │ 异常处理链        ｜
 | Guard → Interceptor → Pipe → Controller |   | Exception Filter |
 └─────────────────────────────────────────┘   └──────────────────┘
```
