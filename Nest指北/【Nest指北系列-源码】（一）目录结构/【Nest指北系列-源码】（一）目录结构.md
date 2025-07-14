这个系列前面的文章已经将 Nest 的核心思想以及核心概念过了一遍，并举例了一些使用场景。相信看过这些介绍后，应该可以上手使用 Nest 开发项目。

> 指路之前的文章：
>
> - [【Nest 指北系列】初识](https://juejin.cn/post/7471453409704919091)
>
> - [【Nest 指北系列】Controller](https://juejin.cn/post/7474077946543259702)
>
> - [【Nest 指北系列】Provider](https://juejin.cn/post/7478989325494517769)
>
> - [【Nest 指北系列】Module](https://juejin.cn/post/7481595366623674420)
>
> - [【Nest 指北系列】中间件](https://juejin.cn/post/7485200128730218507)
>
> - [【Nest 指北系列】异常过滤器](https://juejin.cn/post/7487815126302457856)
>
> - [【Nest 指北系列】管道](https://juejin.cn/post/7490445394468913164)
>
> - [【Nest 指北系列】守卫](url)
>
> - [【Nest 指北系列】拦截器](url)

从这章开始，我们将更进阶的了解 Nest 的实现思想，对其源码进行分析。

# 获取源码

首先，从 github 找到 Nest 的[源码](https://github.com/nestjs/nest)，git clone 到本地。

# 目录结构

## 根目录

根目录结构大致如下：

```
nestjs/
├── benchmarks/
├── packages/
├── sample/
├── integration/
├── tools/
├── scripts/
├── package.json
└── tsconfig.json
```

其中每个子目录的分工为：

- `benchmarks`: 性能测试相关代码
- `integration`: 集成测试相关代码
- `packages`: **核心代码，每个子目录对应一个核心模块包**
- `sample`: 示例项目代码
- `scripts`: 发布脚本、CI 工具脚本等
- `tools`: 构建和性能测试相关的工具函数
- `tsconfig.json`: Typescript 配置

## packages 目录

下面来看下包含核心源码的 packages 目录：

```
packages/
├── common/
├── core/
├── microservices/
├── platform-express/
├── platform-fastify/
|—— platform-socket.io/
|—— platform-ws/
|—— websockets/
├── testing/
└── ...
```

它的每个子目录都是一个模块，可以单独发布到 npm，比如 `@nestjs/common`、`@nestjs/core`。

### common

```
common/
├── decorator/
├── exceptions/
├── pipes/
├── enums/
├── utils/
└── ...
```

包含全平台通用的公共模块，比如：

- `decorator`: 内置装饰器，比如模块装饰器 `@Module`、控制器装饰器 `@Controller`、依赖注入装饰器 `@Injectable`，`@UsePipes`、`@UseGuards` 等，以及 http 相关 `@Get`、`@Post` 等等
- `exceptions`: 内置异常处理类，比如 `HttpException`、`NotFoundException`、`ForbiddenException` 等
- `pipes`: 内置管道，比如：`ParseIntPipe`、`ValidationPipe` 等
- `enums`: 枚举类，比如 http 状态码等
- `utils`: 工具类
- ...

### core

整个框架最核心逻辑，包含依赖注入系统、模块系统、反射机制、路由等。

```
core/
├── guards/
├── interceptors/
├── middleware/
├── services/
├── router/
├── injector/
├── nest-application.ts
├── nest-factory.ts
├── scanner.ts
├── ...
```

- `guards`: 守卫
- `interceptors`: 拦截器
- `middleware`: 中间件
- `services`: 服务，比如 Reflector 类
- `router`: 控制器路由注册逻辑
- `injector`: 依赖注入相关逻辑，包含 NestContainer（IoC 容器）、Module 等定义
- `nest-application.ts`: NestApplication 类
- `nest-factory.ts`: NestFactory 类
- `scanner.ts`: 模块依赖关系扫描

### microservices

微服务模块

### platform-express / platform-fastify

- 用于在 nest 中集成 express、fastify 框架。
- 默认使用 express 基础框架，但 nest 不依赖其中任何一个。
- Fastify 是一个更快速、低开销的 Web 框架，对性能要求高的应用场景会使用到它。

### platform-socket.io / platform-ws / websockets

WebSocket 支持模块

### testing

提供测试工具，如 `Test.createTestingModule()`。

# 总结

本章介绍了 Nest 源码的目录结构，代码按照功能模块组织，遵循单一职责原则。接下来我们对源码的分析会集中在 `packages/common` 和 `packages/core` 目录中。
