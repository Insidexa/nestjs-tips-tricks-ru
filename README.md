# Содержание
 - [Как заинжектить класс динамически](#dynamic-inject-provider)
 - [Можно ли полностью отключать модуля взависимости от конфига ConfigService](#dynamic-module-disabling)
 - [Как заинжектить класс в `main.ts`](#inject-provider-in-maints)
 - [CLI на минималках](#creating-cli)
 - [Queue Resolver - Nest Way](#queue-resolver)
 - [Как запускать и отлаживать приложение через Nest CLI и зачем он нужен](#start-with-nest-cli)
   - [Не работает отладка в WebStorm (JetBrains, Intellij IDEA)](#debug-in-idea)
   - [Пути в Stacktrace ошибок указывают на js файлы в dist, а не на исходные ts файлы](#stactrace-with-source-maps)

### Как заинжектить класс динамически ? <a id="dynamic-inject-provider"></a>

В NestJS существует [ModuleRef](https://docs.nestjs.com/fundamentals/module-ref).  
Этот провайдер позволяет инжектить сервисы на ходу в приложении.  
Например  

```typescript
const service = this.moduleRef.get<Service>(Service);
```

Осталось только заинжектить `ModuleRef` у ваш провайдер.  

Обратите внимание на то что если вы хотите заинжектить провайдер, не только с текущего модуля, то вторым параметром передаем опцию `{ strict: false }`.  

Зачем это нужно ?  
В случаях кодга нужно получить класс взависимости от условия, что бы не инжектить в конструктор кучу провайдеров. 
Допустим у нас есть несколько провайдеров для разных `API` сервисов.  
Нужно использовать один из адаптеров взависимости от параметра который прилетел с `Request`.  
Дабы не инжектить все адаптеры.  


### Можно ли полностью отключать модуля взависимости от конфига ConfigService ? <a id="dynamic-module-disabling"></a>

Ответ: нет.

Можно это сделать с помощью environment variables.  
На примере псевдокода:  

```typescript
const modules = [ modules ]

if (process.env.ENABLE_SOME_MODULE) {
  modules.push(SomeModule)
}

@Module({
  imports: modules
})
```


### Как заинжектить класс в `main.ts` ? <a id="inject-provider-in-maints"></a>

Это делается очень просто:  

```typescript
const service = app.get<ServiceName>(ServiceName);

service.callMethod();
```


### Как сделать на минималках cli ? <a id="creating-cli"></a>

Есть такая штука в несте как [Application Context](https://docs.nestjs.com/standalone-applications).  
Эта вещь позволяет инициализировать только провайды у ваших модулях без инициализации `HTTP` части, то есть без контроллеров.  


Создаем отдельно файл в `src` с названием `cli.ts`.

```typescript
import { NestFactory, ModuleRef } from '@nestjs/core';
import { CLIModule } from './cli.module';
import { cliCommandInvoker } from './cli-command-invoker';
import { cliCommands } from './cli-commands';

async function bootstrapCLI() {
    const cli = await NestFactory.createApplicationContext(CLIModule);
    const moduleRef = cli.get(ModuleRef);
    cli.enableShutdownHooks();
    const [,,command] = process.argv;

    cliCommandInvoker(moduleRef, cliCommands, command);
}

bootstrapCLI();
```

Создаем `CLIModule` который будет содержать в себе `cli` команды.  

```typescript
import { Module } from '@nestjs/common';
import { HelloCommand } from './hello.command';

@Module({
    imports: [],
    providers: [
        HelloCommand,
    ]
})
export class CLIModule {
}
```

Создаем interface для наших `cli` комманд.  

```typescript
export interface CLICommand {
    invoke(...args);
}
```

Создаем нашу первую `cli` команду `HelloCommand`.  

```typescript
import { CLICommand } from './command';
import { Injectable } from '@nestjs/common';

@Injectable()
export class HelloCommand implements CLICommand {
    public invoke() {
        console.log('Hello world');
    }
}
```

Следующим шагом замапим наши команды на наши классы. Сделаем это с помощью `Map`.  
Создадим файл `cli-commands.ts`.  

```typescript
export const cliCommands = new Map<string, Type<CLICommand>>();
cliCommands.set('hello', HelloCommand);
```

Теперь нужно создать функцию для вызова `cli` команд `cliCommandInvoker`.  
Создадим файл `cli-command-invoker.ts`  

```typescript
import { Type } from '@nestjs/common';
import { ModuleRef } from '@nestjs/core';
import { CLICommand } from './command';

export function cliCommandInvoker(moduleRef: ModuleRef, commands: Map<string, Type<CLICommand>>, command: string) {
    const commandHandlerClass = commands.get(command);
    const commandHandler = moduleRef.get<CLICommand>(commandHandlerClass);
    commandHandler.invoke(process.argv);
}
```

Пользуемся так  

```bash
npx ts-node ./src/cli.ts hello
```

Выхлоп  

```
[Nest] 548231   - 2020-04-17 6:45 PM   [NestFactory] Starting Nest application...
[Nest] 548231   - 2020-04-17 6:45 PM   [InstanceLoader] CLIModule dependencies initialized +23ms
Hello world
```


### Как сделать поиск классов и методов класса по metadata если вы захотите сделать что-то в стиле nest-way с декоратоми и куртизанками ? <a id="queue-resolver"></a>

Данная статья является примером и не претендует на production ready код.  
Пример является реальным поскольку на момент разработки бекенда на `NestJS` в начале 2019 года не было модуля для очередей.  
На момент написания статьи в NestJS уже сущестует данный функционал - [Queue](https://docs.nestjs.com/techniques/queues).  

Например нужно нужно сделать обработчик очереди и он должен выглядеть как класс с методом  `invoke` который и будет выполнять таску  
К примеру прикрутим сюда `bulljs`.  

Напишем прототип чего мы хотим  

```typescript
class SomePayload {
  constructor(public text: string) {}
}

@Queue('QueueName', jobOptions)
@Injectable()
export class QueueHandler implements Consumer {

    public async invoke(bullJob: Bull.Job<SomePayload>) {
        console.log(bullJob.data)
    }
}
```

И как будет выглядеть вызов этой таски из вашего сервиса:  

```typescript
this.bull.queue('QueueName').add(queuePayload, options);
```

Будем создавать постепенно недостающие части для нашей фичи.  

Для начала давайте сделаем декоратор для обработчика очереди:

```typescript
import { Injectable, SetMetadata } from '@nestjs/common';
import * as Bull from 'bull';

export const QUEUE_ARGS_METADATA = '__queue-args-metadata__';

export function Queue(name: string, options?: Bull.QueueOptions): ClassDecorator {
    return (target: object) => {
        SetMetadata(QUEUE_ARGS_METADATA, { name, options })(
            target,
        );
    };
}
```

Создадим интерфейс для класса который будет служить обработчиком очереди:  
Мы же должны четко понимать какой метод класса должен быть вызван для обработки очереди.  

```typescript
export interface Consumer {
    invoke(job: Bull.Job<Dictionary>): unknown;
}
```


Теперь нужно просканировать `NestContainer` ( DI Container ) и выбрать наши классы "обработчики" для очереди.  
Напишем так званый `QueueExplorer` задача которого прошелестеть `NestContainer` и отфильтровать классы.  

```typescript
import { flattenDeep, compact } from 'lodash';
import * as Bull from 'bull';
import { ModulesContainer } from '@nestjs/core/injector/modules-container';
import { Injectable } from '@nestjs/common';

interface QueueInfo {
    name: string;
    options?: Bull.QueueOptions;
    instance: Consumer;
}

@Injectable()
export class QueueExplorer {
    constructor(
        private modulesContainer: ModulesContainer,
    ) {
    }

    // преобразуем древовидную структуру NestContainer в плоскую и фильтруем инстансы
    public explore(): QueueInfo[] {
        const components = [
            ...this.modulesContainer.values(),
        ].map(module => module.providers);

        return compact(flattenDeep(
            components.map(component =>
                [...component.values()]
                    .map(({ instance }) => this.filterCommands(instance as Consumer)),
            ),
        ));
    }

    // фильтруем наши классы по наличии QUEUE_ARGS_METADATA метадаты в провайдере
    protected filterCommands(instance: Consumer) {
        if (!instance) {
            return;
        }

        const metadata = Reflect.getMetadata(QUEUE_ARGS_METADATA, instance.constructor);

        if (metadata === undefined) {
            return;
        }

        return { ...metadata, instance };
    }
}
```

Теперь нужно связать наши обработчики с `bulljs`.  
Добавляем наш `QueueExplorer` в массив провайдеров, допустим, в `AppModule`.  
А так же добавляем `QueueHandler` в массив провайдеров, допустим, в дочерний модуль для того что бы увидеть что сканируется полностью все дерево `NestContainer`. 

В `AppModule` создадим `onModuleInit` для связывания обработчиков и `bulljs`.  
Заинжектим в конструктор модуля наш `QueueExplorer`.  
В хуке `onModuleInit` запустим сканирование наших обработчиков для очереди:  
```typescript
const queueHandlers = this.queueExplorer.explore();
```

А теперь давайте связывать:  

```typescript
for (const { instance, options, name } of queueHandlers) {
    // Logger.log(`Init '${name}' consumer`, QUEUE_LOGGER_CTX);
    // создание очереди
    const queue = this.bull.queue(name, options);
    // биндинг обработчика очереди до класса.
    queue.process(instance.invoke.bind(instance));
}
```


Теперь можно смело пользоваться этой удобной фичей.  

```typescript
this.bull.queue('QueueName').add(new Somepayload('text'), optionalOptions);
```

По сути мы сделали следующие вещи:
 - отсканировали наши обработчики
 - забиндили вызов очереди на наш обработчик

Что можно добавить ? Задача сделать уже для вас.  
 - Можно добавить обработку ошибок:
  `queue.on('failed', instance.failedAction.bind(instance));`  
 - добавить проверку существования метода `invoke` в обработчике событий и выдавать `RuntimeExeception` если метод не реализован.  
 
 
### Как запускать и отлаживать приложение через Nest CLI и зачем он нужен <a id="start-with-nest-cli"></a>

В стандартном шаблоне проекта, созданном через `nest create`, скрипты запуска установлены как `nest start` с различными параметрами. 

`nest` -- это небольшая прослойка, позволяющая легко запускать Nest приложение в разных режимах, просто добавляя аргументы, например, `--watch`,  `--debug` и даже `--webpack` для сборки через Webpack.

##### Не работает отладка в WebStorm (JetBrains, Intellij IDEA) <a id="debug-in-idea"></a>

При запуске через `nest start --debug` (он же `npm run start:debug`) по кнопке кнопка Debug в WebStorm отладка не запускается (точки остановки не срабатывают).

Требуется создать отдельную задачу `Attach to Node.js` и запускать параллельно `start:debug` (`nest start --debug`).  
https://www.jetbrains.com/help/webstorm/run-debug-configuration-node-js-remote-debug.html

##### Пути в Stacktrace ошибок указывают на js файлы в dist, а не на исходные ts файлы <a id="stactrace-with-source-maps"></a>

Требуется добавить `--exec \"node -r source-map-support/register\"` к параметрам запуска, например:  
`"start:dev": "nest start --watch --exec \"node -r source-map-support/register\"",`
