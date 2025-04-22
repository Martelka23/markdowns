# TypeORM миграции при нескольких репликах
## Обычный запуск двух реплик
```ts
async function runMigrationsWithoutLock() {
	await dataSource.initialize();
	
	console.log(`[${process.pid}] Запуск миграций...`);
	await dataSource.runMigrations();
	console.log(`[${process.pid}] Миграции выполнены.`);
	
	await dataSource.destroy();
}
```

### Реплика 1️⃣
```bash
[1] stdout:

> nest-app@0.0.1 migration:run
> ts-node ./src/migration-runner.service.ts

[25484] Запуск миграций...
[25484] Миграции выполнены.
```

### Реплика 2️⃣
```bash
[2] error: Command failed: npm run migration:run
D:\Work\rolf\nest-app\node_modules\typeorm\src\driver\postgres\PostgresQueryRunner.ts:328
            throw new QueryFailedError(query, parameters, err)
                  ^
QueryFailedError: duplicate key value violates unique constraint "pg_class_relname_nsp_index"
    at PostgresQueryRunner.query (D:\Work\rolf\nest-app\node_modules\typeorm\src\driver\postgres\PostgresQueryRunner.ts:328:19)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at async Init1700000000001.up (D:\Work\rolf\nest-app\migration\1700000000001-Init.ts:11:5)
    at async MigrationExecutor.executePendingMigrations (D:\Work\rolf\nest-app\node_modules\src\migration\MigrationExecutor.ts:336:17)    
    at async DataSource.runMigrations (D:\Work\rolf\nest-app\node_modules\src\data-source\DataSource.ts:403:13)
    at async runMigrationsWithoutLock (D:\Work\rolf\nest-app\src\migration-runner.service.ts:6:2) {
  query: 'CREATE TABLE IF NOT EXISTS test (id SERIAL PRIMARY KEY, created_at TIMESTAMP DEFAULT NOW())'
}
```
>Текст ошибки сокращен для экономии места

## 🔒 Advisory lock
**Advisory locks** (рекомендательные блокировки) - средства для создания блокировок, которые позволяют приложениям синхронизировать доступ к базе данных по произвольным числовым ключам.

Рассмотрим два основных вида:
- **pg_advisory_lock(lock_key)** - блокирующий вызов
- **pg_try_advisory_lock(lock_key)** - неблокирующий вызов

> **Note:** Есть еще **Shared** блокировки, но они не важны в текущей проблематике. Если интересно, можно подробнее ознакомиться с [pg_advisory_lock_shared](https://pgpedia.info/p/pg_advisory_lock_shared.html) и [pg_try_advisory_lock_shared](https://pgpedia.info/p/pg_try_advisory_lock_shared.html).

### 🆚 Ключевые отличия
|                |pg_advisory_lock|pg_try_advisory_lock|
|----------------|-------------------------------|-----------------------------|
|Блокирование выполнения кода | ✅ Да | ❌ Нет|
|Возвращаемый результат | ❌ void | ✅ `true` / `false`|
|Применение | Синхронные задачи / Очереди | Асинхронные задачи / Проверки|

### ✈️ В контексте миграций:
При запуске миграций из нескольких реплик приложения:
    
-   `pg_advisory_lock` — ❌ **небезопасен**: все реплики будут стоять в очереди, и каждый по очереди выполнит миграции, что навредит производительности и может привести к проблемам.

-   `pg_try_advisory_lock` — ✅ **подходит**: только одна реплика займет блокировку, остальные просто пропустят выполнение миграций.


## 🔐 Пример "Advisory lock"
```ts
async function runMigrationsWithLock() {
	await dataSource.initialize();
	
	console.log(`[${process.pid}] Ожидаю освобождение блокировки...`);
	
	const lockKey = 12345678;
	await dataSource.query(`SELECT pg_advisory_lock(${lockKey})`);

	console.log(`[${process.pid}] Блокировка успешно занята. Запуск миграций...`);

	await dataSource.runMigrations();
	await dataSource.query(`SELECT pg_advisory_unlock(${lockKey})`);
	
	console.log(`[${process.pid}] Миграции выполнены. Блокировка освобождена.`);
	
	await dataSource.destroy();
}
```

### Реплика 1️⃣
```bash
[1] stdout:

> nest-app@0.0.1 migration:run
> ts-node ./src/migration-runner.service.ts

[42568] Ожидаю освобождение блокировки...
[42568] Блокировка успешно занята. Запуск миграций...
[42568] Миграции выполнены. Блокировка освобождена.
```

### Реплика 2️⃣
```bash
[2] stdout:

> nest-app@0.0.1 migration:run
> ts-node ./src/migration-runner.service.ts

[2932] Ожидаю освобождение блокировки...
[2932] Блокировка успешно занята. Запуск миграций...
[2932] Миграции выполнены. Блокировка освобождена.
```

## 🔒 Пример "Try advisory lock"
```ts
async function runMigrationsWithTryLock() {
	await dataSource.initialize();
	console.log(`[${process.pid}] Пробую занять блокировку...`);
	const lockKey = 12345678;
	
	const result = await dataSource.query(`SELECT pg_try_advisory_lock(${lockKey})`);
	const acquired = result[0]?.pg_try_advisory_lock;
	
	if (acquired) {
		console.log(`[${process.pid}] Блокировка успешно занята. Запуск миграций...`);
		
		await dataSource.runMigrations();
		await dataSource.query(`SELECT pg_advisory_unlock(${lockKey})`);
		
		console.log(`[${process.pid}] Миграции выполнены. Блокировка освобождена.`);
	} else {
		console.log(`[${process.pid}] Блокировка уже занята. Миграции не запущены.`);
	}
	
	await dataSource.destroy();
}
```

### Реплика 1️⃣
```bash
[1] stdout:

> nest-app@0.0.1 migration:run
> ts-node ./src/migration-runner.service.ts

[42840] Пробую занять блокировку...
[42840] Блокировка успешно занята. Запуск миграций...
[42840] Миграции выполнены. Блокировка освобождена.
```

### Реплика 2️⃣
```bash
[2] stdout:
 
> nest-app@0.0.1 migration:run
> ts-node ./src/migration-runner.service.ts

[30072] Пробую занять блокировку...
[30072] Блокировка уже занята. Миграции не запущены.
```

## ⚠️ На что обратить внимание
Обязательно нужно освободить блокировку после выполнения миграций
```ts
await dataSource.query(`SELECT pg_advisory_unlock(${lockKey})`);
```
