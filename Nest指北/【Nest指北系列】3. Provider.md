Provider 是 Nest 的核心概念之一，从字面意义可以理解为「服务的提供者」，通常用于提供一些具体的功能实现，比如访问数据库等。Provider 是 Nest 中依赖注入的基本单元，可以被其他 Controller 或 Provider 依赖，通过 Nest IoC 容器自动实例化和注入（[Nest 中的控制反转和依赖注入](https://juejin.cn/post/7471453409704919091#heading-8))。

# 基本用法

首先让我们通过示例快速了解 Provider 最基本的使用方式。

## nest cli 快速创建 Provider 类

    nest g service user

这条命令在 src 下 user 文件夹中创建 user.service 文件，内容如下：

```ts
import { Injectable } from "@nestjs/common";

@Injectable()
export class UserService {}
```

可以看到，Provider 本质上是被 `@Injectable` 装饰的类，`@Injectable` 装饰器声明这个类被 IoC 容器接管。

## 在 module 中注册

接下来在模块的 `@Module` 装饰器的 `providers` 数组中注册定义好的 Provider 类。这里的注册实际上是标记 token 与同名类 UserService 的关联。这个写法是一种语法糖，下文会介绍其完整写法以及更多的 Provider 注册方式。

```ts
import { Module } from "@nestjs/common";
import { UserController } from "./user.controller";
import { UserService } from "./user.service";

@Module({
  controllers: [UserController],
  providers: [UserService],
})
export class UserModule {}
```

## 注入

1.  定义和注册好的 Provider 类可以通过 constructor 构造函数注入 Controller 使用。

```ts
@Controller("/user")
export class UserController {
  constructor(private userService: UserService) {}

  @Post("/")
  createUser(@Body() body: CreateUserDto) {
    return this.userService.createUser(body);
  }
}
```

2.  也可以注入其他 Provider，让 Provider 之间建立依赖关系。

```ts
@Injectable()
export class AppService {
  constructor(private userService: UserService) {}
  getUsers() {
    return this.userService.getUser();
  }
}
```

# 更多注册方式和使用场景

## 类注册的语法糖与完整写法

上面提到的在 `@Module` 中 `providers: [UserService]` **类注册**的用法是一种语法糖，实际上等价于以下写法：

```ts
providers: [
  {
    provide: UserService,
    useClass: UserService,
  },
];
```

- `provide` 用于**自定义 token**。
- `useClass` 用于指定提供的类，Nest IoC 容器将管理它的实例化和注入。

一般来说，实际使用中一般使用类注册的方式，方便简洁。也有一些场景需要使用自定义 token 的注册方式，下面会举例说明它们的使用场景。

## 自定义 token 注册

Provider 注册时，自定义的 token 可以是**类名**、**字符串**、**Symbol**。有多种注册方式，比如 `useClass` 提供类、`useValue` 提供值、`useFactory` 提供工厂函数、`useExisting` 提供别名。

```ts
// token 为类名
providers: [
  {
    provide: UserService,
    useClass: UserService,
  },
];
// token 为字符串
providers: [
  {
    provide: "USER",
    useValue: { user: "test" },
  },
];
// token 为 Symbol
providers: [
  {
    provide: Symbol("user"),
    useClass: UserService,
  },
];
```

**注意**：当自定义 token 是类名时，任何依赖这个类的地方，都会注入提供的实例，甚至提供的可以不是类名对应的类。比如下面这样注册，依赖 ConfigService 的地方，都会被 DevConfigService 覆盖。

```ts
providers: [
  {
    provide: ConfigService,
    useClass: DevConfigService,
  },
];
```

再比如下面这样注册，依赖 ConfigService 的地方，将不再把 ConfigService 解析为类，而是解析为 `useValue` 提供的值。当然这种写法不多见。

```ts
providers: [
  {
    provide: ConfigService,
    useValue: { config: "xxx" },
  },
];
```

## 自定义 token 注册的使用场景

### useClass 注册类

#### 使用场景 1：动态确定 token 解析的类

```ts
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === "development"
      ? DevelopmentConfigService
      : ProductionConfigService,
};
@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

#### 使用场景 2：一个接口或抽象类有多个实现

```ts
interface PaymentService {
  pay(amount: number): void;
}
@Injectable()
class PaypalService implements PaymentService {
  pay(amount: number): void {
    console.log(`Paid ${amount} using PayPal`);
  }
}
@Injectable()
class StripeService implements PaymentService {
  pay(amount: number): void {
    console.log(`Paid ${amount} using Stripe`);
  }
}

@Module({
  providers: [
    { provide: "PAYPAL_SERVICE", useClass: PaypalService },
    { provide: "STRIPE_SERVICE", useClass: StripeService },
  ],
})
class PaymentModule {}

// 使用时明确指定要注入的实现
@Injectable()
class OrderService {
  constructor(
    @Inject("PAYPAL_SERVICE") private readonly paymentService: PaymentService
  ) {}
  processOrder(amount: number) {
    this.paymentService.pay(amount);
  }
}
```

### useValue 注册值

#### 使用场景：注入常量值

```ts
@Module({
  providers: [{ provide: "CONFIG", useValue: { apiKey: "xxx" } }],
})
class ConfigModule {}

@Injectable()
class ApiService {
  constructor(@Inject("CONFIG") private readonly config: { apiKey: string }) {}
  getApiKey(): string {
    return this.config.apiKey;
  }
}
```

### useFactory 注册工厂

#### 使用场景：注入动态依赖

某些场景中，依赖需要在运行时进行配置，比如需要从配置文件或环境变量中读取，可以通过 `useFactory` 动态返回依赖。

```ts
@Module({
  providers: [
    {
      provide: "DATABASE_CONNECTION",
      useFactory: async (optionsProvider: OptionsProvider) => {
        const options = optionsProvider.get();
        const connection = await createDatabaseConnection(options); // 假设这是一个异步函数
        return connection;
      },
      inject: [OptionsProvider],
    },
  ],
})
class DatabaseModule {}

@Injectable()
class UserService {
  constructor(
    @Inject("DATABASE_CONNECTION") private readonly dbConnection: any
  ) {}
  getUser(id: number) {
    return this.dbConnection.findUserById(id);
  }
}
```

`inject` 属性接受 Provider 数组，在实例化过程中，Nest 解析该数组并将其作为参数传递给 `useFactory` 工厂函数。`useFactory` 支持异步。

### useExisting 注册别名

#### 使用场景：为现有的 Provider 提供别名

```ts
@Injectable()
class LoggerService {
  /* implementation details */
}

const loggerAliasProvider = {
  provide: "AliasLoggerService",
  useExisting: LoggerService,
};
@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

上面例子中，LoggerService 和 AliasLoggerService 将解析为同一个实例。

# 注入方式

定义的 Provider 在 `@Module` 中注册后，可以在 Controller 或其他 Provider 中注入使用。

## 基于构造函数的输入

前面举的例子中都是基于构造函数的注入，也细分为两种：`通过类名注入`或`通过 @Inject 注入`，使用哪种和 Provider 注册的方式相关。

### 通过类名注入

如果 Provider 通过类语法糖注册，或通过自定义 token 为类名注册，注入时可以通过类名注入，也可以通过 `@Inject` 装饰器手动注入。

```ts
export class UserController {
  constructor(private userService: UserService) {}
}

export class UserController {
  constructor(@Inject(UserService) private userService: UserService) {}
}
```

### 通过 `@Inject` 装饰器注入

2.  如果 Provider 通过自定义 token 为字符串、Symbol 注册，注入时必须使用 `@Inject` 装饰器手动进行。

```ts
export class UserController {
  constructor(@Inject("CONFIG") private readonly config: { apiKey: string }) {}
}
```

### 通过 `@Optional` 装饰器可选注入

`@Optional` 装饰器用于标记注入的依赖项为可选。即如果该依赖项没有在模块中注册，Nest 不会抛出异常，而是注入 undefined。

```ts
@Injectable()
export class SomeService {
  constructor(
    @Optional() private readonly optionalDependency?: SomeDependencyService
  ) {}
  someMethod() {
    if (this.optionalDependency) {
      // 使用 optionalDependency
    } else {
      // 使用默认逻辑
    }
  }
}
```

## 基于属性的注入

比较少见，直接在类的属性上通过 `@Inject` 装饰器标记，注入依赖项。

```ts
@Injectable()
export class MyController {
  @Inject(MyService)
  private readonly myService: MyService;
}
```

# Provider 的可见范围

Provider 的可见范围，仅限于注册的模块。如果在其他模块中没有注册，但又想使用，可以在当前模块导出，然后在其他模块中导入当前模块，则可以使用这个 Provider。

```ts
// 在 UserModule 中导出 UserService
@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// 在 AppModule 中导入 UserModule
@Module({
  imports: [UserModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

// 可以在 AppService 中使用 UserService 了
@Injectable()
export class AppService {
  constructor(private userService: UserService) {}
}
```

如果既没有在 AppModule 中注册 UserService，又没有导出 UserService，在 AppService 中直接使用 UserService，会抛出如下异常。除非使用上述的 `@Optional` 方式注入。
![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/255e880f71394d12a56992e4b9def8d3~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgcGx1c29uZQ==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTYxNDU1Mzg0Mzc3NzI3MSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1744686780&x-orig-sign=xJNsqZfMSEUAXtizBTh2OT4xo3s%3D)

# scope 定义 Provider 实例化的方式

使用 `@Injectable` 装饰器声明一个类被 IoC 容器接管时，可以传入 `scope` 参数控制其实例化的方式。

有三种 `scope` 配置：

### `Scope.DEFAULT`

默认情况，Provider 在整个生命周期中保持单例。适用于大多数情况。

### `Scope.REQUEST`

每个 HTTP 请求创建一个新的实例。适用于每个请求需要不同实例的服务，比如用户认证和会话管理的场景，每个请求都应该有独立的认证实例。不然比如下面的例子，`this.user` 保存的信息会在不同请求间串。

```ts
@Injectable({ scope: Scope.REQUEST })
export class AuthService {
  private readonly user: any;
  constructor(@Inject(REQUEST) private readonly request: Request) {
    this.user = request.user; // 依赖请求上下文，获取当前用户
  }
  getCurrentUser() {
    return this.user;
  }
}
```

### `Scope.TRANSIENT`

每次注入都创建新的实例。适用于每次都需要全新实例的服务。

# 总结

本章主要介绍了 Nest 中 **Provider** 的基本用法，包括其注册方式和注入方式，以及 `useClass`、`useValue`、`useFactory`、`useExisting` 几种注册方式的使用场景。

Provider 是 Nest 中依赖注入系统的核心单元，灵活和合理的使用它可以覆盖实际开发中的大多场景和提升代码可维护性。
