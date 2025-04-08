Nest 中的**管道**（**Pipe**）在调用请求处理程序之前执行，常用于**验证**和**转换**数据。

# 应用场景

管道的两个典型的应用场景:

- **转换**：将输入数据转换为所需的数据输出（例如将字符串转换为整数）
- **验证**：对输入数据进行验证，验证成功则继续传递，验证失败则抛出异常

这两种情况下，管道会拦截路由处理程序的调用参数，进行转换或验证处理，然后用转换或验证好的参数调用原方法。

# 内置管道

Nest 中有 9 个开箱即用的内置管道：

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`

它们都可以从  `@nestjs/common`  包中导出。

下面具体介绍几个常用的内置管道：

## ParseIntPipe

用于将传入的字符串转换为整数，如果转换失败，将返回错误。类似的还有 `ParseFloatPipe`、`ParseBoolPipe`、`ParseArrayPipe` 等。

### 绑定方式

参数级别绑定，将 `ParseIntPipe`  管道类作为  `@Param()`  装饰器的第二个参数。

```ts
@Get('/:id')
findUserById(@Param('id', ParseIntPipe) id: number) {
  console.log(id, typeof id);
  return `user id ${id}`;
}
```

### 效果

传入字符串 123，转换为数值 123

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/a8c6efe47ae44f878edb4a4437c8697c~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgcGx1c29uZQ==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTYxNDU1Mzg0Mzc3NzI3MSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1744686872&x-orig-sign=s0GhKlflPjLaZjqav0EXcn9shIQ%3D)

传入字符串 abc，转换失败报错

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/fa2a47642638488686bf36d3edd1476a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgcGx1c29uZQ==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTYxNDU1Mzg0Mzc3NzI3MSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1744686872&x-orig-sign=JSeGbi3%2FjX1iLdlbAzNmoMHWKcc%3D)

## DefaultValuePipe

用于设置默认值，如果没有传递参数就使用默认值

### 绑定方式

参数级别绑定

```ts
@Post('/')
createUser(
  @Body(new DefaultValuePipe({ name: 'test', age: 18 })) body: CreateUserDto,
) {
  return body;
}
```

### 效果

不传入参数时，获取到默认值

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/339c1c7cc0f44057abf866f5bc282c8d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgcGx1c29uZQ==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTYxNDU1Mzg0Mzc3NzI3MSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1744686872&x-orig-sign=TqGETpvNXuRuwoauqPicxLYpZco%3D)

## ValidationPipe

用于自动执行输入数据的验证，通常配合 DTO + `class-validator` 库使用。

### 定义 DTO

关于什么是 DTO，之前有过介绍：[DTO](https://juejin.cn/post/7474077946543259702#heading-9)

```ts
export class CreateUserDto {
  name: string;
  age: number;
}
```

### 使用 class-validator 装饰器

安装 `class-validator` 包：

    pnpm i class-validator

使用 `class-validator` 包中的装饰器在 DTO 上声明验证规则：

```ts
import { IsString, IsNumber } from "class-validator";

export class CreateUserDto {
  @IsString()
  name: string;
  @IsNumber()
  age: number;
}
```

### 使用 DTO

```ts
@Post('/')
createUser(@Body() body: CreateUserDto) {
  return this.userService.createUser(body);
}
```

### 绑定方式

全局级别绑定

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

或者参数级别绑定

```ts
@Post('/')
createUser(@Body(new ValidationPipe()) body: CreateUserDto) {
  return this.userService.createUser(body);
}
```

### 效果

当传入 body 对象的属性不符合验证规则时（比如 age 属性传入一个字符串），会报错：

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/d843c24442a84850ae36bfc15a293a68~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAgcGx1c29uZQ==:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiMTYxNDU1Mzg0Mzc3NzI3MSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1744686872&x-orig-sign=uZfdGLxMe52YQtcXbAVVSdzPuOM%3D)

# 管道绑定方式

上面对几个常用内置管道的介绍中，已经使用了全局和参数两种绑定方式。总的来说，管道的绑定方式有以下几种：

## 全局级别

使用 `app.useGlobalPipes` 绑定:

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

## 控制器级别

使用 `@UsePipes()` 装饰器绑定：

```ts
@Controller("/user")
@UsePipes(MyPipe)
export class UserController {}
```

## 方法级别

同样使用 `@UsePipes()` 装饰器绑定：

```ts
@Controller("/user")
@UsePipes(MyPipe)
export class UserController {
  @Post("/")
  @UsePipes(MyPipe)
  createUser(@Body() body: CreateUserDto) {
    return this.userService.createUser(body);
  }
}
```

## 参数级别

直接传入请求参数装饰器使用：

```ts
@Get('/:id')
findUserById(@Param('id', ParseIntPipe) id: number) {
  console.log(id, typeof id);
  return `user id ${id}`;
}
```

# 自定义管道

当 Nest 内置管道无法满足我们的要求时，也可以自定义管道。

## 创建

使用 nest cli 快速创建 pipe 类：

    nest g pipe auth-validate

执行这行命令后，在 src/auth-validate 文件夹中创建 auth-validate.pipe.ts 文件。

```ts
import { ArgumentMetadata, Injectable, PipeTransform } from "@nestjs/common";

@Injectable()
export class AuthValidatePipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

## transform 方法

可以看到自定义管道需要实现 `PipeTransform` 类，这个类有一个 `transform` 方法。

### 参数

`transform` 方法有两个参数：

- `value`：表示当前请求处理程序的参数。

- `metadata`：表示当前请求处理程序参数的元数据。类型为 `ArgumentMetadata`，包含三个属性：
  - `type`：表示参数是一个 body `@Body`、query `@Query`、param `@Param` 还是自定义参数。
  - `metatype`：表示参数的元类型，比如 `string` 等。如果函数签名中没有定义类型则为 `undefined`。
  - `data`：传递给请求参数装饰器的字符串，比如 `@Body("data")`，如果不传入参数，则为 `undefined`。

```ts
interface ArgumentMetadata {
  type: "body" | "query" | "param" | "custom";
  metatype?: Type<unknow>;
  data?: string;
}
```

### 实现

在 `transform` 方法中进行结构转换或验证，验证失败抛出错误。方法的返回值会覆盖原有的参数值。

# 自定义管道实例

下面举几个实例，来更好的了解自定义管道的实现。

## 自定义转换管道

尝试自己实现 `ParseIntPipe` 的功能：

- 将 `transform` 方法获取的 `string` 类型参数值通过 `parseInt` 进行转换。

- 转换后如果不是数值则抛出错误。

- `transform` 方法返回转换后的数值。

```ts
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from "@nestjs/common";

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException("Validation failed");
    }
    return val;
  }
}
```

## 自定义验证管道

实现用户信息验证管道：

- 将 `transform` 方法获取的参数值进行正则校验。

- 校验失败则抛出错误。

- 校验成功则返回原参数值。

```ts
export class UserValidationPipe implements PipeTransform<UserInfo> {
  transform(value: UserInfo, metadata: ArgumentMetadata): UserInfo {
    const checkExp = /^(\+86-)?1[3456789]\d{9}$/;
    if (!checkExp.test(value.phone)) {
      throw new BadRequestException(HttpStatus.BAD_REQUEST);
    }
    return value;
  }
}
```

## 自定义基于 DTO 的验证管道

尝试自己实现 `ValidatePipe` 的功能：

### class-validator 和 class-transformer

基于 DTO 的验证基于 `class-validator` 和 `class-transformer` 库实现，所以先来介绍一下这俩库。

`class-validator`：提供装饰器（`@IsInt()`、`@IsString()` 等）声明验证规则，提供 `validate` 方法基于装饰器进行类型校验。

`class-transformer`：在普通对象和类实例之间进行互转。

#### 使用示例

1.  定义一个类，使用 `class-validator` 提供的装饰器声明校验规则。
2.  使用 `class-transformer` 的 `plainToClass` 方法将普通对象转为类实例。
    3）通过 `class-validator` 的 `validate` 方法验证转换后的类实例是否符合类定义的验证规则。

```ts
import { plainToClass } from "class-transformer";
import { IsInt, IsString, Min, Max, validate } from "class-validator";

class User {
  @IsInt()
  @Min(1)
  id: number;
  @IsString()
  @Min(3)
  @Max(50)
  name: string;
}

async function main() {
  const userObject = {
    id: 1,
    name: "John",
  };
  // 使用 class-transformer 将普通对象转换为类实例
  const userInstance = plainToClass(User, userObject);
  // 使用 class-validator 进行验证
  const validationErrors = await validate(userInstance);
  if (validationErrors.length > 0) {
    console.log("Validation failed. Errors:");
    validationErrors.forEach((error) => {
      console.log(error.property, error.constraints);
    });
  } else {
    console.log("Validation succeeded!");
  }
}
```

### 实现

1.  定义异步 `transform` 方法，以支持 `class-validator` 的异步验证。
2.  从 `transform` 方法的 `metadata` 参数获取 `metatype` 字段，表示参数的元类型。
3.  实现 `toValidate` 方法，验证参数元类型是否为原生 js 类型，是则跳过，因为原生 js 类型不能附加验证装饰器。
4.  使用 `class-transformer` 的 `plainToInstance` 方法将普通 js 对象转为类实例，即可验证对象。
5.  使用 `class-validator` 的 `validate` 方法验证转换后的类实例是否符合类上通过 `class-validator` 提供的装饰器定义的验证规则。
6.  验证不通过抛出异常，验证通过返回原始 value。

```ts
import {
  ArgumentMetadata,
  BadRequestException,
  Injectable,
  PipeTransform,
} from "@nestjs/common";
import { plainToInstance } from "class-transformer";
import { validate } from "class-validator";

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException("Validation failed");
    }
    return value;
  }
  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

## 自定义基于 schema 的验证管道

1.  定义 DTO 并使用

```ts
export class CreateUserDto {
  name: string;
  age: number;
  gender: boolean;
}

@Post()
createUser(@Body() createUserDto: CreateUserDto): string {}
```

2.  使用 `joi` 库根据 DTO 的结构定义 schema

```ts
import Joi from "joi";

export const createUserSchema = Joi.object({
  name: Joi.string().required(),
  age: Joi.number().required(),
  gender: Joi.bool().required(),
});
```

3.  定义 pipe ，实例化时传入定义好的 joi schema 作为参数，`transform` 方法中通过 `schema.validate` 验证传入的参数值，验证失败报错，验证成功则返回原参数。

```ts
@Injectable()
export class JoiValidationPipe implements PipeTransform {
  constructor(private schema: ObjectSchema) {}

  transform(value: any, metadata: ArgumentMetadata) {
    const { error } = this.schema.validate(value);
    if (error) {
      throw new BadRequestException("Validation failed");
    }
    return value;
  }
}
```

```ts
@Post()
@UsePipes(new JoiValidationPipe(createUserSchema))
createUser(@Body() createUserDto: CreateUserDto): string {
  return `${createUserDto.name} is the 100th user`;
}
```

# 总结

Nest 中的 Pipe 有验证和转换两大使用场景。本章先介绍了 9 种内置管道，演示了其基本用法；然后介绍如何自定义管道，并尝试自定义实现了 `ParseIntPipe`、`ValidatePipe` 、`JoiValidationPipe` 等。
