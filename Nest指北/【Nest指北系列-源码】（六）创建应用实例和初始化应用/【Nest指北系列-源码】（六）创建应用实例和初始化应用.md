> 前情回顾：
>
> [【Nest 指北系列-源码】（一）目录结构](https://juejin.cn/post/7498280232764751908)
>
> [【Nest 指北系列-源码】（二）装饰器和 reflect-metadata](https://juejin.cn/post/7500981295105458215)
>
> [【Nest 指北系列-源码】（三）启动流程](https://juejin.cn/post/7503390744385454091)：整体视角讲述 Nest 启动流程：创建 NestContainer 容器、依赖扫描和实例化、创建应用实例
>
> [【Nest 指北系列-源码】（四）NestContainer](https://juejin.cn/post/7510415261741531188)：启动流程第一步：创建 NestContainer 容器
>
> [【Nest 指北系列-源码】（五）依赖扫描和实例化](https://juejin.cn/editor/drafts/7496484123247263781)：启动流程第二步：依赖扫描和实例化

本章分析 Nest 启动流程源码的最后一部分：创建应用实例。我们从 `NestFactory.create()` 的 `new NestApplication()` 进入。

源码位置：`packages/core/nest-factory.ts`

```ts
export class NestFactoryStatic {
  public async create<T extends INestApplication = INestApplication>(
    moduleCls: IEntryNestModule,
    serverOrOptions?: AbstractHttpAdapter | NestApplicationOptions,
    options?: NestApplicationOptions
  ): Promise<T> {
    // ......

    // 1. 创建 IoC 容器
    const container = new NestContainer(applicationConfig);

    // ......

    // 2. initialize: 包含依赖扫描、实例化依赖等逻辑
    await this.initialize(
      moduleCls,
      container,
      graphInspector,
      applicationConfig,
      appOptions,
      httpServer
    );

    // 3. 创建应用实例
    const instance = new NestApplication(
      container,
      httpServer,
      applicationConfig,
      graphInspector,
      appOptions
    );
  }
}
```

# NestApplication

`NestApplication` 是 Nest 中管理整个 HTTP 应用实例的核心类，它维护了一系列方法，比如监听端口的 `listen` 方法，启动服务的 `init` 方法等。

源码位置：`packages/core/nest-application.ts`。

```ts
export class NestApplication
  extends NestApplicationContext<NestApplicationOptions>
  implements INestApplication
{
  constructor(
    container: NestContainer,
    private readonly httpAdapter: HttpServer,
    private readonly config: ApplicationConfig,
    private readonly graphInspector: GraphInspector,
    appOptions: NestApplicationOptions = {}
  ) {
    super(container, appOptions);

    this.selectContextModule();
    this.registerHttpServer();
    this.injector = new Injector({ preview: this.appOptions.preview! });
    this.middlewareModule = new MiddlewareModule();
    this.routesResolver = new RoutesResolver(
      this.container,
      this.config,
      this.injector,
      this.graphInspector
    );
  }

  public async listen(port: number | string, ...args: any[]): Promise<any> {}

  public async init(): Promise<this> {}
}
```

## 构造函数

`NestFactory.create()` 中 `new NestApplication()` 就是调用构造函数，创建应用实例。所以我们先来看 `NestApplication` 构造函数的实现。

**1. `super` 调用父类构造函数**

```ts
super(container, appOptions);
```

即 `NestApplicationContext` 的构造函数：

```ts
export class NestApplicationContext implements INestApplicationContext {
  constructor(
    protected readonly container: NestContainer,
    private readonly scope: ScopeOptions = {},
    private readonly graphInspector: GraphInspector
  ) {
    this.moduleCompiler = new ModuleCompiler();
    this.container.setGraphInspector(graphInspector);
  }
}
```

- 初始化 `NestContainer` 容器
- 创建 `ModuleCompiler` 模块编译器

**2. `selectContextModule` 确定当前上下文模块**

通常将入口模块作为上下文模块。后续在执行 `app.get(...)`、`app.resolve(...)` 等方法时，会从入口模块开始查找依赖。

```ts
public selectContextModule() {
  const modules = this.container.getModules().values();
  this.contextModule = modules.next().value!;
}
```

**3. `registerHttpServer` 注册底层 HTTP 服务器到 `this.httpServer` 上**

```ts
public registerHttpServer() {
  this.httpServer = this.createServer();
}
```

**4. 创建 `Injector` 实例**

用于执行依赖注入。

**5. 创建 `MiddlewareModule` 实例**

用于处理模块 `configure()` 方法中的中间件注册。在 `NestApplication` 类的 `init` 方法中会使用。

**6. 创建 `RoutesResolver` 实例**

用于注册控制器路由。在 `NestApplication` 类的 `init` 方法中会使用。

## listen

使用 Nest 框架时，我们一般通过 `NestFactory.create()` 创建 `app` 应用实例，然后通过 `app.listen()` 监听端口。

`listen` 是 `NestApplication` 类上的方法，其中主要干了两件事：

1. 调用 `NestApplication` 类上的 `init` 方法。

2. 返回一个 Promise，配合 `httpAdapter` 调用底层的 `express.listen()` 或 `fastify.listen()` 异步启动服务。

```ts
export class NestApplication
  extends NestApplicationContext<NestApplicationOptions>
  implements INestApplication
{
  public async listen(port: number | string, ...args: any[]): Promise<any> {
    if (!this.isInitialized) {
      // 1. init
      await this.init();
    }

    return new Promise((resolve, reject) => {
      // ......
      // 2. listen
      this.httpAdapter.listen(
        port,
        ...listenFnArgs,
        (...originalCallbackArgs: unknown[]) => {
          // ......
        }
      );
    });
  }
}
```

## init

`NestApplication` 类上的方法，用于初始化应用。

```ts
public async init(): Promise<this> {
  if (this.isInitialized) { // 1. 避免重复初始化
    return this;
  }

  this.applyOptions(); // 2. 处理应用配置项
  await this.httpAdapter?.init?.(); // 3. 初始化 HTTP 适配器

  const useBodyParser =
    this.appOptions && this.appOptions.bodyParser !== false;
  useBodyParser && this.registerParserMiddleware(); // 4. 注册 body-parser 中间件

  await this.registerModules(); // 5. 注册 WebSocket 模块、微服务系统、中间件
  await this.registerRouter(); // 6. 注册路由
  await this.callInitHook(); // 7. 触发 `onModuleInit` 生命周期钩子
  await this.registerRouterHooks(); // 8. 注册 404 处理器和全局异常处理器
  await this.callBootstrapHook(); // 9. 触发 `onApplicationBootstrap` 生命周期钩子

  this.isInitialized = true;
  this.logger.log(MESSAGES.APPLICATION_READY);
  return this; // 10. 返回 `this` 实例，允许链式调用
}
```

### 避免重复初始化

如果已经初始化，则直接返回，不再重复执行初始化逻辑。

### 处理应用配置项

根据应用配置项决定是否开启 CORS。

```ts
public applyOptions() {
  if (!this.appOptions || !this.appOptions.cors) {
    return undefined;
  }
  const passCustomOptions =
    isObject(this.appOptions.cors) || isFunction(this.appOptions.cors);
  if (!passCustomOptions) {
    return this.enableCors();
  }
  return this.enableCors(this.appOptions.cors);
}
```

### 初始化 HTTP 适配器

`httpAdapter` 是对 Express / Fastify 的封装，这里就是调用 Express / Fastify adapter 的 init 方法。

### 注册 body-parser 中间件

如果未禁用 bodyParser，则注册 JSON 和 URL-encoded 的解析中间件，默认使用 body-parser 包进行解析。

调用 http adapter 的 `registerParserMiddleware` 方法：

```ts
public registerParserMiddleware() {
  const prefix = this.config.getGlobalPrefix();
  const rawBody = !!this.appOptions?.rawBody;
  this.httpAdapter.registerParserMiddleware(prefix, rawBody);
}
```

实际使用 `expressApp.use(...)`，express adapter 源码：`packages/platform-express/adapters/express-adapter.ts`。

```ts
public registerParserMiddleware(prefix?: string, rawBody?: boolean) {
  const bodyParserJsonOptions = getBodyParserOptions(rawBody!);
  const bodyParserUrlencodedOptions = getBodyParserOptions(rawBody!, {
    extended: true,
  });

  const parserMiddleware = {
    jsonParser: express.json(bodyParserJsonOptions),
    urlencodedParser: express.urlencoded(bodyParserUrlencodedOptions),
  };
  Object.keys(parserMiddleware)
    .filter(parser => !this.isMiddlewareApplied(parser))
    .forEach(parserKey => this.use(parserMiddleware[parserKey]));
}
```

### 注册 WebSocket 模块、微服务系统、中间件

```ts
public async registerModules() {
  this.registerWsModule();

  if (this.microservicesModule) {
    this.microservicesModule.register(
      this.container,
      this.graphInspector,
      this.config,
      this.appOptions,
    );
    this.microservicesModule.setupClients(this.container);
  }

  await this.middlewareModule.register(
    this.middlewareContainer,
    this.container,
    this.config,
    this.injector,
    this.httpAdapter,
    this.graphInspector,
    this.appOptions,
  );
}
```

- 初始化并注册 WebSocket 模块，用于支持 @WebSocketGateway() 等功能。

- 初始化微服务系统。

- 使用 `MiddlewareModule` 将 `configure(consumer: MiddlewareConsumer)` 方法里用 `consumer.apply().forRoutes()` 声明的中间件注册到 Express / Fastify 实例，实际使用 `expressApp.use(...)`。

### 注册路由

```ts
public async registerRouter() {
  await this.registerMiddleware(this.httpAdapter);

  const prefix = this.config.getGlobalPrefix();
  const basePath = addLeadingSlash(prefix);
  this.routesResolver.resolve(this.httpAdapter, basePath);
}
```

注册路由的流程中涉及到 `RoutesResolver` 和 `RouterExplorer` 两个类上方法的调用，调用链如下：

```ts
registerRouter()
├── 获取全局 basePath
└──  RoutesResolver.resolve()
     ├── 获取模块路径
     └── RoutesResolver.registerRouters()
         ├── RouterExplorer.extractRouterPath() 获取 controller 路径
         └── RouterExplorer.explore() 扫描路由处理程序中的路径并注册
```

下面来看一下 `RoutesResolver` 和 `RouterExplorer` 上几个方法的实现：

#### `RoutesResolver`

##### `resolve` 方法

```ts
public resolve<T extends HttpServer>(
  applicationRef: T,
  globalPrefix: string,
) {
  const modules = this.container.getModules();
  modules.forEach(({ controllers, metatype }, moduleName) => {
    const modulePath = this.getModulePathMetadata(metatype)!;
    this.registerRouters(
      controllers,
      moduleName,
      globalPrefix,
      modulePath,
      applicationRef,
    );
  });
}
```

1. 从 `NestContainer` 获取所有模块并遍历。

2. 获取模块路径

- 用 `RouterModule.register([{ path: 'users', module: UsersModule }])` 方式指定的模块路径，本质上是通过 `Reflect.defineMetadata()` 定义在模块类上。

```ts
private registerModulePathMetadata(
  moduleCtor: Type<unknown>,
  modulePath: string,
) {
  Reflect.defineMetadata(
    MODULE_PATH + this.modulesContainer.applicationId,
    modulePath,
    moduleCtor,
  );
}
```

- 所以 `getModulePathMetadata` 方法中可以通过 `Reflect.getMetadata()` 获取。

```ts
private getModulePathMetadata(metatype: Type<unknown>): string | undefined {
  const modulesContainer = this.container.getModules();
  const modulePath = Reflect.getMetadata(
    MODULE_PATH + modulesContainer.applicationId,
    metatype,
  );
  return modulePath ?? Reflect.getMetadata(MODULE_PATH, metatype);
}
```

3. 从模块实例上获取模块中所有 controllers，调用 `RoutesResolver` 类的 `registerRouters()` 方法。

##### `registerRouters` 方法

```ts
public registerRouters(
  routes: Map<string | symbol | Function, InstanceWrapper<Controller>>,
  moduleName: string,
  globalPrefix: string,
  modulePath: string,
  applicationRef: HttpServer,
) {
  routes.forEach(instanceWrapper => {
    const { metatype } = instanceWrapper;

    const host = this.getHostMetadata(metatype!);
    const routerPaths = this.routerExplorer.extractRouterPath(
      metatype as Type<any>,
    );
    const controllerVersion = this.getVersionMetadata(metatype!);
    const controllerName = metatype!.name;

    routerPaths.forEach(path => {
      // ......
      const versioningOptions = this.applicationConfig.getVersioning();
      const routePathMetadata: RoutePathMetadata = {
        ctrlPath: path,
        modulePath,
        globalPrefix,
        controllerVersion,
        versioningOptions,
      };
      this.routerExplorer.explore(
        instanceWrapper,
        moduleName,
        applicationRef,
        host!,
        routePathMetadata,
      );
    });
  });
}
```

1. 遍历模块中的 controllers，对每一个控制器调用 `RouterExplorer` 类的 `extractRouterPath()` 方法获取 `@Controller()` 装饰器中定义的控制器路径。

2. 调用 `RouterExplorer` 类的 `explore()` 方法，扫描控制器中的路由处理函数，获取路由方法、合并路径，并将它们注册到应用中。

#### `RouterExplorer`

##### `extractRouterPath` 方法

```ts
public extractRouterPath(metatype: Type<Controller>): string[] {
  const path = Reflect.getMetadata(PATH_METADATA, metatype);

  if (isUndefined(path)) {
    throw new UnknownRequestMappingException(metatype);
  }
  if (Array.isArray(path)) {
    return path.map(p => addLeadingSlash(p));
  }
  return [addLeadingSlash(path)];
}
```

之前讲过[@Controller() 装饰器的源码](https://juejin.cn/post/7500981295105458215#heading-34)，本质上是通过  `Reflect.defineMetadata()`  将 path 等信息注入目标类上。

所以，这里可以通过 `Reflect.getMetadata()` 获取 `@Controller()` 装饰器定义的路径元信息。

##### `explore` 方法

```ts
public explore<T extends HttpServer = any>(
  instanceWrapper: InstanceWrapper,
  moduleKey: string,
  applicationRef: T,
  host: string | RegExp | Array<string | RegExp>,
  routePathMetadata: RoutePathMetadata,
) {
  const { instance } = instanceWrapper;
  const routerPaths = this.pathsExplorer.scanForPaths(instance);
  this.applyPathsToRouterProxy(
    applicationRef,
    routerPaths,
    instanceWrapper,
    moduleKey,
    routePathMetadata,
    host,
  );
}
```

**参数**：

- `instanceWrapper`: 控制器的 `InstanceWrapper` 实例。

- `moduleKey`: 控制器所在模块的唯一标识。

- `applicationRef`: 对应底层 HTTP 服务器的封装，http adapter。

- `host`: 可选的路由 host 过滤器（比如 `@Host('example.com')`）。

- `routePathMetadata`: 路由 path 元数据，包含模块路径、控制器路径。

**核心流程**：

**1. 从控制器 `InstanceWrapper` 上获取控制器实例**

在完成[依赖实例化](url)步骤后，`InstanceWrapper` 上已经可以获取实例化后的 `instance`。之后可以从这个实例上获取路由处理方法。

**2. 调用 `PathsExplorer` 类的 `scanForPaths` 方法扫描路由处理函数生成路由定义**

```ts
public scanForPaths(
  instance: Controller,
  prototype?: object,
): RouteDefinition[] {
  const instancePrototype = isUndefined(prototype)
    ? Object.getPrototypeOf(instance)
    : prototype;

  return this.metadataScanner
    .getAllMethodNames(instancePrototype)
    .reduce((acc, method) => {
      const route = this.exploreMethodMetadata(
        instance,
        instancePrototype,
        method,
      );

      if (route) {
        acc.push(route);
      }

      return acc;
    }, [] as RouteDefinition[]);
}
```

调用 `getAllMethodNames()` 方法获取控制器类原型上的所有方法名。

对每一个方法名调用 `exploreMethodMetadata()` 方法，通过 `Reflect.getMetadata()` 获取该方法上通过 `@Get()` 、`@Post()` 等装饰器定义的**路径**、**请求方法**、**请求处理函数**等元数据。

```ts
public exploreMethodMetadata(
  instance: Controller,
  prototype: object,
  methodName: string,
): RouteDefinition | null {
  const instanceCallback = instance[methodName];
  const prototypeCallback = prototype[methodName];
  const routePath = Reflect.getMetadata(PATH_METADATA, prototypeCallback);
  if (isUndefined(routePath)) {
    return null;
  }
  const requestMethod: RequestMethod = Reflect.getMetadata(
    METHOD_METADATA,
    prototypeCallback,
  );
  const version: VersionValue | undefined = Reflect.getMetadata(
    VERSION_METADATA,
    prototypeCallback,
  );
  const path = isString(routePath)
    ? [addLeadingSlash(routePath)]
    : routePath.map((p: string) => addLeadingSlash(p));

  return {
    path,
    requestMethod,
    targetCallback: instanceCallback,
    methodName,
    version,
  };
}
```

最终生成的路由定义 `routerPaths` 是一个数组，结构大致如下：

```ts
[
  {
    methodName: 'findAll',
    path: '/api/users',
    requestMethod: RequestMethod.GET,
    targetCallback: Function,
    methodRef: Function,
    ...
  },
  ...
]
```

**3. 调用 `applyPathsToRouterProxy()` 方法**

```ts
public applyPathsToRouterProxy<T extends HttpServer>(
  router: T,
  routeDefinitions: RouteDefinition[],
  instanceWrapper: InstanceWrapper,
  moduleKey: string,
  routePathMetadata: RoutePathMetadata,
  host: string | RegExp | Array<string | RegExp>,
) {
  (routeDefinitions || []).forEach(routeDefinition => {
    const { version: methodVersion } = routeDefinition;
    routePathMetadata.methodVersion = methodVersion;

    this.applyCallbackToRouter(
      router,
      routeDefinition,
      instanceWrapper,
      moduleKey,
      routePathMetadata,
      host,
    );
  });
}
```

遍历 `routerPaths` 数组，对每一个路由定义调用 `applyCallbackToRouter` 方法，其中：

- 合并路径：合并模块、控制器和路由处理方法上定义的 path。

- 注册路由：将路由处理函数绑定到 HTTP 服务器的路由系统中，比如注册到 `express.get('/api/users', handler)`。

> `applyPathsToRouterProxy` 方法在 Nest 中起到了桥梁的作用，将控制器中定义的路由处理函数注册到底层的 HTTP 服务器中，确保当接收到相应的 HTTP 请求时，能够正确地调用对应的处理函数。

### 触发 `onModuleInit` 生命周期钩子

- 查找所有实现了 `OnModuleInit` 接口的类，调用其 `onModuleInit()` 方法。

- 用于执行业务模块中的初始化逻辑（比如连接数据库、初始化缓存等）。

### 注册 404 处理器和全局异常处理器

注册两个特殊的处理器：404 处理器和全局异常处理器。

```ts
public async registerRouterHooks() {
  this.routesResolver.registerNotFoundHandler();
  this.routesResolver.registerExceptionHandler();
}
```

通过 http adapter 调用 express adapter：

```ts
public registerNotFoundHandler() {
  const applicationRef = this.container.getHttpAdapterRef();
  const callback = <TRequest, TResponse>(req: TRequest, res: TResponse) => {
    const method = applicationRef.getRequestMethod(req);
    const url = applicationRef.getRequestUrl(req);
    throw new NotFoundException(`Cannot ${method} ${url}`);
  };
  const handler = this.routerExceptionsFilter.create({}, callback, undefined);
  const proxy = this.routerProxy.createProxy(callback, handler);
  applicationRef.setNotFoundHandler &&
    applicationRef.setNotFoundHandler(
      proxy,
      this.applicationConfig.getGlobalPrefix(),
    );
}
```

实际使用 `expressApp.use(...)`，express adapter 源码：`packages/platform-express/adapters/express-adapter.ts`。

```ts
public setNotFoundHandler(handler: Function, prefix?: string) {
  return this.use(handler);
}
```

### 触发 `onApplicationBootstrap` 生命周期钩子

- 调用所有实现了 `OnApplicationBootstrap` 接口的类的 `onApplicationBootstrap()` 方法。

- 用于执行在依赖注入全部完成后的一些业务启动逻辑。

### 返回 `this` 实例，允许链式调用

# 总结

本文介绍了 Nest 应用实例的创建，以及初始化应用时所做的事情，比较重要的有：

- 初始化 http adapter
- 注册中间件
- 注册路由
- 触发生命周期钩子

其实，注册路由处理函数时，即 `applyCallbackToRouter` 方法中还有做更多的事情，比如注册守卫、拦截器、管道、异常过滤器，并使它们能按照一定顺序调度执行。这些将在下一篇文章中讲解。
