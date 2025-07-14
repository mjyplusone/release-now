我们知道 Nest 项目有一个 main 入口文件，它最基本的内容如下。关键点是通过 `NestFactory.create()` 创建 `app` 实例，然后通过 `app.listen()` 监听端口。

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

那么，让我们从 `NestFactory` 开始了解 Nest 的**启动机制**吧。

# NestFactory

源码位置：`packages/core/nest-factory.ts`

```ts
export class NestFactoryStatic {
  public async create<T extends INestApplication = INestApplication>(
    moduleCls: IEntryNestModule,
    serverOrOptions?: AbstractHttpAdapter | NestApplicationOptions,
    options?: NestApplicationOptions
  ): Promise<T> {
    const [httpServer, appOptions] = this.isHttpServer(serverOrOptions!)
      ? [serverOrOptions, options]
      : [this.createHttpAdapter(), serverOrOptions];

    const applicationConfig = new ApplicationConfig();
    const container = new NestContainer(applicationConfig); // 1. 创建 IoC 容器
    const graphInspector = this.createGraphInspector(appOptions!, container);

    this.setAbortOnError(serverOrOptions, options);
    this.registerLoggerConfiguration(appOptions);

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
    const target = this.createNestInstance(instance);
    return this.createAdapterProxy<T>(target, httpServer);
  }
}

export const NestFactory = new NestFactoryStatic(); // 得到实例
```

`NestFactory` 的本质：

- `NestFactoryStatic` 类的实例。
- 具有 `create()` 方法。
- `NestFactory.create()` 实际就是调用 `NestFactoryStatic.prototype.create()`。

# create 流程

`create` 方法的核心流程如下：

## 创建 IoC 容器

```ts
const container = new NestContainer(applicationConfig);
```

`NestContainer` 是 Nest 中核心的 IoC 容器，负责模块管理、依赖关系注册等。

## initialize

`initialize` 方法主要有**依赖扫描**和**依赖实例化**两个作用。

```ts
private async initialize(
  module: any,
  container: NestContainer,
  graphInspector: GraphInspector,
  config = new ApplicationConfig(),
  options: NestApplicationContextOptions = {},
  httpServer: HttpServer | null = null,
) {
  UuidFactory.mode = options.snapshot
    ? UuidFactoryMode.Deterministic
    : UuidFactoryMode.Random;

  const injector = new Injector({ preview: options.preview! });
  // 1. 创建实例加载器
  const instanceLoader = new InstanceLoader(
    container,
    injector,
    graphInspector,
  );
  const metadataScanner = new MetadataScanner();
  // 2.1 依赖扫描
  const dependenciesScanner = new DependenciesScanner(
    container,
    metadataScanner,
    graphInspector,
    config,
  );
  container.setHttpAdapter(httpServer);

  const teardown = this.abortOnError === false ? rethrow : undefined;
  await httpServer?.init?.();
  try {
    this.logger.log(MESSAGES.APPLICATION_START);

    await ExceptionsZone.asyncRun(
      async () => {
        // 2.2 依赖扫描
        await dependenciesScanner.scan(module);
        // 3. 依赖实例化
        await instanceLoader.createInstancesOfDependencies();
        dependenciesScanner.applyApplicationProviders();
      },
      teardown,
      this.autoFlushLogs,
    );
  } catch (e) {
    this.handleInitializationError(e);
  }
}
```

### 创建实例加载器

```ts
const instanceLoader = new InstanceLoader(container, injector, graphInspector);
```

将之前创建的 `NestContainer` 容器作为参数传给实例加载器，之后可以用这个加载器为 `NestContainer` 中注册的依赖创建实例。

### 依赖扫描

```ts
const dependenciesScanner = new DependenciesScanner(
  container,
  metadataScanner,
  graphInspector,
  config
);
await dependenciesScanner.scan(module);
```

以用户在 `NestFactory.create()` 传入的根模块为入口，递归寻找依赖（比如分析其 `@Module` 装饰器上的 `imports`, `providers`, `controllers`, `exports` 等字段），构建依赖图，存储到 `NestContainer` 中。

### 实例化依赖

```ts
await instanceLoader.createInstancesOfDependencies();
```

Nest 中通过构造函数注入依赖，Typescript 配合 reflect-metadata 库，可以获取类构造函数的参数类型。用其作为 token 在 `NestContainer` 中找到对应的类，即为当前类依赖的类。然后深度优先遍历，逐步实例化依赖。

## 创建应用实例

```ts
const instance = new NestApplication(
  container,
  httpServer,
  applicationConfig,
  graphInspector,
  appOptions
);
```

- 通过 `new NestApplication()` 创建应用实例。
- `NestApplication` 实现了 `INestApplication` 接口，其中封装了底层平台（如 Express / Fastify）相关的逻辑，实现了 `listen` 方法，以及全局拦截器、守卫、过滤器等注册逻辑。

# listen 方法

`listen` 是 `NestApplication` 类上的方法，其中主要干了两件事：

1. 调用 `NestApplication` 类上的 `init` 方法。

2. 通过 `httpAdapter` 调用底层的 `express.listen()` 或 `fastify.listen()` 启动服务。

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

然后看下 `init` 方法做了什么：

```ts
public async init(): Promise<this> {
  if (this.isInitialized) {
    return this;
  }

  this.applyOptions(); // 1. 应用配置项
  await this.httpAdapter?.init?.(); // 2. 初始化 http 适配器

  const useBodyParser =
    this.appOptions && this.appOptions.bodyParser !== false;
  useBodyParser && this.registerParserMiddleware(); // 3. 注册 body-parser 中间件

  await this.registerModules(); // 4. 注册子系统模块
  await this.registerRouter(); // 5. 注册路由
  await this.callInitHook(); // 6. 触发 onModuleInit 生命周期钩子
  await this.registerRouterHooks(); // 7. 注册两个特殊的处理器：404处理器和全局异常处理器
  await this.callBootstrapHook(); // 8. 触发 onApplicationBootstrap 生命周期钩子

  this.isInitialized = true;
  this.logger.log(MESSAGES.APPLICATION_READY);
  return this;
}
```

1. 应用配置项，比如 cors。

2. 初始化底层 http 适配器。

`httpAdapter` 是对 Express / Fastify 的封装，默认使用 Express（除非显式传入 adapter）。

3. 注册 body-parser 中间件

4. 注册子系统模块
   包括：

- 初始化并注册 WebSocket 模块。
- 初始化微服务系统：注册策略、代理、客户端等。
- 注册用户定义的中间件：调用自定义中间件里的 `configure(consumer: MiddlewareConsumer)` 方法，把你用 `consumer.apply().forRoutes()` 声明的中间件注册到 Express / Fastify 实例。

5. 注册路由

```ts
public async registerRouter() {
  await this.registerMiddleware(this.httpAdapter);

  const prefix = this.config.getGlobalPrefix();
  const basePath = addLeadingSlash(prefix);
  this.routesResolver.resolve(this.httpAdapter, basePath);
}
```

调用 `RoutesResolver` ，其中又调用 `RouterExplorer`，扫描所有 `@Controller()` 装饰器的路由元数据，生成路由映射，并注册到 http 适配器。

6. 触发 `onModuleInit` 生命周期钩子

7. 注册两个特殊的处理器：404 处理器和全局异常处理器

```ts
public async registerRouterHooks() {
  this.routesResolver.registerNotFoundHandler();
  this.routesResolver.registerExceptionHandler();
}
```

8. 触发 `onApplicationBootstrap` 生命周期钩子

# 总结

本章从整体视角分析了 Nest 启动流程的源码，其中很多步骤仅描述了大致思路，没有仔细探究其实现方式，比如 NestContainer、依赖扫描和实例化、路由注册、生命周期钩子等，后续的文章中会展开分析这些核心模块。
