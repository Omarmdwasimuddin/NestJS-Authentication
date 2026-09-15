# Authentication (NestJS)

**Source:** https://docs.nestjs.com/security/authentication

বেশিরভাগ application এর জন্যই Authentication একটা **essential** অংশ। Authentication handle করার অনেক approach আর strategy আছে। কোন project এ কোন approach নেওয়া হবে সেটা নির্ভর করে সেই application এর নির্দিষ্ট requirement এর উপর। এই chapter এ এমন কয়েকটা approach দেখানো হবে, যেগুলো বিভিন্ন ধরনের requirement অনুযায়ী adapt করা যায়।

চলো আমাদের requirement টা একটু clear করে নেই। এই use case এ, client প্রথমে username আর password দিয়ে authenticate করবে। Authenticate হয়ে গেলে, server একটা JWT issue করবে, যেটা পরবর্তী request গুলোতে authorization header এ [bearer token](https://tools.ietf.org/html/rfc6750) হিসেবে পাঠিয়ে authentication প্রমাণ করা যাবে। আমরা একটা protected route ও বানাবো, যেটাতে শুধু valid JWT থাকা request দিয়েই access করা যাবে।

প্রথমে শুরু করব প্রথম requirement দিয়ে: user কে authenticate করা। এরপর সেটাকে extend করে JWT issue করব। শেষে, request এ valid JWT আছে কিনা সেটা check করে এমন একটা protected route বানাব।

---

## 1. Authentication Module তৈরি করা

প্রথমে আমরা একটা `AuthModule` generate করব, এবং তার ভেতরে `AuthService` আর `AuthController`। Authentication এর logic implement করার জন্য `AuthService` ব্যবহার করব, আর authentication endpoint গুলো expose করার জন্য `AuthController`।

```bash
nest g module auth
nest g controller auth
nest g service auth
```

`AuthService` implement করার সময় user সংক্রান্ত operation গুলো একটা `UsersService` এ encapsulate করলে সুবিধা হবে, তাই এখনই সেই module আর service টা generate করে নেই:

```bash
nest g module users
nest g service users
```

এই generate হওয়া file গুলোর default content নিচের মতো replace করো। আমাদের sample app এ, `UsersService` শুধু একটা hard-coded in-memory user list রাখে, আর username দিয়ে একটা user খুঁজে বের করার জন্য একটা find method রাখে। Real app এ, এই জায়গাতেই তুমি তোমার পছন্দের library (যেমন TypeORM, Sequelize, Mongoose ইত্যাদি) ব্যবহার করে user model আর persistence layer বানাবে।

```typescript
import { Injectable } from '@nestjs/common';

// এটা আসলে একটা real class/interface হওয়া উচিত, যেটা user entity represent করে
export type User = any;

@Injectable()
export class UsersService {
  private readonly users = [
    {
      userId: 1,
      username: 'john',
      password: 'changeme',
    },
    {
      userId: 2,
      username: 'maria',
      password: 'guess',
    },
  ];

  async findOne(username: string): Promise<User | undefined> {
    return this.users.find((user) => user.username === username);
  }
}
```

`UsersModule` এ, শুধু একটাই change দরকার — `@Module` decorator এর exports array এ `UsersService` কে add করা, যাতে এই module এর বাইরেও এটা visible থাকে (আমরা শীঘ্রই এটা `AuthService` এ ব্যবহার করব)।

```typescript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service.js';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

---

## 2. "Sign in" Endpoint Implement করা

আমাদের `AuthService` এর কাজ হলো একটা user খুঁজে বের করা এবং password verify করা। এই purpose এ আমরা একটা `signIn()` method বানাব। নিচের code এ, user object return করার আগে ES6 spread operator ব্যবহার করে password property বাদ দেওয়া হয়েছে। User object return করার সময় এটা একটা common practice, কারণ password বা অন্য কোনো sensitive field expose করা উচিত না।

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service.js';

@Injectable()
export class AuthService {
  constructor(private readonly usersService: UsersService) {}

  async signIn(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const { password, ...result } = user;
    // TODO: এখানে user object এর বদলে JWT generate করে সেটা return করতে হবে
    return result;
  }
}
```

> **⚠️ Warning:** Real application এ, password কখনোই plain text এ store করবে না। এর বদলে [bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme) এর মতো একটা library ব্যবহার করবে, salted one-way hash algorithm সহ। এই approach এ, শুধু hashed password store করবে, আর stored password কে incoming password এর hashed version এর সাথে compare করবে — ফলে plain text এ কখনো password store বা expose হবে না। Sample app টা simple রাখার জন্য আমরা এই rule violate করে plain text ব্যবহার করছি। **Real app এ এটা কখনো করবে না!**

এখন আমরা `AuthModule` কে update করব যাতে `UsersModule` import করা হয়।

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service.js';
import { AuthController } from './auth.controller.js';
import { UsersModule } from '../users/users.module.js';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  controllers: [AuthController],
})
export class AuthModule {}
```

এবার `AuthController` open করে এতে একটা `signIn()` method add করি। Client এই method কে call করবে user কে authenticate করার জন্য। এটা request body তে username আর password receive করবে, এবং user authenticate হলে একটা JWT token return করবে।

```typescript
import { Body, Controller, Post, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service.js';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }
}
```

> **Hint:** `Record<string, any>` type ব্যবহার করার বদলে, ideally request body এর shape define করার জন্য একটা DTO class ব্যবহার করা উচিত। বিস্তারিত জানার জন্য [validation](https://docs.nestjs.com/techniques/validation) chapter দেখো।

---

## 3. JWT Token

এবার আমরা auth system এর JWT অংশে যাব। আমাদের requirement টা একটু review আর refine করি:

- User দের username/password দিয়ে authenticate করতে দেওয়া, এবং পরবর্তী protected API endpoint call গুলোর জন্য একটা JWT return করা। আমরা এই requirement এর কাছাকাছি চলে এসেছি। এটা সম্পূর্ণ করতে, JWT issue করার code লিখতে হবে।
- Bearer token হিসেবে valid JWT থাকার উপর ভিত্তি করে protected API route তৈরি করা।

JWT requirement এর জন্য আরেকটা package install করতে হবে:

```bash
npm install --save @nestjs/jwt
```

> **Hint:** `@nestjs/jwt` package ([এখানে আরো দেখো](https://github.com/nestjs/jwt)) একটা utility package, যেটা JWT manipulation এ সাহায্য করে — যার মধ্যে JWT token generate এবং verify করা দুটোই আছে।

Service গুলো clean আর modular রাখার জন্য, JWT generate করার কাজটা আমরা `authService` এ করব। `auth` folder এর `auth.service.ts` file open করো, `JwtService` inject করো, এবং `signIn` method কে update করো যাতে নিচের মতো JWT token generate হয়:

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { UsersService } from '../users/users.service.js';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async signIn(
    username: string,
    pass: string,
  ): Promise<{ access_token: string }> {
    const user = await this.usersService.findOne(username);
    if (user?.password !== pass) {
      throw new UnauthorizedException();
    }
    const payload = { sub: user.userId, username: user.username };
    return {
      // 💡 এখানে payload sign করার জন্য যে JWT secret key ব্যবহার হচ্ছে,
      // সেটাই হলো JwtModule এ পাস করা key
      access_token: await this.jwtService.signAsync(payload),
    };
  }
}
```

আমরা `@nestjs/jwt` library ব্যবহার করছি, যেটা `signAsync()` function দেয় — এটা দিয়ে `user` object এর কিছু property থেকে JWT generate করা হয়, যেটা আমরা তারপর একটা simple object এ `access_token` নামে single property হিসেবে return করি। লক্ষ্য করো: `userId` value রাখার জন্য আমরা property এর নাম `sub` দিয়েছি, JWT standard এর সাথে সামঞ্জস্য রাখার জন্য।

এখন `AuthModule` কে update করতে হবে নতুন dependency import আর `JwtModule` configure করার জন্য।

প্রথমে, `auth` folder এ `constants.ts` তৈরি করো এবং নিচের code add করো:

```typescript
export const jwtConstants = {
  secret:
    'DO NOT USE THIS VALUE. INSTEAD, CREATE A COMPLEX SECRET AND KEEP IT SAFE OUTSIDE OF THE SOURCE CODE.',
};
```

JWT sign আর verify — দুই step এই key শেয়ার করার জন্য আমরা এটা ব্যবহার করব।

> **⚠️ Warning:** এই key কখনো publicly expose করবে না। এখানে code টা clear করার জন্য এভাবে দেখানো হয়েছে, কিন্তু production system এ **এই key protect করতেই হবে** — secrets vault, environment variable, বা configuration service এর মতো উপযুক্ত ব্যবস্থা ব্যবহার করে।

এবার `auth` folder এর `auth.module.ts` open করো এবং নিচের মতো update করো:

```typescript
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service.js';
import { UsersModule } from '../users/users.module.js';
import { JwtModule } from '@nestjs/jwt';
import { AuthController } from './auth.controller.js';
import { jwtConstants } from './constants.js';

@Module({
  imports: [
    UsersModule,
    JwtModule.register({
      global: true,
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

> **Hint:** আমরা `JwtModule` কে global হিসেবে register করছি, কাজ সহজ করার জন্য। মানে হলো, application এর আর কোথাও `JwtModule` আলাদা করে import করার দরকার নেই।

`JwtModule` কে আমরা `register()` দিয়ে configure করছি, একটা configuration object পাস করে। Nest এর `JwtModule` সম্পর্কে আরো জানতে [এখানে](https://github.com/nestjs/jwt/blob/master/README.md) দেখো, আর available configuration option গুলো সম্পর্কে [এখানে](https://github.com/auth0/node-jsonwebtoken#usage)।

চলো আবার cURL দিয়ে আমাদের route গুলো test করি। `UsersService` এ hard-code করা যেকোনো `user` object দিয়ে test করতে পারবে।

```bash
# POST to /auth/login
curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
# Note: উপরের JWT টা truncated
```

---

## 4. Authentication Guard Implement করা

এবার আমরা শেষ requirement টা handle করতে পারি: endpoint গুলোকে protect করা, যাতে request এ valid JWT থাকা বাধ্যতামূলক হয়। এর জন্য আমরা একটা `AuthGuard` তৈরি করব, যেটা দিয়ে route গুলো protect করা যাবে।

```typescript
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request } from 'express';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      // 💡 এখানে payload verify করার জন্য যে JWT secret key ব্যবহার হচ্ছে,
      // সেটাই হলো JwtModule এ পাস করা key
      const payload = await this.jwtService.verifyAsync(token);
      // 💡 আমরা payload কে এখানে request object এ assign করছি,
      // যাতে route handler গুলোতে এটা access করা যায়
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

এবার আমরা আমাদের protected route implement করতে পারি, আর সেটা protect করার জন্য `AuthGuard` register করতে পারি।

`auth.controller.ts` file open করো এবং নিচের মতো update করো:

```typescript
import {
  Body,
  Controller,
  Get,
  HttpCode,
  HttpStatus,
  Post,
  Request,
  UseGuards,
} from '@nestjs/common';
import { AuthGuard } from './auth.guard.js';
import { AuthService } from './auth.service.js';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @HttpCode(HttpStatus.OK)
  @Post('login')
  signIn(@Body() signInDto: Record<string, any>) {
    return this.authService.signIn(signInDto.username, signInDto.password);
  }

  @UseGuards(AuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

আমরা এইমাত্র বানানো `AuthGuard` টা `GET /profile` route এ apply করছি, যাতে এটা protected হয়।

App টা running আছে কিনা নিশ্চিত করে, `cURL` দিয়ে route গুলো test করো।

```bash
# GET /profile
curl http://localhost:3000/auth/profile
{"statusCode":401,"message":"Unauthorized"}

# POST /auth/login
curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."}

# GET /profile — আগের step এ পাওয়া access_token টা bearer code হিসেবে ব্যবহার করে
curl http://localhost:3000/auth/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
{"sub":1,"username":"john","iat":...,"exp":...}
```

লক্ষ্য করো, `AuthModule` এ আমরা JWT এর expiration `60 seconds` set করেছিলাম। এটা expiration এর জন্য অনেক কম সময়, আর token expiration আর refresh এর detail এই article এর scope এর বাইরে। তবে JWT এর একটা গুরুত্বপূর্ণ বৈশিষ্ট্য দেখানোর জন্যই আমরা এটা বেছে নিয়েছি। Authenticate করার পর যদি ৬০ সেকেন্ড অপেক্ষা করে তারপর `GET /auth/profile` request করো, তাহলে `401 Unauthorized` response পাবে। কারণ `@nestjs/jwt` automatically JWT এর expiration time check করে, ফলে application এ আলাদা করে সেটা করার ঝামেলা থাকে না।

এখানেই আমাদের JWT authentication implementation সম্পূর্ণ হলো। JavaScript client গুলো (যেমন Angular/React/Vue) এবং অন্যান্য JavaScript app এখন আমাদের API Server এর সাথে securely authenticate আর communicate করতে পারবে।

---

## 5. Global ভাবে Authentication Enable করা

তোমার বেশিরভাগ endpoint যদি default ভাবেই protected থাকা উচিত হয়, তাহলে authentication guard কে একটা [global guard](https://docs.nestjs.com/guards#binding-guards) হিসেবে register করতে পারো, আর প্রতিটা controller এর উপরে `@UseGuards()` decorator ব্যবহার করার বদলে, শুধু কোন কোন route public হবে সেটা flag করে দিতে পারো।

প্রথমে, `AuthGuard` কে global guard হিসেবে register করো (যেকোনো module এ, যেমন `AuthModule` এ):

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: AuthGuard,
  },
],
```

এটা করার পর, Nest automatically সব endpoint এ `AuthGuard` bind করে দেবে।

এখন route গুলোকে public হিসেবে declare করার একটা mechanism দিতে হবে। এর জন্য, `SetMetadata` decorator factory function ব্যবহার করে একটা custom decorator বানানো যায়।

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

উপরের file এ, আমরা দুটো constant export করেছি। একটা হলো আমাদের metadata key, নাম `IS_PUBLIC_KEY`, আর অন্যটা হলো আমাদের নতুন decorator, যার নাম দিয়েছি `Public` (চাইলে এর নাম `SkipAuth` বা `AllowAnon` — যেটা তোমার project এর সাথে যায়, সেটাই দিতে পারো)।

এখন যেহেতু আমাদের কাছে একটা custom `@Public()` decorator আছে, এটা দিয়ে যেকোনো method decorate করা যায়, এভাবে:

```typescript
@Public()
@Get()
findAll() {
  return [];
}
```

সবশেষে, `"isPublic"` metadata পাওয়া গেলে `AuthGuard` কে `true` return করতে হবে। এর জন্য আমরা `Reflector` class ব্যবহার করব (বিস্তারিত জানতে [এখানে](https://docs.nestjs.com/guards#putting-it-all-together) দেখো)।

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private readonly jwtService: JwtService,
    private reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(
      IS_PUBLIC_KEY,
      [context.getHandler(), context.getClass()],
    );
    if (isPublic) {
      // 💡 এই condition টা লক্ষ্য করো
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    if (!token) {
      throw new UnauthorizedException();
    }
    try {
      // 💡 এখানে payload verify করার জন্য যে JWT secret key ব্যবহার হচ্ছে,
      // সেটাই হলো JwtModule এ পাস করা key
      const payload = await this.jwtService.verifyAsync(token);
      // 💡 আমরা payload কে এখানে request object এ assign করছি,
      // যাতে route handler গুলোতে এটা access করা যায়
      request['user'] = payload;
    } catch {
      throw new UnauthorizedException();
    }
    return true;
  }

  private extractTokenFromHeader(request: Request): string | undefined {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : undefined;
  }
}
```

---

## 6. Passport Integration

[Passport](https://github.com/jaredhanson/passport) হলো node.js এর সবচেয়ে জনপ্রিয় authentication library, যেটা community তে খুব পরিচিত এবং অনেক production application এ সফলভাবে ব্যবহার হচ্ছে। `@nestjs/passport` module ব্যবহার করে এই library কে একটা **Nest** application এর সাথে integrate করা খুবই straightforward।

Passport কে NestJS এর সাথে কীভাবে integrate করবে সেটা জানতে, এই [chapter](https://docs.nestjs.com/recipes/passport) টা দেখো।

---

## 7. Example

এই chapter এর সম্পূর্ণ code [এখানে](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt) পাবে।
