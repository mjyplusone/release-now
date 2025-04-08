# Nest 是啥？

Nest 是一个 nodejs 服务端应用程序开发框架，它有几个特点：

1.  内置支持 Typescript
2.  底层默认使用 Express 框架，也可以通过配置使用 Fastify
3.  核心思想是控制反转和依赖注入

下面我们先创建一个简单的 Nest 项目，通过项目了解 Nest 中的一些基本概念，最后再聊一聊核心思想控制反转（IoC）和依赖注入（DI）是咋回事。

# 创建项目与基本概念

## 创建与启动

1.  全局安装 @nestjs/cli

`pnpm i -g @nestjs/cli`

2.  通过 cli 创建项目

`nest new <project-name>`

3.  启动项目

`cd <project-name>`

`pnpm run start`

可以看到项目已经成功运行起来了。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/83405d9acdd64546ae6d0e1834ad9fff~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgcGx1c29uZQ==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTYxNDU1Mzg0Mzc3NzI3MSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1744686614&x-orig-sign=X5YTFEHuS54Enm19d93JL2eIt4M%3D)

## 项目结构与基本概念

项目的核心结构如下：

```js
src
├── app.controller.spec.ts
├── app.controller.ts
├── app.module.ts
├── app.service.ts
├── main.ts
```

### main

main 文件是应用程序的入口文件，其中通过 NestFactory.create 创建应用实例，然后监听端口。

```js
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

### module

- module 是使用 @Module 装饰器声明的类，其中可以注册其他 module、controller、provider ，也可以导出。
- Nest 中用模块来组织应用程序，每个应用至少需要一个根模块。

```js
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService],
  exports: [],
})
export class AppModule {}
```

### controller

- controller 是被 @Controller 装饰器声明的类。
- 负责处理请求和响应，可以注入 provider，比如下面例子中创建了一个 Get 接口，其中依赖了 AppService。

```js
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get("/hello")
  getHello(): string {
    return this.appService.getHello();
  }
}
```

### provider

- provider 是被 @Injectable 装饰器声明的类。
- 通常用于提供一些具体的功能实现，可以被注入到 controller 或者其他 provider 使用。

```js
import { Injectable } from "@nestjs/common";

@Injectable()
export class AppService {
  getHello(): string {
    return "Hello World!";
  }
}
```

至此，Nest 应用程序对外提供了一个 /hello 接口，返回 "hello world!"，尝试 curl 调用：

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/920b333f7da74b8ba74076bb38ffea05~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgcGx1c29uZQ==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTYxNDU1Mzg0Mzc3NzI3MSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1744686614&x-orig-sign=%2FZ1Lm6xKqMtj7ZrS77VSLK4BiPM%3D)

# 控制反转和依赖注入

控制反转和依赖注入是 Nest 的核心思想，怎么理解呢？我们来举个 🌰。

## 传统写法的问题

一个应用程序的组织，本质上就是很多的类，互相调用来实现整体的功能。比如有两个低层类 Dog 和 Cat，用于创建猫和狗；有一个高层 Animal 类，依赖低层类去创建小动物。下面的写法中，直接在 Animal 中实例化 Dog 类并创建了一只小狗。

```js
class Dog {
    create() {
        console.log("create a dog, wangwang～")
    }
}

class Cat {
    create() {
        console.log("create a cat, miaomiao~")
    }
}

class Animal {
    private dog = new Dog()
    create() {
        this.dog.create()
    }
}
```

这个写法会有什么问题呢？思考一下：

1.  假如需求改变，不想让 Animal 创建一只小狗而是想创建一只小猫，那么需要改变 Animal 类的实现，如果想创建其他小动物就需要一次又一次的改变类的实现，`Animal 和它依赖的类产生了强耦合`。

```js
class Animal {
    private cat = new Cat()
    create() {
        this.cat.create()
    }
}
```

2.  假如还有很多个 Animal2、Animal3 ......，它们都想要创建一只小狗，那么代码就变成下面这样，`Dog 需要被实例化很多次`。

```js
class Animal {
    private dog = new Dog()
    create() {
        this.dog.create()
    }
}

class Animal2 {
    private dog = new Dog()
    create() {
        this.dog.create()
    }
}

class Animal3 {
    private dog = new Dog()
    create() {
        this.dog.create()
    }
}
```

当应用复杂度增加，类和类之间依赖关系复杂度也随之增加，这两个问题会变得越来越严重。

## 改进

那更好的写法是什么呢？当 Animal 类依赖 Dog 类时，可以不在 Animal 中直接创建 Dog 的实例，而是定义一套抽象接口，Animal 类和 Dog 类都依赖于这套抽象去实现，然后在外部工厂类中创建 Dog 实例传入 Animal。即: `Animal 不用关心实例如何创建，只需要按照抽象接口的规范去使用它即可`。

```js
class Animal {
    private animal
    constructor(animal) {
        this.animal = animal
    }
    create() {
        this.animal.create()
    }
}

class Animal2 {
    private animal
    constructor(animal) {
        this.animal = animal
    }
    create() {
        this.animal.create()
    }
}

class Factory {
    create() {
        // 先创建一只狗
        const dog = new Dog()
        const dogAnimal = new Animal(dog)
        dogAnimal.create()
        // 再创建一只猫
        const cat = new Cat()
        const catAnimal = new Animal(cat)
        catAnimal.create()
        // Animal2 复用同一个 Dog 实例创建一只狗
        const animal2 = new Animal2(dog)
        animal2.create()
    }
}

```

这样就解决了上面两个问题：

1.  当我们想从创建一只狗变为创建一只猫时，Animal 类的实现完全不需要改动。
2.  当多个 Animal 都想创建一只狗时，可以复用同一个实例。

这个例子中，将类实例化的过程交给 IoC 容器（即 Factory 类）处理，就是**控制反转（IoC）**；在类之外创建依赖对象并提供给类，就是**依赖注入（DI）**。

## Nest 中的 IoC

Nest 框架本身就是一个 IoC 容器，负责统一管理类的实例化以及注入，开发者不需要手动实例化，只需要向 IoC 容器拿实例化后的对象使用即可。这种将控制权交给框架的做法，有利于实现类和类之间的松耦合。

### 具体实现

1.  module 中通过 @Module 装饰器注册依赖。

2.  通过 @Injectable 装饰器定义 provider，声明这个类被 IoC 容器接管，即实例化过程委托给 IoC 容器。

3.  controller 和 provider 中都可以通过 constructor 注入依赖关系。

```js
// test.module.ts
@Module({
    controllers: [TestController],
    providers: [TestService],
})
export class TestModule {}

// test.service.ts
@Injectable()
export class TestService {
    constructor(private test2Service: Test2Service) {}

    findAll() {}
}

// test.controller.ts
@Controller('test')
export class TestController {
    // 注入对 TestService 的依赖
    constructor(private testService: TestService) {}

    @Get()
    async findAll() {
        return this.testService.findAll();
    }
}
```

# 最后

至此，我们简单了解了 Nest 的几个基本概念 module、controller、provider，以及它的核心思想控制反转与依赖注入。此系列后续文章会更详细的介绍它的使用并对核心源码进行解析。
