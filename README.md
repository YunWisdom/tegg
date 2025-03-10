# `@eggjs/tegg`

## Install

```shell
# tegg 注解
npm i --save @eggjs/tegg
# tegg 插件
npm i --save @eggjs/tegg-plugin
```

## Config

```js
// config/plugin.js
exports.tegg = {
  package: '@eggjs/tegg-plugin',
  enable: true,
};
```

## Usage

### 原型
module 中的对象基本信息，提供了

- 实例化方式：每个请求实例化/全局单例/每次注入都实例化
- 访问级别：module 外是否可访问



#### ContextProto
每次请求都会实例化一个 ContextProto，并且只会实例化一次


##### 定义
```typescript
@ContextProto(params: {
  // 原型的实例化名称
  // 默认行为：会把 Proto 的首字母转为小写
  // 如 UserAdapter 会转换为 userAdapter
  // 如果有不符合预期的可以手动指定,比如
  // @ContextProto({ name: 'mistAdapter' })
  // MISTAdapter
  // MISTAdapter 的实例名称即为 mistAdapter
  name?: string;

  // 对象是在 module 内可访问还是全局可访问
  // PRIVATE: 仅 module 内可访问
  // PUBLIC: 全局可访问
  // 默认值为 PRIVATE
  accessLevel?: AccessLevel;
})
```
##### 示例
###### 简单示例
```typescript
import { ContextProto } from '@eggjs/tegg';

@ContextProto()
export class HelloService {
  async hello(): Promise<string> {
    return 'hello, module!';
  }
}

```
###### 复杂示例
```typescript
import { ContextProto, AccessLevel } from '@eggjs/tegg';

@ContextProto({
  accessLevel: AccessLevel.PUBLIC,
  name: 'helloInterface',
})
export default class HelloService {
  async hello(user: User): Promise<string> {
    const echoResponse = await this.echoAdapter.echo({ name: user.name });
    return `hello, ${echoResponse.name}`;
  }
}
```
#### SingletonProto
整个应用声明周期只会实例化一个 SingletonProto


##### 定义
```typescript
@SingletonProto(params: {
  // 原型的实例化名称
  // 默认行为：会把 Proto 的首字母转为小写
  // 如 UserAdapter 会转换为 userAdapter
  // 如果有不符合预期的可以手动指定,比如
  // @SingletonProto({ name: 'mistAdapter' })
  // MISTAdapter
  // MISTAdapter 的实例名称即为 mistAdapter
  name?: string;

  // 对象是在 module 内可访问还是全局可访问
  // PRIVATE: 仅 module 内可访问
  // PUBLIC: 全局可访问
  // 默认值为 PRIVATE
  accessLevel?: AccessLevel;
})
```
##### 示例
###### 简单示例
```typescript
import { SingletonProto } from '@eggjs/tegg';

@SingletonProto()
export class HelloService {
  async hello(): Promise<string> {
    return 'hello, module!';
  }
}

```
###### 复杂示例
```typescript
import { SingletonProto, AccessLevel } from '@eggjs/tegg';

@SingletonProto({
  accessLevel: AccessLevel.PUBLIC,
  name: 'helloInterface',
})
export class HelloService {
  async hello(user: User): Promise<string> {
    const echoResponse = await this.echoAdapter.echo({ name: user.name });
    return `hello, ${echoResponse.name}`;
  }
}
```
#### 生命周期 hook
由于对象的生命周期交给了容器来托管，代码中无法感知什么时候对象初始化，什么时候依赖被注入了。所以提供了生命周期 hook 来实现这些通知的功能。


##### 定义
```typescript
/**
 * lifecycle hook interface for egg object
 */
interface EggObjectLifecycle {
  /**
   * call after construct
   */
  postConstruct?(): Promise<void>;

  /**
   * call before inject deps
   */
  preInject?(): Promise<void>;

  /**
   * call after inject deps
   */
  postInject?(): Promise<void>;

  /**
   * before object is ready
   */
  init?(): Promise<void>;

  /**
   * call before destroy
   */
  preDestroy?(): Promise<void>;

  /**
   * destroy the object
   */
  destroy?(): Promise<void>;
}
```
##### 示例
```typescript
import { EggObjectLifecycle } from '@eggjs/tegg';

@ContextProto()
export class Foo implements EggObjectLifecycle {
  // 构造函数
  constructor() {
  }

  async postConstruct(): Promise<void> {
    console.log('对象构造完成');
  }

  async preInject(): Promise<void> {
    console.log('依赖将要注入');
  }

  async postInject(): Promise<void> {
    console.log('依赖注入完成');
  }

  async init(): Promise<void> {
    console.log('执行一些异步的初始化过程');
  }

  async preDestroy(): Promise<void> {
    console.log('对象将要释放了');
  }

  async destroy(): Promise<void> {
    console.log('执行一些释放资源的操作');
  }
}
```


### 注入
Proto 中可以依赖其他的 Proto，或者 egg 中的对象。


#### 定义
```typescript
@Inject(param?: {
  // 注入对象的名称，在某些情况下一个原型可能有多个实例
  // 比如说 egg 的 logger
  // 默认为属性名称
  name?: string;
  // 注入原型的名称
  // 在某些情况不希望注入的原型和属性使用一个名称
  // 默认为属性名称
  proto?: string;
})
```
#### 示例
##### 简单示例
```typescript
import { EggLogger } from 'egg';
import { Inject } from '@eggjs/tegg';

@ContextProto()
export class HelloService {
  @Inject()
  logger: EggLogger;

  async hello(user: User): Promise<string> {
    this.logger.info(`[HelloService] hello ${user.name}`);
    const echoResponse = await this.echoAdapter.echo({ name: user.name });
    return `hello, ${echoResponse.name}`;
  }
}
```
##### 复杂示例
```typescript
import { EggLogger } from 'egg';
import { Inject } from '@eggjs/tegg';

@ContextProto()
export class HelloService {
  // 在 config.default.js 中
  // 配置了 customLogger，
  // 名称为 bizLogger
  @Inject({ name: 'bizLogger' })
  logger: EggLogger;

  async hello(user: User): Promise<string> {
    this.logger.info(`[HelloService] hello ${user.name}`);
    const echoResponse = await this.echoAdapter.echo({ name: user.name });
    return `hello, ${echoResponse.name}`;
  }
}
```
#### 限制

- ContextProto 可以注入任何 Proto 但是 SingletonProto 不能注入 ContextProto
- 原型之间不允许有循环依赖，比如 Proto A - inject -> Proto B - inject- > Proto A，这种是不行的
- 类似原型之间不允许有循环依赖，module 直接也不能有循环依赖
- 一个 module 内不能有实例化方式和名称同时相同的原型
- **不可以注入 ctx/app，用什么注入什么**



#### 兼容 egg
egg 中有 extend 方式，可以将对象扩展到 Context/Application 上，这些对象均可注入。Context 上的对象类比为 ContextProto，Application 上的对象类比为 SingletonProto。


#### 进阶
### module 内原型名称冲突
一个 module 内，有两个原型，原型名相同，实例化不同，这时直接 Inject 是不行的，module 无法理解具体需要哪个对象。这时就需要告知 module 需要注入的对象实例化方式是哪种。


###### 定义
```typescript
@InitTypeQualifier(initType: ObjectInitType)
```
###### 示例
```typescript
import { EggLogger } from 'egg';
import { Inject, InitTypeQualifier, ObjectInitType } from '@eggjs/tegg';

@ContextProto()
export class HelloService {
  @Inject()
  // 明确指定示例化方式为 CONTEXT 的 logger
  @InitTypeQualifier(ObjectInitType.CONTEXT)
  logger: EggLogger;
}
```
##### module 间原型名称冲突
可能多个 module 都实现了名称为 `HelloService` 的原型，需要明确的告知 module 需要注入的原型来自哪个 module.


###### 定义
```typescript
@ModuleQualifier(moduleName: string)
```
###### 示例
```typescript
import { EggLogger } from 'egg';
import { Inject, InitTypeQualifier, ObjectInitType } from '@eggjs/tegg';

@ContextProto()
export class HelloService {
  @Inject()
  // 明确指定使用来自 foo module 的 HelloAdapter
  @ModuleQualifier('foo')
  helloAdapter: HelloAdapter;
}
```

### 单测

#### 单测 Context
在单测中需要获取 egg Context 时，可以使用以下 API。


```typescript
export interface Application {
  /**
   * 创建 module 上下文
   */
  mockModuleContext(data?: any): Promise <Context> ;
  /**
   * 销毁 module 上下文
   */
  destroyModuleContext(context: Context): Promise <void> ;
}
```


#### 获取对象实例
在单测中需要获取 module 中的对象实例时，可以使用以下 API。

```typescript
export interface Application {
  /**
   * 通过一个类来获取实例
   * 注：app.getEggObject 只能获取 Singleton 实例
   */
  getEggObject<T> (clazz: EggProtoImplClass<T> ): Promise <T>;
}

export interface Context {
  /**
   * 通过一个类来获取实例
   * 注：ctx.getEggObject 可以获取 Singleton/Context 实例
   */
  getEggObject<T> (clazz: EggProtoImplClass<T> ): Promise <T>;
}
```

### Egg 兼容性

目前 module 尚未实现所有 egg 的特性，如果需要使用 egg 的功能，可以通过一些方式来兼容。


##### Schedule
目前 Schedule 尚未实现注解，仍然需要使用 egg 的目录和继承方式，在这种场景下如果需要使用 module 的实现，需要使用 `ctx.beginModuleScope`。举个例子：


```typescript
// notify/EC_FOO.js

import { Subscription, Context } from 'egg;

class FooSubscriber extends Subscription {
  private readonly ctx: Context;

  constructor(ctx: Context) {
    super(ctx);
  }

  async subscribe(msg) {
    await ctx.beginModuleScope(async () => {
      await ctx.module.fooService.hello(msg);
    });
  }
}

module.exports = Subscription;

```


#### 注入 egg 对象
module 会自动去遍历 egg 的 Application 和 Context 对象，获取其所有的属性，所有的属性都可以进行无缝的注入。举个例子，如何注入现在的 egg proxy:


```typescript
import { IProxy } from 'egg'

@ContextProto()
class FooService {
  @Inject()
  private readonly proxy: IProxy;

  get fooFacade() {
    return this.proxy.fooFacade;
  }
}
```


##### 注入 logger
专为 logger 做了优化，可以直接注入 custom logger。


举个例子:


有一个自定义的 fooLogger
```javascript
// config.default.js
exports.customLogger = {
  fooLogger: {
    file: 'foo.log',
  },
};
```


代码中可以直接注入:
```typescript
import { EggLogger } from 'egg';

class FooService {
  @Inject()
  private fooLogger: EggLogger;
}
```


#### 注入 egg 方法

由于 module 注入时，只可以注入对象，不能注入方法，如果需要使用现有 egg 的方法，就需要对方法进行一定的封装。

举个例子：假设 context 上有一个方法是 `getHeader`，在 module 中需要使用这个方法需要进行封装。
```typescript
// extend/context.ts

export default {
  getHeader() {
    return '23333';
  }
}

```


先将方法封装成一个对象。
```typescript
// HeaderHelper.ts

class HeaderHelper {
  constructor(ctx) {
    this.ctx = ctx;
  }

  getHeader(): string {
    return this.ctx.getHeader();
  }
}
```

再将对象放到 Context 扩展上即可。
```typescript
// extend/context.ts

const HEADER_HELPER = Symbol('context#headerHelper');

export default {
  get headerHelper() {
    if (!this[HEADER_HELPER]) {
      this[HEADER_HELPER] = new HeaderHelper(this);
    }
    return this[HEADER_HELPER];
  }
}

```

### 生命周期

在 module 中，每个对象实例都有自己的生命周期，开发者可以对每个对象进行细致的控制。只要为对象实现 module 定义好的接口即可。所有生命周期 hook 均为可选方法，不需要的可以不实现。

#### 接口定义
```typescript
interface EggObjectLifecycle {
  /**
   * 在对象的构造函数执行完成之后执行
   */
  async postConstruct?();

  /**
   * 在注入对象依赖之前执行
   */
  async preInject?();

  /**
   * 在注入对象依赖之后执行
   */
  async postInject?();

  /**
   * 执行对象自定义异步初始化函数
   */
  async init?();

  /**
   * 在对象释放前执行
   */
  async preDestroy?();

  /**
   * 释放对象依赖的底层资源
   */
  async destroy?();
}
```


#### 实现
```typescript
import { EggObjectLifecycle } from '@eggjs/tegg';

@SingletonProto()
class FooService implement EggObjectLifecycle {
  @Inject()
  cacheService: CacheService;

  cache: Record<string, string>;

  async init() {
    this.cache = await this.cacheService.get(key);
  }
}
```

### 异步任务
module 在请求结束后会把请求相关的对象释放，所以在请求中使用 `process.nextTick`、 `setTimeout`、 `setInterval`这类接口并不安全，可能导致一些错误。因此需要使用框架提供的接口，以便框架了解当前请求有哪些异步任务在执行，等异步任务执行完成后再释放对象。


#### 安装
```shell
npm i --save @eggjs/tegg-background-task
```


#### 使用
```typescript
import { BackgroundTaskHelper } from '@eggjs/tegg-background-task';

@ContextProto()
export default class BackgroundService {
  @Inject()
  private readonly backgroundTaskHelper: BackgroundTaskHelper;

  async backgroundAdd() {
    this.backgroundTaskHelper.run(async () => {
      // do the background task
    });
  }
}
```


#### 超时时间
框架不会无限的等待异步任务执行，在默认 5s 之后如果异步任务还没有完成则会放弃等待开始执行释放过程。如果需要等待更长的时间，建议有两种方式：

- **推荐方式：将异步任务转发给单例对象（SingletonProto）来执行，单例对象永远不会释放**
- 调整超时时间，对 `backgroundTaskHelper.timeout` 进行赋值即可

### 动态注入

#### 使用

定义一个抽象类和一个类型枚举。
```ts
export enum HelloType {
  FOO = 'FOO',
  BAR = 'BAR',
}

export abstract class AbstractHello {
  abstract hello(): string;
}
```

定义一个自定义枚举。
```ts
import { ImplDecorator, QualifierImplDecoratorUtil } from '@eggjs/tegg';
import { ContextHelloType } from '../FooType';
import { AbstractContextHello } from '../AbstractHello';

export const HELLO_ATTRIBUTE = 'HELLO_ATTRIBUTE';

export const Hello: ImplDecorator<AbstractHello, typeof HelloType> =
  QualifierImplDecoratorUtil.generatorDecorator(AbstractHello, HELLO_ATTRIBUTE);


```

实现抽象类。
```ts
import { ContextProto } from '@eggjs/tegg';
import { ContextHello } from '../decorator/Hello';
import { ContextHelloType } from '../FooType';
import { AbstractContextHello } from '../AbstractHello';

@ContextProto()
@Hello(HelloType.BAR)
export class BarHello extends AbstractHello {
  hello(): string {
    return `hello, bar`;
  }
}

```

动态获取实现。
```ts
import { EggObjectFactory } from '@eggjs/tegg';

@ContextProto()
export class HelloService {
  @Inject()
  private readonly eggObjectFactory: EggObjectFactory;
  
  async hello(): Promise<string> {
    const helloImpl = await this.eggObjectFactory.getEggObject(AbstractHello, HelloType.BAR);
    return helloImpl.hello();
  }
}
```

