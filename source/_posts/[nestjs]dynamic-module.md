---
title: NestJS dynamic module 직접 만들어보기
date: 2021-11-12 19:00:22
index_img:
category: 글쓰기
tags:
  - 글쓰기
math: true
mermaid: true
sticky: 100
author: maskman
---
### 소개

---

*A module is a class annotated with a `@Module()` decorator. The `@Module()` decorator provides metadata that **Nest** makes use of to organize the application structure.*

Nest 에서는 `@Module()` 데코레이터를 이용해 모듈 클래스를 생성하고 등록할 수 있다.

특히 동적 모듈 (Dynamic Module) 기능을 이용해서 Provider 를 동적으로 등록하고 구성 할 수 있는 커스텀 가능한 모듈을 쉽게 만들 수 있다.

Nestjs에서 공식적으로 지원하는 JwtModule 을 분석해보고, 직접 만들어보는 과정을 정리해보자.

### 분석

---

먼저 JwtModule 이 어떻게 사용되는지 살펴보겠다.

```jsx
// auth.module.ts
// https://github.com/nestjs/nest/blob/master/sample/19-auth-jwt/src/auth/auth.module.ts

import { JwtModule } from '@nestjs/jwt';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

NestJS 공식 샘플 코드중 JwtModule 이 등록되는 부분이다.

module.register(options) 와 같은 형태로 사용할 모듈의 static method 를 옵션과 함께 호출하여
동적으로 프로바이더를 생성한다.

위와 같이 등록해주면 아래와 같이 jwtService 를 이용할 수 있게 된다

```jsx
// auth.service.ts
// https://github.com/nestjs/nest/blob/master/sample/19-auth-jwt/src/auth/auth.service.ts

import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
  ) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user: any) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

그럼 이제 JwtModule 이 어떻게 정의되어있는지 살펴보자.

[https://github.com/nestjs/jwt/tree/master/lib](https://github.com/nestjs/jwt/tree/master/lib)

폴더구조는 아래와 같다.

```jsx
lib
	├── interfaces
	├── index.ts
	├── jwt.constants.ts
	├── jwt.module.ts
	├── jwt.providers.ts
	└── jwt.service.ts
```

가장 먼저 등록되는 부분을 담당하는 jwt.module.ts 파일을 살펴보자.

```jsx
import { DynamicModule, Module, Provider } from '@nestjs/common';
import {
  JwtModuleAsyncOptions,
  JwtModuleOptions,
  JwtOptionsFactory
} from './interfaces/jwt-module-options.interface';
import { JWT_MODULE_OPTIONS } from './jwt.constants';
import { createJwtProvider } from './jwt.providers';
import { JwtService } from './jwt.service';

@Module({
  providers: [JwtService],
  exports: [JwtService]
})
export class JwtModule {
  static register(options: JwtModuleOptions): DynamicModule {
    return {
      module: JwtModule,
      providers: createJwtProvider(options)
    };
  }

  static registerAsync(options: JwtModuleAsyncOptions): DynamicModule {
    return {
      module: JwtModule,
      imports: options.imports || [],
      providers: this.createAsyncProviders(options)
    };
  }

  private static createAsyncProviders(
    options: JwtModuleAsyncOptions
  ): Provider[] {
    if (options.useExisting || options.useFactory) {
      return [this.createAsyncOptionsProvider(options)];
    }
    return [
      this.createAsyncOptionsProvider(options),
      {
        provide: options.useClass,
        useClass: options.useClass
      }
    ];
  }

  private static createAsyncOptionsProvider(
    options: JwtModuleAsyncOptions
  ): Provider {
    if (options.useFactory) {
      return {
        provide: JWT_MODULE_OPTIONS,
        useFactory: options.useFactory,
        inject: options.inject || []
      };
    }
    return {
      provide: JWT_MODULE_OPTIONS,
      useFactory: async (optionsFactory: JwtOptionsFactory) =>
        await optionsFactory.createJwtOptions(),
      inject: [options.useExisting || options.useClass]
    };
  }
}
```

이 중 NestJS 공식 샘플에서 사용된 register method  와 관련된 부분을 살펴보면 아래와 같다

```jsx
@Module({
  providers: [JwtService],
  exports: [JwtService]
})
export class JwtModule {
	static register(options: JwtModuleOptions): DynamicModule {
	    return {
	      module: JwtModule,
	      providers: createJwtProvider(options)
	    };
	  }
}
	
export function createJwtProvider(options: JwtModuleOptions): any[] {
  return [{ provide: JWT_MODULE_OPTIONS, useValue: options || {} }];
}

export const JWT_MODULE_OPTIONS = 'JWT_MODULE_OPTIONS';
```

register 함수는 JwtService 가 provide 되고, export 된 JwtModule 을 사용할 수 있도록
NestJS 모듈 객체 형태로 리턴해준다.

이때 사용자가 입력한 옵션을 파라미터로 받아 동적 프로바이더를 생성해 주기 때문에

이 모듈을 사용하는 모든 곳에서 같은 암호화 옵션, 비밀키 등을 공유 할 수 있게 해준다.

또한, registerAsync 라는 다른 등록 구문을 활용해서 useClass, useExisting, useFactory 같은 구문을 이용해 다른 클래스를 추가로 이용하거나, 이미 존재하는 프로바이더를 alias 해서 사용하거나, 비동기적으로 생성 또한 가능하다.

```jsx
static registerAsync(options: JwtModuleAsyncOptions): DynamicModule {
    return {
      module: JwtModule,
      imports: options.imports || [],
      providers: this.createAsyncProviders(options)
    };
  }

  private static createAsyncProviders(
    options: JwtModuleAsyncOptions
  ): Provider[] {
    if (options.useExisting || options.useFactory) {
      return [this.createAsyncOptionsProvider(options)];
    }
    return [
      this.createAsyncOptionsProvider(options),
      {
        provide: options.useClass,
        useClass: options.useClass
      }
    ];
  }

  private static createAsyncOptionsProvider(
    options: JwtModuleAsyncOptions
  ): Provider {
    if (options.useFactory) {
      return {
        provide: JWT_MODULE_OPTIONS,
        useFactory: options.useFactory,
        inject: options.inject || []
      };
    }
    return {
      provide: JWT_MODULE_OPTIONS,
      useFactory: async (optionsFactory: JwtOptionsFactory) =>
        await optionsFactory.createJwtOptions(),
      inject: [options.useExisting || options.useClass]
    };
  }
```

Dynamic 모듈을 만들기 위한 필요 요건을 정리해보면 아래와 같다.

1. 모듈을 등록할 수 있는 static register, static registerAsync 메소드가 포함된 모듈 클래스
2. 모듈에서 제공할 서비스로직이 포함된 주입 가능한 서비스 클래스
3. 동적으로 넘겨지는 객체를 주입받아 사용할 수 있도록 미리 선언된 상수와 동적 프로바이더 생성 로직

### 직접 만들어보기

---

NodeJS 에서 기본으로 제공하는 util 라이브러리중 inspect 라는 메소드가 있다.

[https://nodejs.org/api/util.html#utilinspectobject-options](https://nodejs.org/api/util.html#utilinspectobject-options)

이를 래핑하는 동적 모듈을 만들어보자

The `util.inspect()` method returns a string representation of `object` that is intended for debugging. The output of `util.inspect` may change at any time and should not be depended upon programmatically. Additional `options` may be passed that alter the result. `util.inspect()` will use the constructor's name and/or `@@toStringTag` to make an identifiable tag for an inspected value.

이 메소드는 javscript object 디버깅하거나, 읽기 쉽게 변형해서 string 으로 리턴해주는 유틸 메소드다.

아래 예시처럼 동작한다.

```jsx
const util = require('util');

const o = {
  a: [1, 2, [[
    'Lorem ipsum dolor sit amet,\nconsectetur adipiscing elit, sed do ' +
      'eiusmod \ntempor incididunt ut labore et dolore magna aliqua.',
    'test',
    'foo']], 4],
  b: new Map([['za', 1], ['zb', 'test']])
};
console.log(util.inspect(o, { compact: true, depth: 5, breakLength: 80 }));

// { a:
//   [ 1,
//     2,
//     [ [ 'Lorem ipsum dolor sit amet,\nconsectetur [...]', // A long line
//           'test',
//           'foo' ] ],
//     4 ],
//   b: Map(2) { 'za' => 1, 'zb' => 'test' } }
```

먼저, 필요 요건들을 만들기 전에 어떻게 등록할지 정의해보자.

```jsx
import { Module } from '@nestjs/common';
import { InspectModule } from './inspect/inspect.module';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
    InspectModule.register({showHidden: false, depth: null, colors: true}),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

맨 처음 JwtModule 을 어떻게 등록하는지 살펴 봤던 그 파일이다. 
JwtModule 아래에 InspectModule을 등록하는 부분을 추가해주었다.

이제 필요요건들을 하나씩 만들어보자.

- 모듈을 등록할 수 있는 static register, static registerAsync 메소드가 포함된 모듈 클래스

```jsx
import { DynamicModule, Module, Provider } from '@nestjs/common';
import { INSPECT_MODULE_OPTION } from './inspect.constants';
import { InspectAsyncOptions, InspectOptions, InspectOptionsFactory } from './inspect.interfaces';
import { createInspectProviders } from './inspect.providers';
import { InspectService } from './inspect.service';

@Module({
  providers: [InspectService],
  exports: [InspectService]
})
export class InspectModule {
  public static register(options: InspectOptions): DynamicModule {
    return {
      module: InspectModule,
      providers: createInspectProviders(options),
    };
  }

  static registerAsync(options: InspectAsyncOptions): DynamicModule {
    return {
      module: InspectModule,
      imports: options.imports || [],
      providers: this.createAsyncProviders(options)
    };
  }

  private static createAsyncProviders(
    options: InspectAsyncOptions
  ): Provider[] {
    if (options.useExisting || options.useFactory) {
      return [this.createAsyncOptionsProvider(options)];
    }
    return [
      this.createAsyncOptionsProvider(options),
      {
        provide: options.useClass,
        useClass: options.useClass
      }
    ];
  }

  private static createAsyncOptionsProvider(
    options: InspectAsyncOptions
  ): Provider {
    if (options.useFactory) {
      return {
        provide: INSPECT_MODULE_OPTION,
        useFactory: options.useFactory,
        inject: options.inject || []
      };
    }
    return {
      provide: INSPECT_MODULE_OPTION,
      useFactory: async (optionsFactory: InspectOptionsFactory) =>
        await optionsFactory.createInspectOptions(),
      inject: [options.useExisting || options.useClass]
    };
  }
}
```

- 모듈에서 제공할 서비스로직이 포함된 주입 가능한 서비스 클래스

```jsx
import { Inject, Injectable } from '@nestjs/common';
import util, { InspectOptions } from 'util';
import { INSPECT_MODULE_OPTION } from './inspect.constants';

@Injectable()
export class InspectService {
    constructor(@Inject(INSPECT_MODULE_OPTION) private readonly options: InspectOptions) {}

    inspect(object: any): string {
        return util.inspect(object, this.options)
    }
}
```

- 동적으로 넘겨지는 객체를 주입받아 사용할 수 있도록 미리 선언된 상수와 동적 프로바이더 생성 로직

```jsx
import { Provider } from "@nestjs/common";
import { INSPECT_MODULE_OPTION } from "./inspect.constants";
import { InspectOptions } from "./inspect.interfaces";

export function createInspectProviders(options: InspectOptions): Provider[] {
    return [
      {
        provide: INSPECT_MODULE_OPTION,
        useValue: options || {},
      },
    ];
  }
```

```jsx
import { ModuleMetadata, Type } from '@nestjs/common';
import { InspectOptions } from 'util';

const INSPECT_MODULE_OPTION = "INSPECT_MODULE_OPTION"

export { INSPECT_MODULE_OPTION }
```

sample src : [https://github.com/seonjl/nestjs-inspect/tree/main/sample](https://github.com/seonjl/nestjs-inspect/tree/main/sample)

참고한 레퍼런스 

- [https://dev.to/nestjs/advanced-nestjs-how-to-build-completely-dynamic-nestjs-modules-1370](https://dev.to/nestjs/advanced-nestjs-how-to-build-completely-dynamic-nestjs-modules-1370)