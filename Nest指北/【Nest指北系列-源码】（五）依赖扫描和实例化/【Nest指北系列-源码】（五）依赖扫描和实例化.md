> 前情回顾：
>
> [【Nest 指北系列-源码】（一）目录结构](https://juejin.cn/post/7498280232764751908)
>
> [【Nest 指北系列-源码】（二）装饰器和 reflect-metadata](https://juejin.cn/post/7500981295105458215)
>
> [【Nest 指北系列-源码】（三）启动流程](https://juejin.cn/post/7503390744385454091)：整体视角讲述 Nest 启动流程：创建 NestContainer 容器、依赖扫描和实例化、创建应用实例
>
> [【Nest 指北系列-源码】（四）NestContainer](https://juejin.cn/post/7510415261741531188)：启动流程第一步：创建 NestContainer 容器

> 长长长长长文预警~~~

本章开始拆解 Nest 启动流程的第二个大步骤：依赖扫描和实例化。我们从 `NestFactory.create()` 源码的 `initialize` 方法进入。w

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
  }
}
```

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
  // 1.1 创建依赖注入器
  const injector = new Injector({ preview: options.preview! });
  // 1.2 创建实例加载器
  const instanceLoader = new InstanceLoader(
    container,
    injector,
    graphInspector,
  );
  // 2.1 依赖扫描
  const metadataScanner = new MetadataScanner();
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

`initialize` 方法中主要做了以下几件事：

# 创建依赖注入器和实例加载器

```ts
const injector = new Injector({ preview: options.preview! });
const instanceLoader = new InstanceLoader(container, injector, graphInspector);
```

`Injector` 和 `InstanceLoader` 都是 Nest 中的核心工具类：

## Injector 类

Nest 内部工具类，提供了一系列关键方法，用于解析依赖关系和创建实例，即完成「**注入**」动作。

```ts
export class Injector {
  public loadPrototype() {}

  public loadProvider() {}

  public loadInjectable() {}

  public loadController() {}

  public async loadInstance() {}

  public async resolveConstructorParams() {}
}
```

## InstanceLoader 类

Nest 内部工具类，负责遍历所有模块和其中的 provider、controller 等，调用 `Injector` 类提供的方法进行实例化。即调用链为 `InstanceLoader` -> `Injector`。

> 这两个类的实现暂且跳过，后续依赖实例化流程会使用到它们提供的方法，到时候再来讲解。

# 依赖扫描

```ts
// 创建元数据扫描器和依赖扫描器
const metadataScanner = new MetadataScanner();
const dependenciesScanner = new DependenciesScanner(
  container,
  metadataScanner,
  graphInspector,
  config
);

// 开始扫描
await ExceptionsZone.asyncRun(async () => {
  await dependenciesScanner.scan(module);
});
```

先简单看下依赖扫描的流程：

**1. 创建元数据扫描器和依赖扫描器**

- `MetadataScanner`：提供 `getAllMethodNames` 方法，用于扫描类原型链上所有的方法，后续流程中会用到。

- `DependenciesScanner`：提供多个方法串起依赖扫描的流程，比如：
  - 扫描和注册模块
  - 扫描和注册模块上依赖的 imports、providers、controllers、exports
  - 扫描和注册 provider 和 controller 依赖的 injectables

> 提前预告一下：这些方法的调用链都是这样的形式：`DependenciesScanner` -> `NestContainer` -> `Module`，后面会具体讲到。

**2. 调用 `DependenciesScanner` 类的 `scan` 扫描方法**

`scan` 方法使用 Nest 统一的异常处理包装器 `ExceptionsZone.asyncRun` 包装。简化后的核心逻辑如下：

```ts
public async scan(
  module: Type<any>,
  options?: { overrides?: ModuleOverride[] },
) {
  await this.scanForModules({
    moduleDefinition: module,
    overrides: options?.overrides,
  });
  await this.scanModulesForDependencies();

  this.calculateModulesDistance();

  this.container.bindGlobalScope();
}
```

**3. 执行 `scanForModules`**

以用户传入的根模块为入口，递归扫描模块，注册到 `NestContainer` 容器的 `modules` 结构中。

**4. 执行 `scanModulesForDependencies`**

遍历 `scanForModules` 注册在 `NestContainer` 中的每个模块，扫描它依赖的 imports、exports、providers、controllers，对于 provider 和 controller 额外扫描它们依赖的 injectables，注册在模块中，形成**模块依赖图**。

**5. 执行 `calculateModulesDistance`**

计算注册的各个模块相对于根模块的依赖深度。

**6. 执行 `bindGlobalScope`**

把所有使用 `@Global()` 装饰器声明的模块，自动注入进所有其他模块的 `_imports` 中，使其中注册的 providers 在全局可用。

---

下面我们具体分析依赖扫描流程中的几个核心方法：

## scanForModules

`DependenciesScanner` 类提供的工具方法。

```ts
public async scanForModules({
  moduleDefinition,
  lazy,
  scope = [],
  ctxRegistry = [],
  overrides = [],
}: ModulesScanParameters): Promise<Module[]> {
  const { moduleRef: moduleInstance, inserted: moduleInserted } =
    (await this.insertOrOverrideModule(moduleDefinition, overrides, scope)) ??
    {};

  // ......
  ctxRegistry.push(moduleDefinition);
  // ......

  const modules = !this.isDynamicModule(
    moduleDefinition as Type<any> | DynamicModule,
  )
    ? this.reflectMetadata(
        MODULE_METADATA.IMPORTS,
        moduleDefinition as Type<any>,
      )
    : [
        ...this.reflectMetadata(
          MODULE_METADATA.IMPORTS,
          (moduleDefinition as DynamicModule).module,
        ),
        ...((moduleDefinition as DynamicModule).imports || []),
      ];

  let registeredModuleRefs: Module[] = [];
  for (const [index, innerModule] of modules.entries()) {
    // ......
    if (ctxRegistry.includes(innerModule)) {
      continue;
    }
    const moduleRefs = await this.scanForModules({
      moduleDefinition: innerModule,
      scope: ([] as Array<Type>).concat(scope, moduleDefinition as Type),
      ctxRegistry,
      overrides,
      lazy,
    });
    registeredModuleRefs = registeredModuleRefs.concat(moduleRefs);
  }
  if (!moduleInstance) {
    return registeredModuleRefs;
  }
  // ......
  return [moduleInstance].concat(registeredModuleRefs);
}
```

**参数**：

- `moduleDefinition`: 待扫描的模块，可以是 `@Module()` 装饰的普通模块类，也可以是动态模块对象。

- `ctxRegistry`: 已处理模块的记录，防止重复扫描。

**核心逻辑**：

**1. 插入模块**

调用 `DependenciesScanner` 类提供的方法扫描并注册模块，调用链为：`dependenciesScanner.insertOrOverrideModule()` -> `dependenciesScanner.insertModule()` -> `container.addModule()`。

即，最终调用 `NestContainer` 类上的 [addModule](https://juejin.cn/post/7510415261741531188#heading-10) 方法将传入的待扫描模块注册到 `NestContainer` 实例上的 `modules` Map 结构中，然后返回模块实例。

```ts
private insertOrOverrideModule(
  moduleDefinition: ModuleDefinition,
  overrides: ModuleOverride[],
  scope: Type<unknown>[],
): Promise<
  | {
      moduleRef: Module;
      inserted: boolean;
    }
  | undefined
> {
  // ......
  return this.insertModule(moduleDefinition, scope);
}
```

```ts
public async insertModule(
  moduleDefinition: any,
  scope: Type<unknown>[],
): Promise<
  | {
      moduleRef: Module;
      inserted: boolean;
    }
  | undefined
> {
  // ......
  return this.container.addModule(moduleToAdd, scope);
}
```

**2. 通过 reflect-metadata 库提取模块 `imports` 的子模块**

```ts
const modules = !this.isDynamicModule(moduleDefinition)
  ? this.reflectMetadata(MODULE_METADATA.IMPORTS, moduleDefinition)
  : [
      ...this.reflectMetadata(MODULE_METADATA.IMPORTS, moduleDefinition.module),
      ...(moduleDefinition.imports || []),
    ];
```

```ts
public reflectMetadata<T = any>(
  metadataKey: string,
  metatype: Type<any>,
): T[] {
  return Reflect.getMetadata(metadataKey, metatype) || [];
}
```

- 如果是静态模块：直接从 `@Modules()` 装饰器定义的元数据中提取 `imports`。

- 如果是动态模块：需要将 `@Modules()` 装饰器定义的 `imports` 和动态模块对象中定义的 `imports` 合并。

这里涉及到[【Nest 指北系列-源码】（二）装饰器和 reflect-metadata](https://juejin.cn/post/7500981295105458215)中提到的点，`@Modules()` 装饰器的实现中会调用  `Reflect.defineMetadata` 将传入的模块描述对象里定义的  `imports`、`providers`、`controllers`  等作为元数据注入到模块类上。所以这里可以通过 `Reflect.getMetadata` 在模块类上获取。

**3. 递归扫描子模块**

```ts
ctxRegistry.push(moduleDefinition);
let registeredModuleRefs: Module[] = [];
for (const [index, innerModule] of modules.entries()) {
  // ......
  if (ctxRegistry.includes(innerModule)) {
    continue;
  }
  const moduleRefs = await this.scanForModules({
    moduleDefinition: innerModule,
    scope: ([] as Array<Type>).concat(scope, moduleDefinition as Type),
    ctxRegistry,
    overrides,
    lazy,
  });
  registeredModuleRefs = registeredModuleRefs.concat(moduleRefs);
}
```

遍历模块 `imports` 的子模块，递归调用 `scanForModules` 扫描子模块，将子模块也注册到  `NestContainer`  实例上的  `modules` Map 结构中。

过程中会通过 `ctxRegistry` 参数判断是否已经处理过该模块（处理过的模块会放入 `ctxRegistry` 中，并传入扫描子模块的 `scanForModules` 方法），防止重复扫描和无限循环。

**4. 返回包含依赖树中所有模块实例的数组**

```ts
return [moduleInstance].concat(registeredModuleRefs);
```

## scanModulesForDependencies

`DependenciesScanner` 类提供的工具方法。

```ts
public async scanModulesForDependencies(
  modules: Map<string, Module> = this.container.getModules(),
) {
  for (const [token, { metatype }] of modules) {
    await this.reflectImports(metatype, token, metatype.name);
    this.reflectProviders(metatype, token);
    this.reflectControllers(metatype, token);
    this.reflectExports(metatype, token);
  }
}
```

**参数**：

- `modules`：不传入时默认为 `this.container.getModules()`，即 `scanForModules` 注册在 `NestContainer` 上的所有模块。

**核心逻辑**：

1. 遍历 `modules` Map 结构，获取模块 token 和模块实例上的 `metatype` 属性（即模块类本身）。

2. 对每一个模块依次执行 `reflectImports`、`reflectProviders`、`reflectControllers`、`reflectExports`。

### reflectImports

```ts
public async reflectImports(
  module: Type<unknown>,
  token: string,
  context: string,
) {
  const modules = [
    ...this.reflectMetadata(MODULE_METADATA.IMPORTS, module),
    ...this.container.getDynamicMetadataByToken(
      token,
      MODULE_METADATA.IMPORTS as 'imports',
    )!,
  ];
  for (const related of modules) {
    await this.insertImport(related, token, context);
  }
}
```

```ts
public getDynamicMetadataByToken<
  K extends Exclude<keyof DynamicModule, 'global' | 'module'>,
>(token: string, metadataKey: K): DynamicModule[K];
public getDynamicMetadataByToken(
  token: string,
  metadataKey?: Exclude<keyof DynamicModule, 'global' | 'module'>,
) {
  const metadata = this.dynamicModulesMetadata.get(token);
  return metadataKey ? (metadata?.[metadataKey] ?? []) : metadata;
}
```

通过 `Reflect.getMetadata` 在模块类上获取 `@Modules()` 装饰器定义的 `imports`，通过 `NestContainer` 的 [dynamicModulesMetadata](https://juejin.cn/post/7510415261741531188#heading-4) 属性获取动态模块对象中定义的 `imports`。合并得到当前模块依赖的所有子模块。

遍历依赖的子模块，依次调用 `dependenciesScanner.insertImport()` -> `container.addImport()`。

即，最终调用 `NestContainer` 类上的 [addImport](https://juejin.cn/post/7510415261741531188#heading-19) 方法将子模块的实例添加到当前模块的 `_imports`  集合中，形成模块依赖关系。

### reflectProviders

```ts
public reflectProviders(module: Type<any>, token: string) {
  const providers = [
    ...this.reflectMetadata(MODULE_METADATA.PROVIDERS, module),
    ...this.container.getDynamicMetadataByToken(
      token,
      MODULE_METADATA.PROVIDERS as 'providers',
    )!,
  ];
  providers.forEach(provider => {
    this.insertProvider(provider, token);
    this.reflectDynamicMetadata(item, token);
  });
}
```

通过 `Reflect.getMetadata` 在模块类上获取 `@Modules()` 装饰器定义的 `providers`，通过 `NestContainer` 的 [dynamicModulesMetadata](https://juejin.cn/post/7510415261741531188#heading-4) 属性获取动态模块对象中定义的 `providers`。合并得到当前模块依赖的所有 `providers`。

遍历依赖的 `providers`，依次调用 `DependenciesScanner` 类提供的方法 `dependenciesScanner.insertProvider()` -> `container.addProvider()`。即，最终调用 `NestContainer` 类上的 [addProvider](https://juejin.cn/post/7510415261741531188#heading-15) 方法将依赖的 provider 注册到模块的  `_providers`  中，形成 `Map<token, InstanceWrapper>` 结构。

### reflectControllers

```ts
public reflectControllers(module: Type<any>, token: string) {
  const controllers = [
    ...this.reflectMetadata(MODULE_METADATA.CONTROLLERS, module),
    ...this.container.getDynamicMetadataByToken(
      token,
      MODULE_METADATA.CONTROLLERS as 'controllers',
    )!,
  ];
  controllers.forEach(item => {
    this.insertController(item, token);
    this.reflectDynamicMetadata(item, token);
  });
}
```

和 `reflectProviders` 的实现类似，将模块依赖的 controller 注册到模块的 `_controllers` 中，形成 `Map<token, InstanceWrapper>` 结构。

但这里需要额外关注一个方法 `dependenciesScanner.reflectDynamicMetadata`，它调用`dependenciesScanner.reflectInjectables` 将 controller 依赖的 guards、interceptors、exceptionFilters、pipes 注册到模块的 `_injectables` 中，形成 `Map<token, InstanceWrapper>` 结构。具体实现如下：

```ts
public reflectDynamicMetadata(cls: Type<Injectable>, token: string) {
  if (!cls || !cls.prototype) {
    return;
  }
  this.reflectInjectables(cls, token, GUARDS_METADATA);
  this.reflectInjectables(cls, token, INTERCEPTORS_METADATA);
  this.reflectInjectables(cls, token, EXCEPTION_FILTERS_METADATA);
  this.reflectInjectables(cls, token, PIPES_METADATA);
  this.reflectParamInjectables(cls, token, ROUTE_ARGS_METADATA);
}
```

#### reflectInjectables

```ts
public reflectInjectables(
  component: Type<Injectable>,
  token: string,
  metadataKey: string,
) {
  const controllerInjectables = this.reflectMetadata<Type<Injectable>>(
    metadataKey,
    component,
  );
  const methodInjectables = this.metadataScanner
    .getAllMethodNames(component.prototype)
    .reduce(
      (acc, method) => {
        const methodInjectable = this.reflectKeyMetadata(
          component,
          metadataKey,
          method,
        );

        if (methodInjectable) {
          acc.push(methodInjectable);
        }

        return acc;
      },
      [] as Array<{
        methodKey: string;
        metadata: Type<Injectable>[];
      }>,
    );

  controllerInjectables.forEach(injectable =>
    this.insertInjectable(
      injectable,
      token,
      component,
      ENHANCER_KEY_TO_SUBTYPE_MAP[metadataKey],
    ),
  );
  methodInjectables.forEach(methodInjectable => {
    methodInjectable.metadata.forEach(injectable =>
      this.insertInjectable(
        injectable,
        token,
        component,
        ENHANCER_KEY_TO_SUBTYPE_MAP[metadataKey],metadata key
        methodInjectable.methodKey,
      ),
    );
  });
}
```

**参数**：

- `component`：控制器类。

- `token`：模块 token。

- `metadataKey`：元数据 key，比如 `"__guards__"`，用于从类或方法上读取装饰器定义的元信息。

**核心逻辑**：

1. 通过 `Reflect.getMetadata` 读取定义在控制器类上的 injectables，放入 `controllerInjectables` 数组。

2. 通过 `MetadataScanner` 类提供的 `getAllMethodNames` 方法读取控制器类原型上的所有方法名。遍历方法名，依次通过 `Reflect.getMetadata` 读取定义在方法上的 injectables，放入 `methodInjectables` 数组。

3. 遍历 `controllerInjectables` 和 `methodInjectables`，依次调用 `DependenciesScanner` 类提供的方法 `dependenciesScanner.insertInjectable` -> `container.addInjectable()`。即，最终调用 `NestContainer` 类上的 [addInjectable](https://juejin.cn/post/7510415261741531188#heading-35) 方法将依赖的 injectable 注册到模块的  `_injectables`  中，形成 `Map<token, InstanceWrapper>` 结构。

### reflectExports

和 `reflectImports` 的实现类似，将模块定义的 exports 注册到模块的 `_exports` 集合中。

> 即，`scanModulesForDependencies` 就是通过 reflect-metadata 读取每个模块 `@Module` 装饰器定义的 `imports`、`exports`、`providers`、`controllers` 元信息，对于 provider 和 controller 额外扫描它们依赖的 injectables，然后将它们都注册到模块中，形成应用的模块依赖图。

### Nest 中复用实例的理解

这个系列前面的文章中，介绍 controller、provider、injectables 等的使用时，常提到「**复用同一个实例**」这个说法，看完上面依赖扫描的源码，应该可以更好的理解这个说法了：

- 依赖最终都注册到 `NestContainer` 的 `modules` 中，Map 结构可以保证每个模块只注册一次。`Module` 类中的 `_controllers`、`_providers`、`_injectables` 也都是 Map 结构，就是同一个模块中，相同 token 的依赖也只会注册一次。

- 不同模块中，通过 `imports` 导入相同 token 的模块是同一个实例，再通过 token 去寻找模块中的 provider 也是同一个实例。

  ```ts
  import { Module } from "@nestjs/common";
  import { DogService } from "./dog.service";
  import { DogController } from "./dog.controller";
  import { UserModule } from "src/user/user.module";

  @Module({
    imports: [UserModule],
    providers: [DogService], // 其中使用 UserService，同一个实例
    controllers: [DogController],
  })
  export class DogModule {}

  import { Module } from "@nestjs/common";
  import { CatController } from "./cat.controller";
  import { CatService } from "./cat.service";
  import { UserModule } from "src/user/user.module";

  @Module({
    imports: [UserModule],
    providers: [CatService], // 其中使用 UserService，同一个实例
    controllers: [CatController],
  })
  export class CatModule {}
  ```

- 但如果不是通过 `imports` 公共模块的方式，而是直接在当前模块的 providers 中注册，那么是不同实例。

  ```ts
  import { Module } from "@nestjs/common";
  import { DogService } from "./dog.service";
  import { DogController } from "./dog.controller";
  import { UserService } from "../user/user.service";

  @Module({
    providers: [DogService, UserService],
    controllers: [DogController],
  })
  export class DogModule {}

  import { Module } from "@nestjs/common";
  import { CatController } from "./cat.controller";
  import { CatService } from "./cat.service";
  import { UserService } from "../user/user.service";

  @Module({
    providers: [CatService, UserService],
    controllers: [CatController],
  })
  export class CatModule {}
  ```

## calculateModulesDistance

`DependenciesScanner` 类提供的工具方法。计算各模块相对于根模块的依赖深度，并将这个距离保存在每个模块的 [distance](https://juejin.cn/post/7510415261741531188#heading-30) 字段上。

```ts
public calculateModulesDistance() {
  const modulesGenerator = this.container.getModules().values();

  // Skip "InternalCoreModule" from calculating distance
  modulesGenerator.next();

  const calculateDistance = (
    moduleRef: Module,
    distance = 1,
    modulesStack: Module[] = [],
  ) => {
    const localModulesStack = [...modulesStack];
    if (!moduleRef || localModulesStack.includes(moduleRef)) {
      return;
    }
    localModulesStack.push(moduleRef);

    const moduleImports = moduleRef.imports;
    moduleImports.forEach(importedModuleRef => {
      if (importedModuleRef) {
        if (
          distance > importedModuleRef.distance &&
          !importedModuleRef.isGlobal
        ) {
          importedModuleRef.distance = distance;
        }
        calculateDistance(importedModuleRef, distance + 1, localModulesStack);
      }
    });
  };

  const rootModule = modulesGenerator.next().value;
  calculateDistance(rootModule!);
}
```

**核心逻辑**：

1. 获取注册在 `NestContainer` 上的所有模块，并跳过内部核心模块 `InternalCoreModule`，从注册的第二个模块开始计算，即用户注册的**根模块**。

2. 从根模块开始，递归调用 `calculateDistance` 计算每个模块到根模块的距离。默认 `distance` 为 1，每递归一层 `distance + 1`，找到每个模块的 `imports`，如果当前 `distance` 大于 `imports` 模块上记录的 `distance`，且 `imports` 的模块不是全局模块，则更新模块的 `distance`。

   即，Nest 会优先保留**最长路径**作为最终的 `distance`。

> 这个**距离**会用于排序模块执行顺序（如中间件/生命周期钩子初始化）等场景。

## bindGlobalScope

`NestContainer` 类提供的工具方法。把所有使用 `@Global()` 装饰器声明的模块，自动注入进所有其他模块的 `_imports` 中，从而实现全局 provider 的可用性。

```ts
public bindGlobalScope() {
  this.modules.forEach(moduleRef => this.bindGlobalsToImports(moduleRef));
}

public bindGlobalsToImports(moduleRef: Module) {
  this.globalModules.forEach(globalModule =>
    this.bindGlobalModuleToModule(moduleRef, globalModule),
  );
}

public bindGlobalModuleToModule(target: Module, globalModule: Module) {
  if (target === globalModule || target === this.internalCoreModule) {
    return;
  }
  target.addImport(globalModule);
}
```

**核心逻辑**：

1. 遍历 `NestContainer` 容器中所有模块，对每个模块执行 `bindGlobalsToImports`。

2. `bindGlobalsToImports` 中遍历 `NestContainer` 容器中注册的所有全局模块，调用 `addImport` 将每一个全局模块注册到当前模块的 `_imports` 上。（注意跳过自身和内置核心模块）

> 思考一个问题：`bindGlobalScope` 在 `calculateModulesDistance` 之后，那么全局模块的 `distance` 是怎么设置的？
>
> 答案：全局模块的距离已在 [addModule](https://juejin.cn/post/7510415261741531188#heading-10) -> `setModule` 时设置为 MAX_VALUE。

---

> 至此，依赖扫描流程结束，进入依赖实例化部分。

# 依赖实例化

调用 `InstanceLoader` 实例加载器上的方法进行实例化。`instanceLoader.createInstancesOfDependencies()` 是 Nest 在启动阶段创建所有依赖实例的入口。

```ts
await instanceLoader.createInstancesOfDependencies();
```

```ts
public async createInstancesOfDependencies(
  modules: Map<string, Module> = this.container.getModules(),
) {
  this.createPrototypes(modules);

  try {
    await this.createInstances(modules);
  } catch (err) {
    this.graphInspector.inspectModules(modules);
    this.graphInspector.registerPartial(err);
    throw err;
  }
  this.graphInspector.inspectModules(modules);
}
```

下面我们具体分析依赖实例化流程中的几个核心方法：

## createPrototypes

`InstanceLoader` 类提供的工具方法。提前为模块的所有依赖类创建空的实例（未真正构造），用于解决循环依赖问题。

```ts
private createPrototypes(modules: Map<string, Module>) {
  modules.forEach(moduleRef => {
    this.createPrototypesOfProviders(moduleRef);
    this.createPrototypesOfInjectables(moduleRef);
    this.createPrototypesOfControllers(moduleRef);
  });
}
```

**参数**：

- `modules`：`scanForModules` 注册在 `NestContainer` 上的所有模块。

**核心逻辑**：

1. 遍历所有模块，依次调用 `createPrototypesOfProviders`、`createPrototypesOfInjectables`、`createPrototypesOfControllers`。

2. 其中，遍历模块实例上存储的所有 providers、injectables、controllers，对每一项调用 `Injector` 类上的 `loadPrototype`。

```ts
private createPrototypesOfProviders(moduleRef: Module) {
  const { providers } = moduleRef;
  providers.forEach(wrapper =>
    this.injector.loadPrototype<Injectable>(wrapper, providers),
  );
}
```

```ts
private createPrototypesOfInjectables(moduleRef: Module) {
  const { injectables } = moduleRef;
  injectables.forEach(wrapper =>
    this.injector.loadPrototype(wrapper, injectables),
  );
}
```

```ts
private createPrototypesOfControllers(moduleRef: Module) {
  const { controllers } = moduleRef;
  controllers.forEach(wrapper =>
    this.injector.loadPrototype<Controller>(wrapper, controllers),
  );
}
```

### loadPrototype

`Injector` 类提供的工具方法。

```ts
public loadPrototype<T>(
  { token }: InstanceWrapper<T>,
  collection: Map<InjectionToken, InstanceWrapper<T>>,
  contextId = STATIC_CONTEXT,
) {
  if (!collection) {
    return;
  }
  const target = collection.get(token)!;
  const instance = target.createPrototype(contextId);
  if (instance) {
    const wrapper = new InstanceWrapper({
      ...target,
      instance,
    });
    collection.set(token, wrapper);
  }
}
```

获取依赖的 `InstanceWrapper`，调用 `InstanceWrapper` 类上的 `createPrototype()` 方法，即调用 `Object.create()` **创建依赖类的原型实例**，然后将这个实例赋值到 `InstanceWrapper` 的 `instance` 属性上。

```ts
public createPrototype(contextId: ContextId) {
  const host = this.getInstanceByContextId(contextId);
  if (!this.isNewable() || host.isResolved) {
    return;
  }
  return Object.create(this.metatype!.prototype);
}
```

注意：**这里没有调用构造函数，所以没有进行真正的实例化，只是创建一个空对象，并设置其 `[[Prototype]]` 指向类的 `prototype`**。

这个对象将用于解决循环依赖问题，比如：

```ts
@Injectable()
class A {
  constructor(private b: B) {}
}

@Injectable()
class B {
  constructor(private a: A) {}
}
```

正常构造流程无法完成，因为 A 依赖 B，B 又依赖 A。而 `createPrototypes()` 先为 A 和 B 都创建一个 `Object.create(A.prototype)` 和 `Object.create(B.prototype)` 的空实例，这样 Nest 就可以在构造 A 时先拿到 B 的「壳」，反之亦然。

## createInstances

`InstanceLoader` 类提供的工具方法。用于真正构造实例。

```ts
private async createInstances(modules: Map<string, Module>) {
  await Promise.all(
    [...modules.values()].map(async moduleRef => {
      await this.createInstancesOfProviders(moduleRef);
      await this.createInstancesOfInjectables(moduleRef);
      await this.createInstancesOfControllers(moduleRef);

      const { name } = moduleRef;
      this.isModuleWhitelisted(name) &&
        this.logger.log(MODULE_INIT_MESSAGE`${name}`);
    }),
  );
}
```

**参数**：

- `modules`：`scanForModules` 注册在 `NestContainer` 上的所有模块。

**核心逻辑**：

1. 遍历所有模块，依次调用 `createInstancesOfProviders`、`createInstancesOfInjectables`、`createInstancesOfControllers`。

2. 其中，遍历模块实例上存储的所有 providers、injectables、controllers，对每一项分别调用 `Injector` 类上的 `loadProvider`、`loadInjectable`、`loadController`。

```ts
private async createInstancesOfProviders(moduleRef: Module) {
  const { providers } = moduleRef;
  const wrappers = [...providers.values()];
  await Promise.all(
    wrappers.map(async item => {
      await this.injector.loadProvider(item, moduleRef);
      this.graphInspector.inspectInstanceWrapper(item, moduleRef);
    }),
  );
}
```

```ts
private async createInstancesOfInjectables(moduleRef: Module) {
  const { injectables } = moduleRef;
  const wrappers = [...injectables.values()];
  await Promise.all(
    wrappers.map(async item => {
      await this.injector.loadInjectable(item, moduleRef);
      this.graphInspector.inspectInstanceWrapper(item, moduleRef);
    }),
  );
}
```

```ts
private async createInstancesOfControllers(moduleRef: Module) {
  const { controllers } = moduleRef;
  const wrappers = [...controllers.values()];
  await Promise.all(
    wrappers.map(async item => {
      await this.injector.loadController(item, moduleRef);
      this.graphInspector.inspectInstanceWrapper(item, moduleRef);
    }),
  );
}
```

3. `loadProvider`、`loadInjectable`、`loadController` 实现类似，这里重点分析 `loadProvider` 即可，其中调用 `Injector` 类上的 `loadInstance` 方法构造实例。

```ts
public async loadProvider(
  wrapper: InstanceWrapper<Injectable>,
  moduleRef: Module,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper,
) {
  const providers = moduleRef.providers;
  await this.loadInstance<Injectable>(
    wrapper,
    providers,
    moduleRef,
    contextId,
    inquirer,
  );
  await this.loadEnhancersPerContext(wrapper, contextId, wrapper);
}
```

### loadInstance

`Injector` 类提供的工具方法。从一个类开始，递归其注入的依赖类，逐个实例化。

```ts
public async loadInstance<T>(
  wrapper: InstanceWrapper<T>,
  collection: Map<InjectionToken, InstanceWrapper>,
  moduleRef: Module,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper,
) {
  const inquirerId = this.getInquirerId(inquirer);
  const instanceHost = wrapper.getInstanceByContextId(
    this.getContextId(contextId, wrapper),
    inquirerId,
  );

  if (instanceHost.isPending) {
    const settlementSignal = wrapper.settlementSignal;
    if (inquirer && settlementSignal?.isCycle(inquirer.id)) {
      throw new CircularDependencyException(`"${wrapper.name}"`);
    }

    return instanceHost.donePromise!.then((err?: unknown) => {
      if (err) {
        throw err;
      }
    });
  }

  const settlementSignal = this.applySettlementSignal(instanceHost, wrapper);
  const token = wrapper.token || wrapper.name;

  const { inject } = wrapper;
  const targetWrapper = collection.get(token);
  if (isUndefined(targetWrapper)) {
    throw new RuntimeException();
  }
  if (instanceHost.isResolved) {
    return settlementSignal.complete();
  }
  try {
    const callback = async (instances: unknown[]) => {
      // ......
      const instance = await this.instantiateClass(......);
      // ......
      settlementSignal.complete();
    };
    await this.resolveConstructorParams<T>(wrapper,
      moduleRef,
      inject as InjectionToken[],
      callback,
      contextId,
      wrapper,
      inquirer,
    );
  } catch (err) {
    // ......
  }
}
```

**参数**：

- `wrapper`：要实例化的类的 `InstanceWrapper` 实例。

- `collection`：要实例化的类所在的模块实例上注册的所有该类型的 `token -> InstanceWrapper` Map 映射。比如要实例化一个 provider，此处就是 `moduleRef._providers`，实例化一个 controller，此处就是 `moduleRef._controllers`。

- `moduleRef`：要实例化的类所在的模块实例。

- `contextId`：上下文 id，默认值固定为 STATIC_CONTEXT 支持默认作用域，传值时用于支持 REQUEST 和 TRANSIENT 作用域。

- `inquirer`：触发注入的调用者，用于处理循环依赖检测。

**核心逻辑**：

#### 1. 获取实例上下文，实现上下文隔离

从 `InstanceWrapper` 实例的 `values` 属性上获取 contextId 对应的实例对象。（`InstanceWrapper` 和它的 `values` 属性，上一篇中已讲过，具体可见[InstanceWrapper](url)）

```ts
const instanceHost = wrapper.getInstanceByContextId(
  this.getContextId(contextId, wrapper),
  inquirerId
);
```

#### 2. 追踪实例状态

这里涉及到实例对象上的三个属性：`isPending`、`donePromise`、`isResolved`。

1）首先通过 `isPending` 判断实例是否在构造中，如果在构造中，返回 `donePromise` 等待它构造完成再继续。

2）如果没有在构造中，开始构造之前，先通过 `injector.applySettlementSignal()` 生成 `SettlementSignal` 实例，它内部实现了一个 Promise，将这个 Promise 赋值给实例对象的 `donePromise`，且设置实例对象的 `isPending` 为 true，用于追踪实例构造状态。

```ts
public applySettlementSignal<T>(
  instancePerContext: InstancePerContext<T>,
  host: InstanceWrapper<T>,
) {
  const settlementSignal = new SettlementSignal();
  instancePerContext.donePromise = settlementSignal.asPromise();
  instancePerContext.isPending = true;
  host.settlementSignal = settlementSignal;

  return settlementSignal;
}
```

```ts
export class SettlementSignal {
  private readonly _refs = new Set();
  private readonly settledPromise: Promise<unknown>;
  private settleFn!: (err?: unknown) => void;
  private completed = false;

  constructor() {
    this.settledPromise = new Promise<unknown>((resolve) => {
      this.settleFn = resolve;
    });
  }

  public complete() {
    this.completed = true;
    this.settleFn();
  }

  public error(err: unknown) {
    this.completed = true;
    this.settleFn(err);
  }

  public asPromise() {
    return this.settledPromise;
  }
}
```

3）通过 `isResolved` 判断实例是否构造完成，构造完成则直接返回，**避免重复实例化**。

#### 3. resolveConstructorParams

调用 `injector.resolveConstructorParams()` 解析构造函数参数并实例化。

```ts
public async resolveConstructorParams<T>(
  wrapper: InstanceWrapper<T>,
  moduleRef: Module,
  inject: InjectorDependency[] | undefined,
  callback: (args: unknown[]) => void | Promise<void>,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper,
  parentInquirer?: InstanceWrapper,
) {
  // ......

  const isFactoryProvider = !isNil(inject);
  const [dependencies, optionalDependenciesIds] = isFactoryProvider
    ? this.getFactoryProviderDependencies(wrapper)
    : this.getClassDependencies(wrapper);

  let isResolved = true;
  const resolveParam = async (param: unknown, index: number) => {
    try {
      // ......
      const paramWrapper = await this.resolveSingleParam<T>(
        wrapper,
        param as Type | string | symbol,
        { index, dependencies },
        moduleRef,
        contextId,
        inquirer,
        index,
      );
      // ......
      return instanceHost?.instance;
    } catch (err) {
      // ......
    }
  };
  const instances = await Promise.all(dependencies.map(resolveParam));
  isResolved && (await callback(instances));
}
```

**核心逻辑**：

**1. 获取依赖 token 列表**

根据 InstanceWrapper 上是否有 inject 属性判断是：

- 工厂 provider（`useFactory` 定义）：调用 `getFactoryProviderDependencies` 解析依赖，即 `wrapper.inject` 的值。

- 类 provider：调用 `getClassDependencies` 解析依赖。

```ts
public getClassDependencies<T>(
  wrapper: InstanceWrapper<T>,
): [InjectorDependency[], number[]] {
  const ctorRef = wrapper.metatype as Type<any>;
  return [
    this.reflectConstructorParams(ctorRef),
    this.reflectOptionalParams(ctorRef),
  ];
}
```

```ts
public reflectConstructorParams<T>(type: Type<T>): any[] {
  const paramtypes = [
    ...(Reflect.getMetadata(PARAMTYPES_METADATA, type) || []),
  ];
  const selfParams = this.reflectSelfParams<T>(type);

  selfParams.forEach(({ index, param }) => (paramtypes[index] = param));
  return paramtypes;
}

public reflectOptionalParams<T>(type: Type<T>): any[] {
  return Reflect.getMetadata(OPTIONAL_DEPS_METADATA, type) || [];
}
```

通过 `Reflect.getMetadata` 从内置 key `design:paramtypes` 获取构造函数参数类型作为 token。关于内置 key 之前讲过，可见：[三个内置 metadata key](https://juejin.cn/post/7500981295105458215#heading-28)。

**2. 递归构造每个参数**

调用 `resolveParam` 并发解析当前类的所有依赖，递归调用链如下：

```
resolveConstructorParams()
├─ resolveParam()
│   └─ resolveSingleParam()
│       └─ resolveComponentInstance()
│           ├─ lookupComponent()
│           └─ resolveComponentHost()
│               └─ loadProvider()
│                   └─ loadInstance()
```

`lookupComponent` 负责根据 token 从 scan 流程中构建的依赖图中寻找对应的 `InstanceWrapper`。

**3. 依赖解析完毕后执行 `callback` 进行实例化**

```ts
const callback = async (instances) => {
  // ......
  const instance = await this.instantiateClass(...);
  // ......
  settlementSignal.complete();
};
```

#### 4. instantiateClass

真正执行 `new metatype(...args)` 进行构造，传入的 `instances` 是递归中已构造完成的当前类的依赖实例。构造得到的实例赋值到 `InstanceWrapper` 的 `values` 属性 contextId 对应的实例对象上。

```ts
public async instantiateClass<T = any>(
  instances: any[],
  wrapper: InstanceWrapper,
  targetMetatype: InstanceWrapper,
  contextId = STATIC_CONTEXT,
  inquirer?: InstanceWrapper,
): Promise<T> {
  const { metatype, inject } = wrapper;
  // ......

  instanceHost.instance = wrapper.forwardRef
    ? Object.assign(
      instanceHost.instance,
        new (metatype as Type<any>)(...instances),
      )
    : new (metatype as Type<any>)(...instances);
  instanceHost.isResolved = true;
  return instanceHost.instance;
}
```

#### 5. 标记实例构造完成

最后调用 `settlementSignal.complete()` resolve Promise，标记该实例构造完成。

> 至此，依赖实例化流程结束。

# 总结

本文介绍了 Nest 中核心的依赖注入流程的实现，它分为依赖扫描和依赖实例化两步：

- 扫描阶段：以用户传入的根模块为入口，递归扫描 `imports` 的子模块，注册到 `NestContainer` 实例上的 `modules` 中；同时扫描每个模块依赖的 providers、controllers、injectables，通过 `InstanceWrapper` 包装，记录在 `Module` 结构中；形成依赖图。

- 实例化阶段：通过 reflect-metadata 获取类的构造函数参数类型，作为 token 在依赖图中找到对应的类，递归进行实例化，存储在 `InstanceWrapper` 的 `values` 中。

至此，Nest 的启动流程（`NestFactory.create()`）还剩下最后一步：创建应用实例，将在下一篇文章中继续。
