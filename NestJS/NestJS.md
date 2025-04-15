# NestJS

NestJS - это прогрессивный фраемворк для NodeJS, основанный на TypeScript, он использует модульную архитектуру и встроенный DI(Dependency Injection) что делает код более структурированным и поддерживаемым

## Преимущества перед Express/Fastify

1. Использует TS из коробки
2. Встроенная поддержка solid принципов
3. Гибкая структура на основе модулей
4. Интеграция с TypeORM, Prisma, GraphQL и т.п.
5. Поддержка микросервисов и GRPC
6. Встроенный DI контейнер

## Основные концепции

1. Модули (@Modules)
   Все NestJS приложения состоят из модулей. Это позволяет организовать код и переиспользовать зависимости.

2. Контроллеры (@Controllers)
   Отвечают за обработку входящих HTTP - запросов

3. Сервисы (@Injectable)
   Используются для бизнес-логики и инъекции зависимостей

## Как работает Dependency Injection(DI) в NestJS

DI - позволяет автоматически передавать зависимости между классами(NestJS сам инстанцирует классы, в ручную создавать экземпляры классов не нужно, и все эти классы являются singleton)

## Хуки NestJS

NestJS предоставляет хуки
onModuleInit() - вызывается при инизиализации модуля
onApplicationBootstrap() - вызывается после запуска приложения
beforeApplicationShutdown() - вызывается перед завершением приложения

## Что такое Repository Pattern

Repository Pattern - это паттерн который позволяет абстрагироваться от конкретной реализации бд
(@InjectRepository)

## Разница между Middleware, Guards, Pipes, Interceptors

Middleware - Работает перед попаданием запроса в контроллер(логирование)

Guards - Определяет может ли пользователь выполнить действие(авторизация)

Pipes - Проверяет и преобразует входные данные (валидация)

Interceptors - Меняет данные до или после выполнения запроса (логирование, кэширование)

## Важен ли порядок импортов в @Module

В большинстве случаев порядок не критичен.
Когда порядок важен

1. Когда один модуль зависит от другого(Например TypeOrmModule должен загружаться после ConfigModule, т.к. он использует ConfigService)
2. Глобальные модули должны загружаться раньше

---

В `imports:[]` у **@Module** указываются только модули(ни сервисы, ни контроллеры только модули).
NestJS использует модульную архитектуру и модули служат контейнерами для провайдеров(сервисов, контроллеров) и других зависимостей.
Когда мы импортируем модуль в imports NestJS автоматически делает доступными его публичные провайдеры для других модулей

## providers и exports для чего нужны

В NestJS модули управляют зависимостями с помощью providers и exports

**providers** - регистрирует сервисы(и другие провайдеры) внутри того же модуля
**exports** - позволяет сделать провайдер доступным для других модулей, которые импортируют этот модуль

Простыми словами
**providers** - Какие есть сервисы внутри этого модуля
**exports** - Какие сервисы этот модуль делает доступными для других модулей

### Best Practice

1. Всегда указывай providers, если сервис используется только в этом модуле
2. Добавляй exports, если сервис нужен в других модулях
3. Если exports используется, обязательно добавляй этот модуль в imports другого модуля

---

# Команды

`npm i -g @nestjs/cli`
`nest new <название проекта>`
`nest g module <название модуля>`

# Пример структуры проекта

```
src/
│── auth/
│   ├── dto/
│   ├── entities/
│   ├── guards/              👈 Здесь храним `AuthGuard`
│   │   ├── jwt-auth.guard.ts 👈 Реализация `AuthGuard`
│   ├── strategies/
│   │   ├── jwt.strategy.ts   👈 Реализация `JwtStrategy`
│   ├── auth.module.ts      <-- 📌 Импортируется в `app.module.ts`
│   ├── auth.controller.ts
│   ├── auth.service.ts
│── users/
│   ├── users.module.ts      <-- 📌 Импортируется в `app.module.ts`
│   ├── users.controller.ts
│   ├── users.service.ts
│── app.module.ts
│── main.ts
```

---

Для работы с jwt
@nestjs/jwt
@nestjs/passport
passport-jwt

Для работы с typeorm
@nestjs/typeorm
typeorm

Для создания DTO(data transfer object)
class-validator
class-transformer

# useClass, useValue, useFactory

## useClass

useClass позволяет заменить класс, который по умолчанию внедряется в DI

```
@Injectable()
export class UserService {
  getUsers() {
    return ['User1', 'User2'];
  }
}

@Injectable()
export class MockUserService {
  getUsers() {
    return ['TestUser1', 'TestUser2'];
  }
}

@Module({
  providers: [
    {
      provide: UserService, // 👈 Заменяем оригинальный UserService
      useClass: MockUserService,
    },
  ],
})
export class AppModule {}
```

## useValue

Если нам не нужен класс, а просто объект, число или строка, то можно передавать это значение черех useValue

```
const config = {
  apiKey: '123456789',
  databaseUrl: 'postgres://localhost',
};

@Module({
  providers: [
    {
      provide: 'CONFIG',  // 👈 Уникальный токен
      useValue: config,   // 👈 Просто передаем объект
    },
  ],
  exports: ['CONFIG'],
})
export class ConfigModule {}

---

@Injectable()
export class AppService {
  constructor(@Inject('CONFIG') private config) {}

  getApiKey() {
    return this.config.apiKey;
  }
}

```

## useFactory

Если зависимость должна быть создана на основе других сервисов или изменяться динамически, используем useFactory

```
@Module({
  imports: [ConfigModule],
  providers: [
    {
      provide: 'JWT_SECRET',
      useFactory: (configService: ConfigService) => configService.get('JWT_SECRET'),
      inject: [ConfigService], // 👈 Внедряем ConfigService
    },
  ],
  exports: ['JWT_SECRET'],
})
export class AuthModule {}
```
