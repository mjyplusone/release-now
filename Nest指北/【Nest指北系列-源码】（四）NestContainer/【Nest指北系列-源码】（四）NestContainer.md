上一篇文章中我们从整体视角讲述了 Nest 的启动流程，其中提到的第一个步骤是创建 `NestContainer` 容器。本章我们就来详细了解下 `NestContainer`，为后续学习 Nest 依赖注入系统打下基础。

> 注意～注意～长文预警[手动狗头]

# NestContainer 类

`NestContainer` 是一个容器类，我们来看下这个类中维护的属性与方法。

源码位置：`packages/core/injector/container.ts`。

## 属性

`NestContainer` 类中通过私有成员的方式维护了以下属性：

```ts
export class NestContainer {
  private readonly globalModules = new Set<Module>();
  private readonly modules = new ModulesContainer();
  private readonly dynamicModulesMetadata = new Map<
    string,
    Partial<DynamicModule>
  >();
  private readonly internalProvidersStorage = new InternalProvidersStorage();
  private readonly _serializedGraph = new SerializedGraph();
  private moduleCompiler: ModuleCompiler;
  private internalCoreModule: Module;
}
```

### globalModules

**作用**：存储通过 `@Global()` 装饰器声明的全局模块。

**数据结构**：`Set<Module>`，全局模块的集合。

### modules

**作用**：存储所有被注册的模块。

**数据结构**：`ModulesContainer`，实际上就是 `Map<string, Module>`，模块 token 到模块实例的映射。

### dynamicModulesMetadata

**作用**：存储动态模块对象。

**数据结构**：`Map<string, Partial<DynamicModule>>`，模块 token 到动态模块对象的映射。

```ts
export interface DynamicModule extends ModuleMetadata {
  module: Type<any>;
  global?: boolean;
}

export interface ModuleMetadata {
  imports?: Array<
    Type<any> | DynamicModule | Promise<DynamicModule> | ForwardReference
  >;
  controllers?: Type<any>[];
  providers?: Provider[];
  exports?: Array<
    | DynamicModule
    | string
    | symbol
    | Provider
    | ForwardReference
    | Abstract<any>
    | Function
  >;
}
```

### internalProvidersStorage

**作用**：存储内部使用的框架级别 Provider，主要与 http 适配器相关。

**数据结构**：`InternalProvidersStorage`，一个封装类。

```ts
export class InternalProvidersStorage {
  private readonly _httpAdapterHost = new HttpAdapterHost();
  private _httpAdapter: AbstractHttpAdapter;

  get httpAdapterHost(): HttpAdapterHost {
    return this._httpAdapterHost;
  }

  get httpAdapter(): AbstractHttpAdapter {
    return this._httpAdapter;
  }

  set httpAdapter(httpAdapter: AbstractHttpAdapter) {
    this._httpAdapter = httpAdapter;
  }
}
```

### \_serializedGraph

**作用**：存储序列化后的模块依赖图。

**数据结构**：`SerializedGraph`，一个图结构，node 表示模块，edge 表示模块之间的依赖。

```ts
export class SerializedGraph {
  private readonly nodes = new Map<string, Node>();
  private readonly edges = new Map<string, Edge>();

  public insertNode(nodeDefinition: Node) {}

  public insertEdge(edgeDefinition: WithOptionalId<Edge>) {}

  // ......
}
```

### moduleCompiler

**作用**：解析模块的元数据，生成唯一的模块 token。

**数据结构**：`ModuleCompiler`，一个封装类，提供解析模块的方法。

### internalCoreModule

**作用**：存储 Nest 框架内部的核心模块，包含 Nest 本身注入的一些 Provider，比如 `Reflector` 等。

**数据结构**：`Module`，一个模块实例，和普通用户模块一致。

> `NestContainer` 维护的属性的数据结构中涉及了两个比较重要的类：**`Module`** 和 **`ModuleCompiler`**，后续会展开分析。

## 方法

下面是 `NestContainer` 类中维护的部分常用方法。

```ts
export class NestContainer {
  public async addModule() {}

  public async addDynamicMetadata() {}

  public getModules() {}

  public getModuleByKey() {}

  public addProvider() {}

  public addController() {}

  public addImport() {}

  public bindGlobalScope() {}
}
```

方法可以分为以下几类：

### 模块注册

#### addModule

```ts
public async addModule(
  metatype: ModuleMetatype,
  scope: ModuleScope,
) {
  if (!metatype) {
    throw new UndefinedForwardRefException(scope);
  }
  const { type, dynamicMetadata, token } =
    await this.moduleCompiler.compile(metatype);
  if (this.modules.has(token)) {
    return {
      moduleRef: this.modules.get(token)!,
      inserted: true,
    };
  }

  return {
    moduleRef: await this.setModule(
      {
        token,
        type,
        dynamicMetadata,
      },
      scope,
    ),
    inserted: true,
  };
}
```

**参数**：

- `metatype`: 模块类。

- `scope`: 当前模块的作用域链，用于处理依赖注入的作用域。

**核心逻辑**：

1. 调用 `moduleCompiler.compile` 解析模块元数据，得到模块唯一 token。具体如何解析会在后续对 `ModuleCompiler` 类的介绍中提到。

2. 使用解析得到的模块 token 判断模块是否已经注册。

3. 如果已注册，则直接返回已有模块实例。

4. 如果未注册，调用 `setModule` 方法创建模块实例。

```ts
private async setModule(
  { token, dynamicMetadata, type }: ModuleFactory,
  scope: ModuleScope,
): Promise<Module> {
  const moduleRef = new Module(type, this);
  moduleRef.token = token;
  moduleRef.initOnPreview = this.shouldInitOnPreview(type);
  this.modules.set(token, moduleRef);

  const updatedScope = ([] as ModuleScope).concat(scope, type);
  await this.addDynamicMetadata(token, dynamicMetadata!, updatedScope);

  if (this.isGlobalModule(type, dynamicMetadata)) {
    moduleRef.isGlobal = true;

    moduleRef.distance = Number.MAX_VALUE;
    this.addGlobalModule(moduleRef);
  }

  return moduleRef;
}
```

5. `setModule` 方法中通过 `new Module()` 创建模块实例（具体结构后续介绍），然后将模块实例存储到 `NestContainer` 实例上的 `modules` Map 结构中，key 为模块 token。

6. 判断如果当前注册的模块是动态模块，将动态模块对象存储到 `NestContainer` 实例上的 `dynamicModulesMetadata` 结构中，再调用 `addDynamicMetadata` 方法提取动态模块 `imports` 中的模块并递归添加。

```ts
public async addDynamicMetadata(
  token: string,
  dynamicModuleMetadata: Partial<DynamicModule>,
  scope: Type<any>[],
) {
  if (!dynamicModuleMetadata) {
    return;
  }
  this.dynamicModulesMetadata.set(token, dynamicModuleMetadata);

  const { imports } = dynamicModuleMetadata;
  await this.addDynamicModules(imports!, scope);
}

public async addDynamicModules(modules: any[], scope: Type<any>[]) {
  if (!modules) {
    return;
  }
  await Promise.all(modules.map(module => this.addModule(module, scope)));
}
```

7. 判断如果当前注册的模块是全局模块，调用 `addGlobalModule` 方法，额外将模块实例存储到 `NestContainer` 实例上的 `globalModules` Set 结构中。

```ts
public addGlobalModule(module: Module) {
  this.globalModules.add(module);
}
```

### 模块查找

#### getModules

获取所有已注册的模块。

```ts
public getModules(): ModulesContainer {
  return this.modules;
}
```

#### getModuleByKey

根据模块 token 获取已注册的模块实例。

```ts
public getModuleByKey(moduleKey: string): Module | undefined {
  return this.modules.get(moduleKey);
}
```

### Provider 注册

#### addProvider

将 Provider 添加到模块中。

```ts
public addProvider(
  provider: Provider,
  token: string, // 模块 token
  enhancerSubtype?: EnhancerSubtype,
): string | symbol | Function {
  const moduleRef = this.modules.get(token);
  if (!provider) {
    throw new CircularDependencyException(moduleRef?.metatype.name);
  }
  if (!moduleRef) {
    throw new UnknownModuleException();
  }
  const providerKey = moduleRef.addProvider(provider, enhancerSubtype!);
  const providerRef = moduleRef.getProviderByKey(providerKey);

  DiscoverableMetaHostCollection.inspectProvider(this.modules, providerRef);

  return providerKey as Function;
}
```

**参数**：

- `provider`：支持传入各种类型的 provider，比如直接传类名、`useClass`、`useValue`、`useFactory`、`useExisting`。

- `token`：目标模块 token。

**核心逻辑**：

1. 通过 token 拿到目标模块。

2. 调用目标模块实例上的 `moduleRef.addProvider` 方法将 provider 添加到模块中。即，**真正的 provider 注册是委托给 `Module` 类完成的**。`Module` 类的实现我们后续会讲到。

### Controller 注册

#### addController

将 Controller 添加到模块中。

```ts
public addController(controller: Type<any>, token: string) {
  if (!this.modules.has(token)) {
    throw new UnknownModuleException();
  }
  const moduleRef = this.modules.get(token)!;
  moduleRef.addController(controller);

  const controllerRef = moduleRef.controllers.get(controller)!;
  DiscoverableMetaHostCollection.inspectController(
    this.modules,
    controllerRef,
  );
}
```

**参数**：

- `controller`：控制器类。

- `token`：目标模块 token。

**核心逻辑**：

1. 通过 token 拿到目标模块。

2. 调用目标模块实例上的 `moduleRef.addController` 方法将 controller 添加到模块中。即，**真正的 controller 注册也是委托给 `Module` 类完成的**。

### 依赖关系建立

#### addImport

把一个子模块添加为当前模块的依赖模块。即，如果这样一个模块定义：

```ts
@Module({
  imports: [UserModule],
})
export class AppModule {}
```

那么 Nest 会调用 `addImport(UserModule, AppModuleToken)`，将 `UserModule` 添加为 `AppModule` 的依赖。

看下源码：

```ts
public async addImport(
  relatedModule: Type<any> | DynamicModule,
  token: string,
) {
  if (!this.modules.has(token)) {
    return;
  }
  const moduleRef = this.modules.get(token)!;
  const { token: relatedModuleToken } =
    await this.moduleCompiler.compile(relatedModule);
  const related = this.modules.get(relatedModuleToken)!;
  moduleRef.addImport(related);
}
```

**参数**：

- `relatedModule`：被 imports 的模块，可以是静态类，也可以是动态模块对象。

- `token`：目标模块 token。

**核心逻辑**：

1. 通过 token 拿到目标模块。

2. 调用 `moduleCompiler.compile` 解析模块，得到模块唯一 token。

3. 通过 token 拿到 imports 模块的引用。

4. 调用目标模块实例上的 `moduleRef.addImport` 方法设置目标模块的依赖关系。即，**真正的依赖关系建立是委托给 `Module` 类完成的**。

#### bindGlobalScope

将标记了 `@Global` 的全局模块标记为其他所有模块的依赖。

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

1. 遍历所有注册的模块，对每一个模块，遍历全局模块，调用 `moduleRef.addImport` 方法建立依赖关系。

2. 遍历设置依赖关系时，排除全局模块自身和内部核心模块。

## 职责

通过对 `NestContainer` 类维护的属性与方法的分析，不难看出 `NestContainer` 类的主要职责如下：

1. 模块注册与查找

2. 管理 Provider 和 Controller

3. 建立模块之间的依赖关系

# Module 类

接下来，看一下 `NestContainer` 类中多次使用的 `Module` 类。`Module` 类是用于描述和封装一个模块元信息和运行时状态的内部结构。Nest 应用的每个模块在运行时都会被实例化成一个 `Module` 实例。

## 属性

`Module` 类中维护的属性如下：

```ts
export class Module {
  private readonly _id: string;
  private readonly _imports = new Set<Module>();
  private readonly _providers = new Map<
    InjectionToken,
    InstanceWrapper<Injectable>
  >();
  private readonly _injectables = new Map<
    InjectionToken,
    InstanceWrapper<Injectable>
  >();
  private readonly _middlewares = new Map<
    InjectionToken,
    InstanceWrapper<Injectable>
  >();
  private readonly _controllers = new Map<
    InjectionToken,
    InstanceWrapper<Controller>
  >();
  private readonly _entryProviderKeys = new Set<InjectionToken>();
  private readonly _exports = new Set<InjectionToken>();

  private _distance = 0;
  private _initOnPreview = false;
  private _isGlobal = false;
  private _token: string;
}
```

### \_providers

**作用**：存储该模块中注册的所有 provider。

**数据结构**：`Map<InjectionToken, InstanceWrapper<Injectable>>`，一个 Map 结构，key 是 provider 的 token，value 是 provider 封装后的`InstanceWrapper` 实例。

> `InstanceWrapper` 实例中包含 token、原始类、类实例、生命周期范围等信息，会在后续详细介绍。

### \_injectables

**作用**：存储所有被 `@Injectable()` 装饰，参与依赖注入系统的类。即除了普通的 provider，还包括：Guard、Pipe、Interceptor、Middleware 等。

**数据结构**：`Map<InjectionToken, InstanceWrapper<Injectable>>`。

### \_controllers

**作用**：存储该模块中声明的控制器。

**数据结构**：`Map<InjectionToken, InstanceWrapper<Controller>>`。

### \_imports

**作用**：存储该模块导入的其他模块。**Nest 通过它来构建模块之间的依赖图，进行依赖注入查找**。

**数据结构**：`Set<Module>`，模块实例的集合。

### \_exports

**作用**：存储该模块导出的 provider。

**数据结构**：`Set<InjectionToken>`，provider token 的集合。

### \_distance

**作用**：记录模块之间的依赖深度。

**数据结构**：`number`。

### \_token

**作用**：`ModuleCompiler` 编译模块时生成的唯一 id，用于内部模块映射。

**数据结构**：`string`。

### \_metatype

**作用**：当前模块的类。

**数据结构**：：`Type<any>`。

## 方法

```ts
export class Module {
  public addProvider() {}

  public addInjectable() {}

  public addController() {}

  public addImport() {}
}
```

### addProvider

把 provider 封装成 `InstanceWrapper`，注册到模块的 `_providers` 中。

```ts
public addProvider(provider: Provider, enhancerSubtype?: EnhancerSubtype) {
  if (this.isCustomProvider(provider)) {
    if (this.isEntryProvider(provider.provide)) {
      this._entryProviderKeys.add(provider.provide);
    }
    return this.addCustomProvider(provider, this._providers, enhancerSubtype);
  }

  const isAlreadyDeclared = this._providers.has(provider);
  if (this.isTransientProvider(provider) && isAlreadyDeclared) {
    return provider;
  }

  this._providers.set(
    provider,
    new InstanceWrapper({
      token: provider,
      name: (provider as Type<Injectable>).name,
      metatype: provider as Type<Injectable>,
      instance: null,
      isResolved: false,
      scope: getClassScope(provider),
      durable: isDurable(provider),
      host: this,
    }),
  );

  if (this.isEntryProvider(provider)) {
    this._entryProviderKeys.add(provider);
  }

  return provider as Type<Injectable>;
}
```

**参数**：

- `provider`：支持传入各种类型的 provider，比如直接传类名、`useClass`、`useValue`、`useFactory`、`useExisting`。

**核心逻辑**：

1. 检查是否为自定义 provider（即 `useClass`、`useValue`、`useFactory`、`useExisting` 形式注册的 provider）。

```ts
public isCustomProvider(
  provider: Provider,
): provider is
  | ClassProvider
  | FactoryProvider
  | ValueProvider
  | ExistingProvider {
  return !isNil(
    (
      provider as
        | ClassProvider
        | FactoryProvider
        | ValueProvider
        | ExistingProvider
    ).provide,
  );
}
```

2. 如果是自定义 provider，调用 `addCustomProvider` 方法分类解析和处理。

```ts
public addCustomProvider(
  provider:
    | ClassProvider
    | FactoryProvider
    | ValueProvider
    | ExistingProvider,
  collection: Map<Function | string | symbol, any>,
  enhancerSubtype?: EnhancerSubtype,
) {
  if (this.isCustomClass(provider)) {
    this.addCustomClass(provider, collection, enhancerSubtype);
  } else if (this.isCustomValue(provider)) {
    this.addCustomValue(provider, collection, enhancerSubtype);
  } else if (this.isCustomFactory(provider)) {
    this.addCustomFactory(provider, collection, enhancerSubtype);
  } else if (this.isCustomUseExisting(provider)) {
    this.addCustomUseExisting(provider, collection, enhancerSubtype);
  }
  return provider.provide;
}
```

比如，`{ provide: "xxx", useClass: MyService }` 定义 provider 的场景：

- 取 `provide` 字段的值为 `token`。
- 取 `useClass` 字段传入的类的 name 值为 `name`。
- 取 `useClass` 字段的值为 `metatype`。
- `instance` 值为 null。

传入 `InstanceWrapper` 类得到实例。

```ts
public addCustomClass(
  provider: ClassProvider,
  collection: Map<InjectionToken, InstanceWrapper>,
  enhancerSubtype?: EnhancerSubtype,
) {
  let { scope, durable } = provider;

  const { useClass } = provider;
  if (isUndefined(scope)) {
    scope = getClassScope(useClass);
  }
  if (isUndefined(durable)) {
    durable = isDurable(useClass);
  }

  const token = provider.provide;
  collection.set(
    token,
    new InstanceWrapper({
      token,
      name: useClass?.name || useClass,
      metatype: useClass,
      instance: null,
      isResolved: false,
      scope,
      durable,
      host: this,
      subtype: enhancerSubtype,
    }),
  );
}
```

再比如，`{ provide: "xxx", useValue: "xxx" }` 定义 provider 的场景：

```ts
public addCustomValue(
  provider: ValueProvider,
  collection: Map<Function | string | symbol, InstanceWrapper>,
  enhancerSubtype?: EnhancerSubtype,
) {
  const { useValue: value, provide: providerToken } = provider;
  collection.set(
    providerToken,
    new InstanceWrapper({
      token: providerToken,
      name: (providerToken as Function)?.name || providerToken,
      metatype: null!,
      instance: value,
      isResolved: true,
      async: value instanceof Promise,
      host: this,
      subtype: enhancerSubtype,
    }),
  );
}
```

- 取 `provide` 字段的值为 `token`。
- `provide` 字段值如果是函数，取其 name 值为 `name`，否则取 `provide` 字段的值为 `name`。
- `metatype` 值为 null。
- `instance` 值为 null。

传入 `InstanceWrapper` 类得到实例。

3. 如果是普通 provider：

- 取 `provider` 类名为 `token`。
- 取 `provider` 类名的 name 值为 `name`。
- 取 `provider` 类名为 `metatype`。
- `instance` 值为 null。

传入 `InstanceWrapper` 类得到实例。

> **总结一下**：不论是什么类型的 provider，都是从 provider 的定义解析出 `token`、`name`、`metatype`、`instance` 等属性，传给 `InstanceWrapper` 得到实例，再存储到模块的 `_providers` 中**生成 token -> InstanceWrapper 的映射**。

### addInjectable

**参数**：

- `injectable`：被 `@Injectable()` 装饰，参与依赖注入系统的类。

**核心逻辑**：

和 `addProvider` 基本一致，只是最终将封装后的 `InstanceWrapper` 实例存储到模块的 `_injectables`。

### addController

```ts
public addController(controller: Type<Controller>) {
  this._controllers.set(
    controller,
    new InstanceWrapper({
      token: controller,
      name: controller.name,
      metatype: controller,
      instance: null!,
      isResolved: false,
      scope: getClassScope(controller),
      durable: isDurable(controller),
      host: this,
    }),
  );

  this.assignControllerUniqueId(controller);
}
```

**参数**：

- `controller`：控制器类。

**核心逻辑**：

和 `addProvider` 中对普通 provider 的处理基本一致。

- 取 `controller` 类名为 `token`。
- 取 `controller` 类名的 name 值为 `name`。
- 取 `controller` 类名为 `metatype`。
- `instance` 值为 null。

传入 `InstanceWrapper` 类得到实例，将实例存储到模块的 `_controllers`。

### addImport

```ts
public addImport(moduleRef: Module) {
  this._imports.add(moduleRef);
}
```

**参数**：

- `moduleRef`：`Module` 实例。

**核心逻辑**：

将传入的模块实例添加到当前模块的 `_imports` 集合中，形成模块之间的依赖图。

## InstanceWrapper 类

在对 `Module` 类属性和方法的介绍中，多次提到了 `InstanceWrapper`，模块中的 provider、controller 等都是封装成 `InstanceWrapper` 实例存储在 `Module` 类中的。那么 `InstanceWrapper` 是什么呢？

`InstanceWrapper` 是 Nest 中一个用于管理依赖的类，简化后的核心结构长这样：

```ts
export class InstanceWrapper<T = any> {
  public readonly token: InjectionToken;
  public metatype: Type<T> | Function | null;
  public readonly name: any;
  private readonly values = new WeakMap<ContextId, InstancePerContext<T>>();
  public scope?: Scope = Scope.DEFAULT;
  public readonly host?: Module;
}
```

### token

当前 provider 的唯一标识，可以是类、字符串、Symbol。

### metatype

原始类或函数，用于实例化。

### name

provider 的名字，可能是类名、函数名、`useValue` 场景下 `provide` 的 token 等。

### host

记录 provider 属于哪个模块。

### scope

生命周期范围，有 `DEFAULT`（Singleton）、`REQUEST`、`TRANSIENT`。

### values 实例相关

一个 WeakMap 结构，存储 `contextId -> InstancePerContext` 的映射。

```ts
export interface InstancePerContext<T> {
  instance: T;
  isResolved?: boolean;
  isPending?: boolean;
  donePromise?: Promise<unknown>;
}
```

#### instance

实际生成的实例。大部分场景下，一开始为 null，运行时反射构造函数参数得到 wrapper 并实例化后赋值，这个实例化的过程会在下一章讲解。

#### isResolved

标记是否已经实例化，可以用于避免重复实例化。

#### isPending

标记实例是否正在构造中（异步构造或者异步生命周期钩子还在执行），用于防止多个依赖在同一时间构造导致冲突。

#### donePromise

一个 Promise，表示实例构造状态。

> 下一篇关于依赖实例化的文章中还会提及到 `isPending` 和 `donePromise`。

#### 为什么需要用 Map 结构存储实例？

在默认（Singleton）作用域下，只需要一个实例就够了。但在 `REQUEST` 或 `TRANSIENT` 模式下，一个类的实例不能全局复用，而是需要为每个请求、每次注入生成新的实例。

于是，需要使用 Map 结构存储实例，Map key 是 Nest 用来唯一标识某一个请求上下文或注入上下文的标识符。

比如：

```ts
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {}
```

第一个请求上下文 ID 是 ctx1，第二个请求上下文 ID 是 ctx2。每个上下文都会有自己的实例：

```ts
values = WeakMap {
  ctx1 → { instance: service1, isResolved: true },
  ctx2 → { instance: service2, isResolved: true }
}
```

#### contextId 如何生成？

- 默认作用域，Nest 会统一使用一个固定的 contextId：`STATIC_CONTEXT`。即可做到**全局共享同一个实例**。

```ts
const STATIC_CONTEXT_ID = 1;
export const STATIC_CONTEXT: ContextId = Object.freeze({
  id: STATIC_CONTEXT_ID,
});
```

- `REQUEST` 作用域，每个请求进来时，Nest 会调用 `createContextId` 生成一个唯一的 contextId，它会沿调用链传递下去：

```ts
export function createContextId(): ContextId {
  return { id: Math.random() };
}
```

- `TRANSIENT` 作用域，每次注入时会调用 `createContextId` 动态生成的临时 contextId。

> `InstanceWrapper` 中封装核心结构就是这些，关于不同类型的 provider 转为 `InstanceWrapper` 实例时的具体属性值，在上面 `Module` 类的 `addProvider` 方法中已经有讲到。

# ModuleCompiler 类

`ModuleCompiler` 类主要负责**模块编译**，即解析 `@Module()` 装饰器中的配置，得到内部需要使用的结构。比如，调用 `NestContainer` 类中 `addModule` 方法进行模块注册时，会先用 `moduleCompiler.compile()` 解析得到模块 token。

`ModuleCompiler` 的源码并不长，主要是实现了 `extractMetadata` 和 `compile` 两个方法，完整内容如下：

```ts
export class ModuleCompiler {
  constructor(
    private readonly _moduleOpaqueKeyFactory: ModuleOpaqueKeyFactory
  ) {}

  get moduleOpaqueKeyFactory(): ModuleOpaqueKeyFactory {
    return this._moduleOpaqueKeyFactory;
  }

  public async compile(
    moduleClsOrDynamic:
      | Type
      | DynamicModule
      | ForwardReference
      | Promise<DynamicModule>
  ): Promise<ModuleFactory> {
    moduleClsOrDynamic = await moduleClsOrDynamic;

    const { type, dynamicMetadata } = this.extractMetadata(moduleClsOrDynamic);
    const token = dynamicMetadata
      ? this._moduleOpaqueKeyFactory.createForDynamic(
          type,
          dynamicMetadata,
          moduleClsOrDynamic as DynamicModule | ForwardReference
        )
      : this._moduleOpaqueKeyFactory.createForStatic(
          type,
          moduleClsOrDynamic as Type
        );

    return { type, dynamicMetadata, token };
  }

  public extractMetadata(
    moduleClsOrDynamic: Type | ForwardReference | DynamicModule
  ): {
    type: Type;
    dynamicMetadata: Omit<DynamicModule, "module"> | undefined;
  } {
    if (!this.isDynamicModule(moduleClsOrDynamic)) {
      return {
        type: (moduleClsOrDynamic as ForwardReference)?.forwardRef
          ? (moduleClsOrDynamic as ForwardReference).forwardRef()
          : moduleClsOrDynamic,
        dynamicMetadata: undefined,
      };
    }
    const { module: type, ...dynamicMetadata } = moduleClsOrDynamic;
    return { type, dynamicMetadata };
  }

  public isDynamicModule(
    moduleClsOrDynamic: Type | DynamicModule | ForwardReference
  ): moduleClsOrDynamic is DynamicModule {
    return !!(moduleClsOrDynamic as DynamicModule).module;
  }
}
```

## extractMetadata

提取模块类和动态模块元数据。

**参数**：

- `moduleClsOrDynamic`: 普通模块类或动态模块对象。

**核心逻辑**：

1. 根据传入的参数上是否有 `module` 属性，判断传入的是普通模块类还是调用 `.forRoot()` 等返回的动态模块对象。

2. 如果传入的是普通模块：返回 `type` 为模块类，`dynamicMetadata` 为 undefined。

3. 如果传入的是动态模块对象：返回 `type` 为动态模块对象上的 `module` 属性，即动态模块类；`dynamicMetadata` 为动态模块元数据。

## compile

`ModuleCompiler` 类提供给外部调用的方法，解析模块。

**参数**：

- `moduleClsOrDynamic`: 普通模块类或动态模块对象。

**核心逻辑**：

1. 调用 `extractMetadata` 得到模块类和动态模块元数据。

2. 根据是否有动态模块元数据判断是否为动态模块，分情况生成模块 token。

- 普通模块：随机字符串 + hash(模块类名)

```ts
`${this.generateRandomString()}:${this.hashString(moduleCls.toString())}`;
```

- 动态模块：随机字符串 + hash(模块类名 + 序列化后的动态模块元数据)

```ts
`${this.generateRandomString()}:${this.hashString(
  moduleCls.name + JSON.stringify(dynamicMetadata)
)}`;
```

最终结果类似这样：

```ts
{
  type: MyModule,
  token: 'some-unique-token-based-on-config',
  dynamicMetadata: {
    providers: [...],
    imports: [...],
    ...
  }
}
```

这个结构会交给 `NestContainer` 来注册成一个 `Module` 实例。

# 总结

根据上面的介绍，我们可以了解到 `NestContainer` 是用于管理整个应用程序中依赖注入的容器，但它其实不直接管理具体的 provider / controller，而是负责维护所有模块（`Module`），再由每个 `Module` 负责维护自己的 `providers`、`controllers` 以及模块内的依赖注入信息。

本文以 `NestContainer` 为起点，展开介绍了 `Module`、`InstanceWrapper`、`ModuleCompiler` 等 Nest 中的核心结构。读到这里，可能还留有一些疑问：

- 这些结构中提供了多个方法用于生成整体的模块依赖图，比如 `addModule`、`addProvider` 等，都是在什么时候被调用的呢？

- 封装为 `InstanceWrapper` 的 provider 类，是在什么时候被实例化的呢？

下一篇关于 Nest 依赖系统的文章将会解答这些问题。
