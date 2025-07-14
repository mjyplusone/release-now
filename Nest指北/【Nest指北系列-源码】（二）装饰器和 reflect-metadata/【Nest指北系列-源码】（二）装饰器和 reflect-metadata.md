装饰器在 Nest 中起着巨大的作用，比如 `@Module` 定义模块，`@Controller`、`@Get`、`@Post` 定义路由 ，`@UseGuards`、`@UseInterceptors` 绑定守卫、拦截器等。让我们从 Typescript 中的装饰器语法开始，逐步了解 Nest 中装饰器的实现原理。

# 装饰器

## 本质和作用

Typescript 中装饰器是一种语法糖，它的本质是一个**函数**。

主要用于修饰类、成员（属性、方法）、参数，针对已存在的类，在不改变原类的情况下，动态的扩展功能。

## 新旧版本装饰器

装饰器的发展历经多年，三个阶段，语法上也产生了较大的变更。总的来说，可以分为**旧版**（**实验性装饰器语法**）和**新版**（**符合 ECMAScript 标准的装饰器语法**）。下面从四个方面对这两个版本进行对比：

### 支持的 Typescript 版本和启用方式

- 旧版装饰器：Typescript 1.5 - 5.0 支持，需要在 tsconfig 中启用 `experimentalDecorators`。

```json
// tsconfig.json
{
  "experimentalDecorators": true
}
```

- 新版装饰器：Typescript 5.0 支持，需要在 tsconfig 中启用 `useDefineForClassFields`，**不需要启用 `experimentalDecorators`**。

```json
// tsconfig.json
{
  "experimentalDecorators": false,
  "target": "ES2022",
  "useDefineForClassFields": true
}
```

### 能装饰什么？

- 旧版装饰器：支持装饰类、方法、属性、参数。
- 新版装饰器：支持装饰类、方法、访问器、属性，**和旧版最大的区别是不再支持参数**。

### 接收参数

- 旧版装饰器：装饰器函数接受 target、key、descriptor 参数。
- 新版装饰器：装饰器函数接受 value 和 context 参数，**参数结构和旧版有很大差异**，更加标准化。

### 返回值

新旧版本装饰器的返回值也存在较大差异，具体见下面的例子。

## 装饰器使用

下面来看一下新旧版本中不同类型装饰器的使用。可以从示例中更详细的了解新旧版本装饰器在参数和返回值上的差异。

### 类装饰器

#### 旧版

```ts
function MyClassDecorator(target: Function) {
  return class extends target {
    extra = "I am new!";
  };
}
```

**参数**：

- `target` 表示被修饰的类。

**返回值**：

- 返回新的构造函数，替换原类。

#### 新版

```ts
function MyClassDecorator(value: Function, context: ClassDecoratorContext) {
  // ❌ 返回值无效！
}
```

**参数**：

- `value` 表示被修饰的类。
- `context` 是含有丰富信息的上下文对象。

**返回值**：

- 返回值**无效**，新版类装饰器主要用于副作用，不能替换类本身。

### 方法装饰器

#### 旧版

```ts
function MyFnDecorator(
  target: any,
  key: string,
  descriptor: PropertyDescriptor
) {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log("args:", args);
    return original.apply(this, args);
  };
  return descriptor; // 可选
}
```

**参数**：

- `target`：如果修饰的是静态方法，则为类本身；如果是实例方法，则为类的原型。
- `key`：方法名。
- `descriptor`：属性描述符对象。

**返回值**：

- 修改 `descriptor.value`，返回修改后的 `descriptor`。

#### 新版

```ts
function MyFnDecorator(value: Function, context: ClassMethodDecoratorContext) {
  return function (...args: any[]) {
    console.log("args:", args);
    return value.apply(this, args);
  };
}
```

**参数**：

- `value` 表示被修饰的方法。
- `context` 是含有丰富信息的上下文对象。

**返回值**：

- 返回一个新函数替换方法本体，不再直接操作 `descriptor`，更加函数式。

### 属性装饰器

#### 旧版

```ts
function FieldDecorator(target: any, key: string) {
  // 不能获取初始值，只知道“谁被装饰了”
  // ❌ 返回值无效！
}
```

**参数**：

- `target`：如果修饰的是静态属性，则为类本身；如果是实例属性，则为类的原型。
- `key`：属性名。
- 无法从参数中获取属性初始值。

**返回值**：

- 返回值**无效**，无法影响字段初始化逻辑。

#### 新版

```ts
function FieldDecorator(
  initialValue: any,
  context: ClassFieldDecoratorContext
) {
  return function () {
    return typeof initialValue === "string"
      ? initialValue.toUpperCase()
      : initialValue;
  };
}
```

**参数**：

- `initialValue` 表示被修饰的属性值，**和旧版的最大区别是可以访问初始化值**。
- `context` 是含有丰富信息的上下文对象。

**返回值**：

- 返回初始化器函数，可以动态更改字段初始值。

### 参数装饰器

仅旧版支持，新版不支持。

#### 旧版

```ts
function LogParam(target: any, key: string, parameterIndex: number) {
  console.log(
    `Parameter decorator called: ${propertyKey} at index ${parameterIndex}`
  );
}

class Example {
  greet(@LogParam name: string) {
    console.log(`Hello, ${name}`);
  }
}
```

**参数**：

- `target`：如果修饰的是静态方法的参数，则为类本身；如果是实例方法的参数，则为类的原型。
- `key`：方法名。
- `parameterIndex`: 修饰的参数在参数列表中的索引。

**返回值**：

- 返回值**无效**。

## Nest 中的装饰器

**Nest 目前（截至 2025 年）仍然使用旧版装饰器规范**。所以使用 Nest 仍需要在 tsconfig.json 中开启：

```json
{
  "experimentalDecorators": true,
  "emitDecoratorMetadata": true
}
```

启用 `experimentalDecorators` 如前面所说用于使用旧版装饰器，而启用 `emitDecoratorMetadata` 则是用于在编译时生成装饰器相关的元数据，这个在 reflect-metadata 一节中具体讲。

**为什么 Nest 没有更新到新版装饰器呢？猜测原因有**：

- 参数装饰器：Nest 的控制器方法中通过 `@Body()`、`@Query()` 等参数装饰器获取路由参数，而新版规范中不支持参数装饰器。
- 元数据收集：旧版装饰器可以配合 reflect-metadata 自动收集类型元数据，而新版装饰器目前做不到。因为 `emitDecoratorMetadata` 是 Typescript 的实验性扩展，特性只有在开启 `experimentalDecorators` 时生效，而新版装饰器不开启 `experimentalDecorators`。

**所以，在阅读 Nest 中装饰器源码时仅关注旧版装饰器语法即可**。

# reflect-metadata

Nest 中装饰器的实现最离不开的就是 **reflect-metadata** 这个库。它**提供了运行时注入和获取元数据的能力**。

## 使用

首先来看下 reflect-metadata 中定义和获取元数据的 API。

### 定义元数据

#### defineMetaData

```ts
function defineMetadata(
  metadataKey: any,
  metadataValue: any,
  target: Object
): void;

function defineMetadata(
  metadataKey: any,
  metadataValue: any,
  target: Object,
  propertyKey: string | symbol
): void;
```

**参数**：

- `metadataKey`: 元信息 key。
- `metadataValue`: 元信息 value。
- `target`: 定义元数据的目标对象。
- `propertyKey`: 可选，定义元数据的目标对象属性。

看个具体的例子：

```ts
import "reflect-metadata";

class Example {
  method() {}
}

// 在 Example 类上定义元数据
Reflect.defineMetadata("customKey", "customValue1", Example.prototype);

// 在 Example 的 method 方法上定义元数据
Reflect.defineMetadata(
  "customKey",
  "customValue2",
  Example.prototype,
  "method"
);
```

#### metadata 装饰器

也可以通过装饰器的方式定义元数据。

```ts
class Example {
  @Reflect.metadata("customKey_decorator", "customValue3")
  method() {}
}
```

### 获取元数据

#### getMetaData

```ts
function getMetadata(metadataKey: any, target: Object): any;

function getMetadata(
  metadataKey: any,
  target: Object,
  propertyKey: string | symbol
): any;
```

**参数**：

- `metadataKey`: 元信息 key。
- `target`: 定义元数据的目标对象。
- `propertyKey`: 可选，定义元数据的目标对象属性。

**返回**：

- 元信息 value。

看个具体的例子：

```ts
// 读取元数据
const value1 = Reflect.getMetadata("customKey", Example.prototype);
const value2 = Reflect.getMetadata("customKey", Example.prototype, "method");
const value3 = Reflect.getMetadata(
  "customKey_decorator",
  Example.prototype,
  "method"
);
console.log(value1); // customValue1
console.log(value2); // customValue2
console.log(value3); // customValue3
```

## 自动收集类型元数据

自动收集类型元数据指的是，Typescript 代码中当满足：

- 开启 `experimentalDecorators`
- 开启 `emitDecoratorMetadata`
- 安装并导入 reflect-metadata 库
- **属性或方法必须实际使用了装饰器**（这一点很重要，只有使用了装饰器的地方会自动收集类型元数据，哪怕是一个空的装饰器）

TypeScript 会自动在编译后的 JavaScript 中插入调用 `Reflect.defineMetadata()` 的代码，用来记录类型信息。

```ts
import "reflect-metadata";

function Empty(target, key) {}

class Example {
  @Empty
  name: string;

  @Empty
  age: number;
}

// 获取属性类型元数据
const type1 = Reflect.getMetadata("design:type", Example.prototype, "name");
const type2 = Reflect.getMetadata("design:type", Example.prototype, "age");
console.log(type1); // [Function: String]
console.log(type2); // [Function: Number]
```

比如这里，并没有显式定义 `design:type`，**`design:type` 是 TypeScript 编译器自动加上的元数据键**，`reflect-metadata` 提供了访问这些信息的方法。

### 三个内置 metadata key

当满足上述条件时，TypeScript 会为装饰目标自动生成的元数据键有以下三个：

| Metadata Key        | 含义               | 用于哪里           |
| ------------------- | ------------------ | ------------------ |
| `design:type`       | 属性的类型         | 用于**属性装饰器** |
| `design:paramtypes` | 方法参数的类型数组 | 用于**方法装饰器** |
| `design:returntype` | 方法的返回类型     | 用于**方法装饰器** |

### 原理

编译前的 ts 代码：

```ts
import "reflect-metadata";

function Empty(target, key) {}

class Example {
  @Empty
  name: string;

  @Empty
  age: number;
}

const type = Reflect.getMetadata("design:type", Example.prototype, "name");
console.log(type);
```

编译后的 js 代码：

```ts
"use strict";
var __decorate =
  (this && this.__decorate) ||
  function (decorators, target, key, desc) {
    var c = arguments.length,
      r =
        c < 3
          ? target
          : desc === null
          ? (desc = Object.getOwnPropertyDescriptor(target, key))
          : desc,
      d;
    if (typeof Reflect === "object" && typeof Reflect.decorate === "function")
      r = Reflect.decorate(decorators, target, key, desc);
    else
      for (var i = decorators.length - 1; i >= 0; i--)
        if ((d = decorators[i]))
          r = (c < 3 ? d(r) : c > 3 ? d(target, key, r) : d(target, key)) || r;
    return c > 3 && r && Object.defineProperty(target, key, r), r;
  };
var __metadata =
  (this && this.__metadata) ||
  function (k, v) {
    if (typeof Reflect === "object" && typeof Reflect.metadata === "function")
      return Reflect.metadata(k, v);
  };
Object.defineProperty(exports, "__esModule", { value: true });
require("reflect-metadata");
function Empty(target, key) {}
class Example {}
__decorate(
  [Empty, __metadata("design:type", String)],
  Example.prototype,
  "name",
  void 0
);
__decorate(
  [Empty, __metadata("design:type", Number)],
  Example.prototype,
  "age",
  void 0
);
const type = Reflect.getMetadata("design:type", Example.prototype, "name");
console.log(type);
//# sourceMappingURL=test.js.map
```

ts 编译后给每个使用了装饰器的属性自动加上 `__metadata("design:type", String)` 装饰器工厂，传入 `design:type` 内置 key 和属性的类型作为参数，`__metadata` 工厂函数中调用 `Reflect.meatadata` 定义元数据。

## Nest 中的封装

Nest 中封装了两个和 reflect-metadata 相关的 API。

### SetMetaData

可以从 `@nestjs/common` 包导入，通常和装饰器配合，用于定义元信息。

```ts
// roles.decorator.ts
import { SetMetadata } from "@nestjs/common";

export const Roles = (...roles: string[]) => SetMetadata("roles", roles);
```

`SetMetadata` 实现如下，本质上是调用了 `Reflect.defineMetadata()`。

```ts
export function SetMetadata<T = any>(metadataKey: string, metadataValue: T) {
  return (target: object, key?: any, descriptor?: any) => {
    Reflect.defineMetadata(
      metadataKey,
      metadataValue,
      descriptor ? descriptor.value : target
    );
  };
}
```

### Reflector 类

可以从 `@nestjs/core` 包导入，通常在守卫、拦截器、管道等通过 `constructor` 注入使用，用于获取元信息。

```ts
import { Reflector } from "@nestjs/core";

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}
  canActivate(
    context: ExecutionContext
  ): boolean | Promise<boolean> | Observable<boolean> {
    const roles = this.reflector.get<string[]>("roles", context.getHandler());
    // ......
  }
}
```

`this.reflector.get` 实现如下，本质上是调用了 `Reflect.getMetadata`。

```ts
get<T = any>(metadataKey: any, target: Function | Object): T {
  return Reflect.getMetadata(metadataKey, target);
}
```

# Nest 中装饰器源码解析

- 不同装饰器由不同工厂函数处理，传入工厂函数的参数为要记录的元数据。

- 通过 `Reflect.defineMetadata` 将元数据注入到装饰器修饰的类或类的成员上，需要的时候再通过 `Reflect.getMetadata` 获取使用。

- Nest 中有使用 Typescript 为装饰目标自动生成的三种 metadata key，也有使用自定义的 metadata key。

下面看一些常用装饰器的源码：

## @Controller

源码位置：`packages/common/decorators/core/controller.decorator.ts`

```ts
export function Controller(
  prefixOrOptions?: string | string[] | ControllerOptions
): ClassDecorator {
  const defaultPath = "/";

  const [path, host, scopeOptions, versionOptions] = isUndefined(
    prefixOrOptions
  )
    ? [defaultPath, undefined, undefined, undefined]
    : isString(prefixOrOptions) || Array.isArray(prefixOrOptions)
    ? [prefixOrOptions, undefined, undefined, undefined]
    : [
        prefixOrOptions.path || defaultPath,
        prefixOrOptions.host,
        { scope: prefixOrOptions.scope, durable: prefixOrOptions.durable },
        Array.isArray(prefixOrOptions.version)
          ? Array.from(new Set(prefixOrOptions.version))
          : prefixOrOptions.version,
      ];

  return (target: object) => {
    Reflect.defineMetadata(CONTROLLER_WATERMARK, true, target);
    Reflect.defineMetadata(PATH_METADATA, path, target);
    Reflect.defineMetadata(HOST_METADATA, host, target);
    Reflect.defineMetadata(SCOPE_OPTIONS_METADATA, scopeOptions, target);
    Reflect.defineMetadata(VERSION_METADATA, versionOptions, target);
  };
}
```

1. `Controller` 工厂函数接收 `prefixOrOptions` 作为参数。
2. 如果没有传入参数，则将 `defaultPath = '/'` 作为 `path`，`host`、`scopeOptions` 和 `versionOptions` 为 `undefined`。
3. 如果传入了 `prefixOrOptions` 且其类型为 `string` 或 `array`，则将其作为 `path`，`host`、`scopeOptions` 和 `versionOptions` 为 `undefined`。
4. 否则，将传入的 `prefixOrOptions` 按照 `ControllerOptions` 解析得到 `host`、`scopeOptions` 和 `versionOptions`。
5. 工厂函数返回装饰器函数，接收 `target` 目标类作为参数，将 `path`、`host`、`scopeOptions`、 `versionOptions` 作为元信息，通过 `Reflect.defineMetadata` 注入到 `@Controller` 装饰器修饰的目标类上。
6. 同时将在 `@Controller` 装饰器修饰的目标类上注入 `CONTROLLER_WATERMARK` 为 `true`，在依赖扫描阶段，用于判断此类为 `Controller` 类。

## @Get

源码位置：`packages/common/decorators/http/request-mapping.decorator.ts`

```ts
export const Get = createMappingDecorator(RequestMethod.GET);

const createMappingDecorator =
  (method: RequestMethod) =>
  (path?: string | string[]): MethodDecorator => {
    return RequestMapping({
      [PATH_METADATA]: path,
      [METHOD_METADATA]: method,
    });
  };

export const RequestMapping = (
  metadata: RequestMappingMetadata = defaultMetadata
): MethodDecorator => {
  const pathMetadata = metadata[PATH_METADATA];
  const path = pathMetadata && pathMetadata.length ? pathMetadata : "/";
  const requestMethod = metadata[METHOD_METADATA] || RequestMethod.GET;

  return (
    target: object,
    key: string | symbol,
    descriptor: TypedPropertyDescriptor<any>
  ) => {
    Reflect.defineMetadata(PATH_METADATA, path, descriptor.value);
    Reflect.defineMetadata(METHOD_METADATA, requestMethod, descriptor.value);
    return descriptor;
  };
};
```

1. `Get` 工厂函数接收 `path` 作为参数。
2. 工厂函数返回装饰器函数，接收参数： `target` 装饰的方法所在的类，`key` 装饰的方法，`descriptor` 方法描述器。
3. 调用 `Reflect.defineMetadata`，将 `path` 和 `method` 作为元数据注入到 `descriptor.value` 上。
4. 在框架启动时，通过 `RouterExplorer` 去读取这些元数据并注册路由。

其他 http 请求参数装饰器，比如 `@Post`、`@Put` 等原理类似。

## @UseGuards

源码位置：`packages/common/decorators/core/use-guards.decorator.ts`

```ts
export function UseGuards(
  ...guards: (CanActivate | Function)[]
): MethodDecorator & ClassDecorator {
  return (
    target: any,
    key?: string | symbol,
    descriptor?: TypedPropertyDescriptor<any>
  ) => {
    const isGuardValid = <T extends Function | Record<string, any>>(guard: T) =>
      guard && (isFunction(guard) || isFunction(guard.canActivate));

    if (descriptor) {
      validateEach(
        target.constructor,
        guards,
        isGuardValid,
        "@UseGuards",
        "guard"
      );
      extendArrayMetadata(GUARDS_METADATA, guards, descriptor.value);
      return descriptor;
    }
    validateEach(target, guards, isGuardValid, "@UseGuards", "guard");
    extendArrayMetadata(GUARDS_METADATA, guards, target);
    return target;
  };
}
```

1. `UseGuards` 工厂函数接收单个守卫或守卫列表作为参数。
2. 工厂函数返回装饰器函数，接收参数： `target` 装饰的方法所在的类，`key` 装饰的方法，`descriptor` 方法描述器。
3. 根据 `descriptor` 是否存在分为控制器守卫和路由方法守卫分别处理。
4. 校验传入的守卫的有效性，即是否为函数或带有 `canActive` 方法的类。
5. 调用 `Reflect.defineMetadata`，将传入的 `guards` 作为元数据注入到 `@Useguards` 修饰的类或方法上。
6. 在依赖扫描阶段读取并使用。

## @Module

源码位置：`packages/common/decorators/modules/module.decorator.ts`

```ts
export function Module(metadata: ModuleMetadata): ClassDecorator {
  const propsKeys = Object.keys(metadata);
  validateModuleKeys(propsKeys);

  return (target: Function) => {
    for (const property in metadata) {
      if (Object.hasOwnProperty.call(metadata, property)) {
        Reflect.defineMetadata(property, (metadata as any)[property], target);
      }
    }
  };
}
```

1. `Module` 工厂函数接收模块描述对象作为参数。
2. 校验传入的模块描述对象中 key 的有效性。
3. 工厂函数返回装饰器函数，接收 `target` 目标类为参数，即 `@Module` 修饰的模块类。
4. 调用  `Reflect.defineMetadata`，将传入的模块描述对象中定义的 `imports`、`providers`、`controllers` 等作为元数据注入到目标类上。
5. 在依赖扫描阶段读取并使用。

## 使用定义的元数据

当 Nest 启动时，会通过内部工具类（如 `MetadataScanner`、 `RouterExplorer`）去扫描在类、方法、参数上定义的元数据，用于生成依赖树、依赖实例化、注册路由等。这些内部工具类本质上也是调用了 `Reflect.getMetadata`。

关于这部分我们会在后面的章节中细说。

# 总结

装饰器本质上是一个函数，其参数中可以获取修饰的类或类的成员。Nest 中的装饰器配合 reflect-metadata 库，在类或类的成员上定义元数据，然后在启动阶段获取这些元数据并使用。
