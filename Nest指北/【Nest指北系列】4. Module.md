Nest 中使用「**模块（module）**」来组织应用程序。每个应用程序至少有一个根模块，以根模块为起点组织多个功能模块，每个功能模块是对一系列关联紧密功能的封装。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/dd336e585d8d4a1d8c1fa6d946874abb~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgcGx1c29uZQ==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTYxNDU1Mzg0Mzc3NzI3MSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1744686811&x-orig-sign=dzu8NEBt4QG%2FdHEQRyhjHUGtI9s%3D)

# 使用

模块的使用很简单，大概分以下三步：

1.  创建模块，可以使用 nest cli 快速创建。

<!---->

    nest g module user

执行这条命令后，在 src/user 文件夹中创建 user.module.ts 文件：

```ts
import { Module } from "@nestjs/common";

@Module({})
export class UserModule {}
```

2.  可以看出 **module 是被 @Module 装饰器装饰的类**，接下来在 `@Module` 装饰器中定义模块描述对象。其中可以包含 Controller、Provider 和其他 Module，它们一起协同工作以实现特定的功能，比如：

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

3.  将功能模块导入到根模块。

```ts
import { Module } from "@nestjs/common";
import { CatsModule } from "./cats/cats.module";
@Module({
  imports: [CatsModule],
})
export class ApplicationModule {}
```

4.  此时项目基本结构如下，以 app 根模块为起点，导入 user 功能模块。

<!---->

    src
    ｜-- user
        ｜-- user.module.ts
        ｜-- user.controller.ts
        ｜-- user.service.ts
    ｜-- app.module.ts
    ｜-- main.ts

# @Module 装饰器

用于定义模块描述对象，有几个属性：providers、controllers、imports、exports。

```ts
@Module({
  controllers: [UserController],
  providers: [UserService],
  imports: [],
  exports: [],
})
```

## providers

注册 provider 类，有多种注册方式，具体可见[【Nest 指北系列】Provider](https://juejin.cn/post/7478989325494517769#heading-4)。注册后的 provider 可以在整个模块内使用。

在不同模块中注册同一个 provider，默认会共享实例。

## controllers

注册一组控制器。

## imports/exports

- exports：导出由本模块提供，并且需要在其他模块中使用的 Provider。
- imports：导入模块，这些模块导出的 Provider 可以在本模块中使用。

具体的来讲：
模块为其中 Controller、Provider 这些组件提供执行范围。模块中注册的 Provider 对模块的其他成员可见，但当它需要在模块外部可见时，需要从模块中导出，并在其他消费模块导入。

比如，DogModule 中注册的 DogService 可以在 DogController 中使用，当它需要在 CatController 中使用时，需要把它放到 DogModule 中 `@Module` 的 `exports` 数组中，然后在 CatModule 中 `imports` DogModule。

```ts
import { Module } from "@nestjs/common";
import { DogService } from "./dog.service";
import { DogController } from "./dog.controller";

@Module({
  providers: [DogService],
  controllers: [DogController],
  exports: [DogService],
})
export class DogModule {}
```

```ts
import { Module } from "@nestjs/common";
import { DogService } from "src/dog/dog.service";
import { CatController } from "./cat.controller";

@Module({
  imports: [DogService],
  controllers: [CatController],
})
export class CatModule {}
```

# 全局模块

Nest 中 Provider 默认在注册的模块范围内生效，在模块外使用需要在当前模块导出后，在其他模块导入当前模块。这使得一些公共的模块会被多个地方导入使用。

`@Global` 装饰器使模块成为全局模块。全局模块只需要注册一次，就可以全局使用，不再需要在每个使用的地方 imports 它。

一般来说，我们会将一些公共模块标记为全局模块，并在根模块或核心模块注册它。

```ts
@Global()
@Module({
  providers: [CommonService],
  exports: [CommonService],
})
export class CommonModule {}
```

```ts
@Module({
  imports: [CommonModule],
})
export class AppModule {}
```

# 动态模块

动态模块允许在导入时接收参数或配置，动态注册和配置 Provider。它定义的方式是定义一个静态方法，然后在其中返回动态模块属性。

## 静态方法

首先，定义一个静态方法，同步或异步返回动态模块。方法名称任意，但有一些约定的常用命名：

```ts
@Module({})
export class TestModule {
  static forRoot() {}
}
```

- `register`：每次使用时传入配置，比如在模块 A 中导入 `HttpModule.register({ timeout: '2000'})`，在模块 B 中导入 `HttpModule.register({ timeout: '3000'})`。
- `forRoot` / `forRootAsync`：全局传一次配置，一般在根模块中导入。
- `forFeature`：使用 `forRoot` 传入全局配置后，在具体模块导入使用时再传一些配置。比如 `TypeOrmModule` 使用 `forRoot` 指定数据库连接信息，再用 `forFeature` 在模块级别注入实体。

```ts
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: "mysql",
      host: "localhost",
      port: 3306,
      username: "root",
      password: "password",
      database: "test_db",
      entities: [],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

```ts
@Module({
  imports: [TypeOrmModule.forFeature([User])], // 注册 User 实体
  controllers: [UserController],
  providers: [UserService],
})
export class UserModule {}
```

## 动态模块属性

1.  静态方法返回的动态模块属性和静态模块 `@Module` 中的属性基本一致，仅在此基础上增加一个 `module` 属性，用作模块名称。**该名称必填，且必须与模块类名一致**。

2.  动态模块返回的属性扩展 `@Module` 装饰器中定义的元数据。
    比如：下面代码中，`forRoot` 方法返回的 providers 属性中注册了两个 Provider ：`ConfigService` 和 `CONFIG_OPTIONS`，但它们不会覆盖 `@Module` 中定义的 providers， `DefaultService` 依然生效。

```ts
@Module({
  providers: [DefaultService],
})
export class TestModule {
  static forRoot(config: ConfigOptions): DynamicModule {
    return {
      module: TestModule,
      providers: [
        {
          provide: CONFIG_OPTIONS,
          useValue: config,
        },
        CustomService,
      ],
      exports: [CustomService],
    };
  }
}
```

3.  如果要将动态模块声明为全局模块，可以在返回的属性中增加 `global: true`。

```ts
{
  global: true,
  module: TestModule,
  providers: [],
  exports: [],
}
```

4.  动态模块也可以接收 `useFactory` 和 `inject` 这种动态的参数，用于注册模块中的 Provider。useFactory 方式注册 Provider 具体可以见[【Nest 指北系列】Provider](https://juejin.cn/post/7478989325494517769#heading-13)。这种接收动态参数的动态模块，一般 `forRootAsync` 这种约定命名的静态方法去定义。

```ts
TestModule.forRootAsync({
    imports: [ConfigModule],
    useFactory: (configService: ConfigService) => {
        return configService.get('xxx')
    },
    inject: [ConfigService]
}),
```

```ts
@Module({})
export class TestModule {
    static forRootAsync(options): DynamicModule {
        return {
            module: TestModule,
            imports: options.imports || [],
            providers: [
                {
                    provide: "config",
                    useFactory: options.useFactory,
                    inject: options.inject || [],
                },
                {
                    // ...
                },
            ],
            exports: [...],
        }
    }
}
```

## 导入动态模块

动态模块导入的方式和静态模块一致，传入 `@Module` 装饰器的 `imports` 属性即可。

```ts
@Module({
  imports: [TestModule.forRoot({})],
})
export class AppModule {}
```

如果要重新导出导入的动态模块，可以在 `exports` 数组中省略 `forRoot` 的调用。

```ts
@Module({
  imports: [TestModule.forRoot({})],
  exports: [TestModule],
})
export class AppModule {}
```

# 总结

本章主要介绍了 Nest 中的**模块**，包括使用 `@Module` 装饰器定义模块描述对象组织应用程序，模块的导入导出，全局模块和动态模块的定义。

同时，模块也是 Nest DI 系统的起点。Nest 中的依赖注入，就是从根模块开始进行依赖扫描，递归寻找 module，同时寻找每个 module 依赖的组件（Provider、Controller），构建整体依赖树。然后根据构造函数参数，找到对应的 Provider，深度优先遍历，逐步实例化。这个将在之后的源码分析章节中细说。
