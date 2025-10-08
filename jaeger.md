# Telemetry

## Проблематика
Мы хотим получать больше информации о том, как работают наши микросервисы и их эндпоинты:

 - Метрики
 - Стек трейсы
 - Входные и выходные параметры
 - Анализ ошибок
<br style="page-break-after: always;"/>

Какие есть варианты
 - Console logs (Трудно мониторить удаленные стенды. Нет унификации. Проблемы на больших данных)
 - Elasticsearch + Kibana (Ресурсоёмко)
 - Grafana
 - OpenTelemetry + Jaeger

## Интерфейс Jaeger
### Search

 - **Таймлайн** запросов.
 - **Поверхностная информация** по запросам.
 - **Фильтрация** по сервису, HTTP методу, таймингу, тэгам, успешности и другим полям.


https://www.dropbox.com/scl/fi/jxj66aeejkcoaqqezt1b4/jaeger1.png?rlkey=p486shn5zibihyt07lld9zz6f&st=su5wmixs&dl=0

### Запрос

 - **Стек трейс** запроса.
 - **Входные и выходные параметры** не только всего запроса, но и каждого отдельного шага.
 - **Таймлайн** по всем шагам выполнения запроса.
 - **Ошибки**, возникшие при исполнении, их описание и стек трейс.

**Успешный запрос**

https://www.dropbox.com/scl/fi/o3oedw8fqz4ioys58g8jv/jaeger2.png?rlkey=rshgetjdaalz3wprcf88516h4&st=sex6lzgx&dl=0

**Запрос с ошибкой**

https://www.dropbox.com/scl/fi/0jldk0e2p0hwpqy21k4mx/jaeger4.png?rlkey=ztk21xfgt7g7msvem56ef3ydu&st=9go211wq&dl=0



### Граф запроса

 - Более наглядное изображение шагов исполнения запроса.
https://www.dropbox.com/scl/fi/zs1efsdyn6bxxkddzxy3f/jaeger3.png?rlkey=2gfs2l623c1vfpx0025fu7clz&st=jagpm8xn&dl=0


## Настройка

### Инфраструктура и виртуальное окружение
Добавление необходимых переменных окружение необходимо выполнять через наши infra репозитории (или просто попросить DevOps команду)
```yaml
be-service-worksheet:
  ...
  config:
    ...
    envs:
      ...
      NODE_ENV: 'production'
      ENABLE_OTEL: 'true'
      OTEL_SERVICE_NAME: '{{ .appName }}'
      OTEL_EXPORTER_OTLP_TRACES_ENDPOINT: 'http://otel-telemetry-collector:4318/v1/traces'
      LOG_LEVEL: 'debug'
```

### Библиотека
Добавление .npmrc в корень проекта
```bash
//nexus.int.rolfcorp.ru/repository/npm-group-all/:_authToken=<TOKEN>
registry="https://nexus.int.rolfcorp.ru/repository/npm-group-all/"

always-auth="true"
strict-ssl=false
```
Установка внутренней библиотеки
```bash
npm i @service/lib
```

### Добавление телеметрии в коде
Подключение телеметрии к проекту
```ts
import { bootstrapTelemetry } from '@service/lib';

bootstrapTelemetry({
	NODE_ENV: configService.get<string>('NODE_ENV'),
	ENABLE_OTEL: configService.get<string>('ENABLE_OTEL') ===  'true',
	OTEL_SERVICE_NAME: configService.get<string>('OTEL_SERVICE_NAME'),
	OTEL_EXPORTER_OTLP_TRACES_ENDPOINT: configService.get<string>(
		'OTEL_EXPORTER_OTLP_TRACES_ENDPOINT',
	),
});
```

### JaegerModule (Новое)

Основной модуль для интеграции Jaeger с NestJS приложениями.
Модуль автоматически инициализирует телеметрию и предоставляет готовые сервисы для логирования.

#### Статическая конфигурация (forRoot)

```typescript
import { Module } from '@nestjs/common';
import { JaegerModule } from '@service/lib/telemetry';

@Module({
  imports: [
    JaegerModule.forRoot({
      serviceName: 'my-app',
      tracesEndpoint: 'http://jaeger:4318/v1/traces',
      enableOtel: true,
      nodeEnv: 'production',
      loggerContext: 'MyApp',
      spanName: 'MyAppLogger'
    })
  ]
})
export class AppModule {}
```

#### Асинхронная конфигурация (forRootAsync)

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { JaegerModule } from '@service/lib/telemetry';

@Module({
  imports: [
    ConfigModule.forRoot(),
    JaegerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) =>  {
        return {
          serviceName: configService.get('OTEL_SERVICE_NAME'),
          tracesEndpoint: configService.get('OTEL_EXPORTER_OTLP_TRACES_ENDPOINT'),
          enableOtel: configService.get('ENABLE_OTEL') === 'true',
          nodeEnv: configService.get('NODE_ENV'),
          loggerContext: configService.get('OTEL_SERVICE_NAME'),
          spanName: 'JaegerLoggerService'
        } as JaegerModuleOptions;
      }
    })
  ]
})
export class AppModule {}
```

#### Использование в сервисах

```typescript
import { Injectable } from '@nestjs/common';
import { JaegerNestLoggerService } from '@service/lib/telemetry';

@Injectable()
export class UserService {
  constructor(
    private readonly logger: JaegerNestLoggerService
  ) {}

  async createUser(userData: any) {
    this.logger.log('Создание пользователя', { userId: userData.id });
    // ... логика создания пользователя
  }
}
```

### JaegerNestLoggerWrapper (Новое)

Статический класс-обертка для стандартного NestJS логгера с интеграцией Jaeger.
Автоматически инициализируется при импорте JaegerModule.

#### Инициализация

Телеметрия автоматически инициализируется при импорте JaegerModule. Ручная инициализация не требуется.

```typescript
import { JaegerNestLoggerWrapper } from '@service/lib/telemetry';

// Инициализация происходит автоматически при импорте JaegerModule
// Никаких дополнительных вызовов не требуется
```

#### Использование

```typescript
// Логирование информационных сообщений
JaegerNestLoggerWrapper.log('Приложение запущено.');

// Логирование ошибок
JaegerNestLoggerWrapper.error('Произошла ошибка при подключении к БД.');

// Логирование предупреждений
JaegerNestLoggerWrapper.warn('Высокое потребление памяти.');

// Отладочные сообщения
JaegerNestLoggerWrapper.debug('Текущее состояние кэша.');

// Подробные сообщения
JaegerNestLoggerWrapper.verbose('Детальная информация о запросе.');

// Критические ошибки
JaegerNestLoggerWrapper.fatal('Критическая ошибка системы.');

// Установка уровней логирования
JaegerNestLoggerWrapper.setLogLevels(['error', 'warn', 'log']);
```

#### Особенности

- **Статический класс**: Не требует создания экземпляров
- **Автоматическая инициализация**: Инициализируется при импорте JaegerModule
- **Автоматическая трассировка**: Все логи автоматически отправляются в Jaeger
- **Поддержка всех уровней**: Реализует все методы стандартного NestJS LoggerService

### JaegerNestLoggerService (Новое)

Инжектируемый сервис-обертка для логирования с интеграцией Jaeger.
Предоставляется автоматически при импорте JaegerModule.

#### Автоматическая регистрация

JaegerNestLoggerService автоматически регистрируется при импорте JaegerModule и доступен для инжекции в любом сервисе.

```typescript
import { Module } from '@nestjs/common';
import { JaegerModule } from '@service/lib/telemetry';

@Module({
  imports: [
    JaegerModule.forRoot({
      serviceName: 'my-app',
      tracesEndpoint: 'http://localhost:4318/v1/traces',
      enableOtel: true
    })
  ]
})
export class MyModule {}
```

#### Использование в сервисах

```typescript
import { Injectable } from '@nestjs/common';
import { JaegerNestLoggerService } from '@service/lib/telemetry';

@Injectable()
export class UserService {
  constructor(
    private readonly logger: JaegerNestLoggerService
  ) {}

  async createUser(userData: any) {
    try {
      this.logger.log('Создание нового пользователя', { userId: userData.id });
      
      // Логика создания пользователя
      const user = await this.userRepository.create(userData);
      
      this.logger.log('Пользователь успешно создан', { userId: user.id });
      return user;
    } catch (error) {
      this.logger.error('Ошибка при создании пользователя', { 
        error: error.message, 
        userData 
      });
      throw error;
    }
  }

  async getUserById(id: string) {
    this.logger.debug('Поиск пользователя по ID', { userId: id });
    
    const user = await this.userRepository.findById(id);
    
    if (!user) {
      this.logger.warn('Пользователь не найден', { userId: id });
      return null;
    }
    
    this.logger.verbose('Детали пользователя получены', { 
      userId: id, 
      userRole: user.role 
    });
    
    return user;
  }
}
```

#### Использование в контроллерах

```typescript
import { Controller, Post, Body } from '@nestjs/common';
import { JaegerNestLoggerService } from '@service/lib/telemetry';

@Controller('users')
export class UserController {
  constructor(
    private readonly logger: JaegerNestLoggerService
  ) {}

  @Post()
  async createUser(@Body() userData: any) {
    this.logger.log('Получен запрос на создание пользователя');
    
    try {
      const user = await this.userService.createUser(userData);
      this.logger.log('Пользователь создан успешно', { userId: user.id });
      return user;
    } catch (error) {
      this.logger.error('Ошибка в контроллере', { 
        endpoint: 'POST /users', 
        error: error.message 
      });
      throw error;
    }
  }
}
```

#### Глобальная регистрация

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { JaegerNestLoggerService } from '@service/lib/telemetry';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Получаем логгер который нужно обернуть
  const standardLogger = new ConsoleLogger(); // Или winston, итд
  // Получаем логгер-обертку
  const logger = new JaegerNestLoggerService(standardLogger);

  // Установка глобального логгера
  app.useLogger(logger);
  
  await app.listen(3000);
}
bootstrap();
```

либо, оборачивая логгер по умолчанию:

```typesctipt
  const globalLogger = app.get(JaegerNestLoggerService);
  app.useLogger(globalLogger);

```

### Декорирование классов и методов
Декорирование контроллера
```ts
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  @UseInterceptors(getTelemetryInterceptor())
  async get(): Promise<User[]> {
    return this.userService.get();
  }
}
```

Декорирование метода (Новое. Добавлены параметры logInput, logOutput)
```ts
@Injectable()
export class UserService {
  constructor(private readonly prismaService: PrismaService) {}

  @CoverMethodTelemetry({
    logInput: true,
    logOutput: true,
  })
  async get(): Promise<TestUser[]> {
    return this.prismaService.testUser.findMany({});
  }
}
```

Декорирование класса
```ts
@Injectable()
@CoverClassTelemetry()
export class UserService {
  constructor(private readonly prismaService: PrismaService) {}
}
```

Декорирование класса с автоматическим покрытием его методов (Новое)
```ts
@Injectable()
@CoverClassTelemetry({ coverMethods: true })
export class UserService {
  constructor(private readonly prismaService: PrismaService) {}

  async get(): Promise<TestUser[]> {
    return this.prismaService.testUser.findMany({});
  }
}
```

### Ручное добавление спанов 
```ts
const spanData = jaegerLogger.getSpan('SPAN_NAME');
jaegerLogger.log({
  ok: false,
  span: spanData.span,
  data: { message: 'MyMessage', otherInfo: 123 },
});
```

```ts
getHello(): string {
  setTimeout(() => {
    const spanData = jaegerLogger.getSpan('setTimeout');
    try {
      throw new Error('TEST ERRRROR');
    } catch (err: any) {
      const { message, stack } = err;
      Logger.log('Catched ERROR', message);
      jaegerLogger.log({
        ok: false,
        span: spanData.span,
        data: { message, stack },
      });
    }
  }, 2000);
  return this.appService.getHello();
}
```

## Полезные ссылки

 - Dev Jaeger - https://jaeger-telemetry-dev.flora-dev.int.rolfcorp.ru/search
 - Gitlab service-lib - https://gitlab.int.rolfcorp.ru/one-rolf/be/service/service-lib
 - Gitlab service-template - https://gitlab.int.rolfcorp.ru/one-rolf/be/service/service-template
 - Мой контакт в Time - https://time.rolf.ru/rolftech/messages/@adisupov
