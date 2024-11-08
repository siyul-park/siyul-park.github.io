---
title: "캐시 및 배치 처리로 성능 최적화하기"
date: 2024-11-08 12:00:00 +0900
categories: "pattern"
tags: ["pattern", "ddd", "system"]
---

[다단계 캐시를 통해 읽기 성능 높이기](https://siyul-park.github.io/pattern/improving-read-performance-with-multi-level-caching/)에서는 Redis와 인메모리 로컬 캐시를 활용하여 계층형 캐시를 구성하고, 트랜잭션 내 데이터 고립성을 유지하면서 데이터 접근 성능을 향상시키는 방법을 다루었습니다. [배치 처리를 통해 읽기 성능 높이기](https://siyul-park.github.io/pattern/improve-read-performance-with-batch/)에서는 연관된 쿼리를 배치 처리하여 불필요한 I/O를 줄이고, 1+N 문제를 해결함으로써 성능을 개선하는 방법을 설명했습니다.

이 글에서는 두 가지 성능 개선 방법을 Spring Boot 2, R2DBC, 그리고 MongoDB Reactive 환경에서 구현한 후, 실제 성능 개선 효과를 측정한 결과를 공유합니다.

전체 코드는 [spring-webflux-multi-module-boilerplate](https://github.com/siyul-park/spring-webflux-multi-module-boilerplate)에서 확인할 수 있습니다.

> 이 글은 2022년에 작업한 프로젝트를 기반으로 작성되었습니다. 최신 환경과 차이가 있을 수 있습니다.

## 계층형 캐시를 통한 읽기 성능 개선

먼저 여러 데이터 소스를 공통 인터페이스로 간편하게 제어하고, Kotlin의 Coroutine 환경에서 효율적으로 사용할 수 있도록 `Repository` 인터페이스를 정의합니다. 이 인터페이스는 데이터베이스와의 CRUD 연산을 비동기적으로 처리하며 기본적인 CRUD 기능을 제공합니다.

```kotlin
interface Repository<T : Any, ID : Any> {
    suspend fun create(entity: T): T
    fun createAll(entities: Flow<T>): Flow<T>
    suspend fun findById(id: ID): T?
    fun findAll(): Flow<T>
    suspend fun updateById(id: ID, patch: Patch<T>): T?
    suspend fun deleteById(id: ID)
    suspend fun count(): Long
    // ...
}
```

복잡한 조건을 바탕으로 데이터를 조회하거나 여러 데이터를 한 번에 업데이트하는 기능을 구현하기 위해 `QueryableRepository` 인터페이스를 확장합니다. 이를 통해 더 유연한 데이터 처리 작업이 가능합니다.

```kotlin
sealed class Criteria {
    // 조건 정의
}

interface QueryableRepository<T : Any, ID : Any> : Repository<T, ID> {
    suspend fun exists(criteria: Criteria): Boolean
    suspend fun findOne(criteria: Criteria): T?
    fun findAll(criteria: Criteria? = null, limit: Int? = null, offset: Long? = null, sort: Sort? = null): Flow<T>
    suspend fun update(criteria: Criteria, patch: Patch<T>): T?
    // ...
}
```

위에서 정의한 `QueryableRepository`는 R2DBC와 MongoDB에 각각 구현되어, 각 데이터 소스에 맞는 비동기 조회 및 저장 작업을 처리합니다.

```kotlin
class R2DBCQueryableRepository<T : Any, ID : Any> : QueryableRepository<T, ID> {
    // R2DBC를 이용한 구현
}

class MongoDBQueryableRepository<T : Any, ID : Any> : QueryableRepository<T, ID> {
    // MongoDB를 이용한 구현
}
```

계층형 캐시를 효과적으로 구성하기 위해 키-값 기반의 멀티 인덱스를 지원하는 캐시 저장소 인터페이스를 정의합니다.

```kotlin
interface Storage<ID : Any, T : Any> {
    suspend fun <KEY : Any> createIndex(name: String, property: WeekProperty<T, KEY>)
    suspend fun removeIndex(name: String)
    suspend fun containsIndex(name: String): Boolean

    suspend fun getIndexes(): Map<String, WeekProperty<T, *>>

    suspend fun <KEY : Any> getIfPresent(index: String, key: KEY): T?
    suspend fun <KEY : Any> getIfPresent(index: String, key: KEY, loader: suspend () -> T?): T?

    suspend fun getIfPresent(id: ID): T?
    suspend fun getIfPresent(id: ID, loader: suspend () -> T?): T?

    fun <KEY : Any> getAll(index: String, keys: Iterable<KEY>): Flow<T?>
    fun getAll(ids: Iterable<ID>): Flow<T?>

    suspend fun remove(id: ID)
    suspend fun add(entity: T)

    suspend fun entries(): Set<Pair<ID, T>>

    suspend fun clear()

    suspend fun status(): Status
}
```

이 인터페이스는 Redis와 로컬 인메모리를 사용하여 구현됩니다.

```kotlin
class InMemoryStorage<ID : Any, T : Any> : Storage<ID, T> {
    // ...
}

class RedisStorage<ID : Any, T : Any> : Storage<ID, T> {
    // ...
}
```

캐시를 여러 계층으로 관리하기 위한 `NestedStorage` 인터페이스를 정의합니다. 트랜잭션을 통해 자식 캐시와 부모 캐시를 분리하여 처리하고, 트랜잭션 종료 후 자식 캐시의 내용을 부모 캐시로 병합합니다.

```kotlin
interface NestedStorage<ID : Any, T : Any> : Storage<ID, T> {
    val parent: NestedStorage<ID, T>?

    suspend fun fork(): NestedStorage<ID, T>
    suspend fun merge(storage: NestedStorage<ID, T>)
    suspend fun clear()

    suspend fun checkout(): Map<ID, T?>
}
```

이제 `TransactionSynchronization`을 상속하여 트랜잭션이 시작되면 캐시를 분기시키고, 트랜잭션이 커밋되면 자식 캐시를 부모 캐시로 병합하여 최종적으로 일관성 있는 데이터를 유지할 수 있게 합니다.

```kotlin
class TransactionalStorageProvider<S : NestedStorage<S>> {
    suspend fun get(): S {
        try {
            val context = TransactionContextManager.currentContext().awaitSingleOrNull() ?: return root
            val currentStorage = cacheTransactionSynchronization.get(context)
            if (currentStorage != null) {
                return currentStorage
            }

            val chains = chains(context)

            var current: TransactionContext?
            var storage = root
            while (chains.isNotEmpty()) {
                current = chains.pop()
                storage = cacheTransactionSynchronization.get(current) ?: mutex.withLock {
                    cacheTransactionSynchronization.get(current) ?: run {
                        val child = storage.fork()
                        cacheTransactionSynchronization.put(current, child)
                        logger.debug("Forked Cache Storage [parent: $storage, child: $child]")
                        child
                    }
                }
            }

            return storage
        } catch (e: NoTransactionException) {
            return root
        }
    }

    // ...
}

class CacheTransactionSynchronization<S : NestedStorage<S>> : TransactionSynchronization {
    override fun afterCompletion(status: Int): Mono<Void> {
        return TransactionContextManager.currentContext().flatMap {
            if (status == STATUS_COMMITTED) {
                mono {
                    val storage = storages[it]
                    val parent = storage?.parent
                    if (storage?.parent != null) {
                        logger.debug("Merging Cache Storage [parent: $parent, child: $storage]")
                        parent?.merge(storage)
                    }
                    storages.remove(it)
                }.then()
            } else {
                mono {
                    val storage = storages[it]
                    if (storage != null) {
                        logger.debug("Removing Cache Storage $storage")
                        storage.clear()
                    }
                    storages.remove(it)
                }.then()
            }
        }
    }

    // ...
}
```

이 캐시 저장소를 사용하여 레포지토리에 캐시 기능을 추가할 수 있습니다.

```kotlin
class SimpleCachedRepository<T : Any, ID : Any>(/* ... */) : Repository<T, ID> {
    override fun findAllById(ids: Iterable<ID>): Flow<T> {
        return flow {
            val ids = ids.toList()

            val result = mutableListOf<T>()
            val notCachedIds = mutableListOf<ID>()

            storage.getAll(ids).collectIndexed { index, cached ->
                val id = ids[index]
                if (cached == null) {
                    notCachedIds.add(id)
                } else {
                    result.add(cached)
                }
            }

            if (notCachedIds.isNotEmpty()) {
                delegator.findAllById(notCachedIds)
                    .onEach { storage.add(it) }
                    .collect { result.add(it) }
            }

            emitAll(
                result.also {
                    it.sortWith { p1, p2 ->
                        val p1Id = id.get(p1)
                        val p2Id = id.get(p2)

                        ids.indexOf(p1Id) - ids.indexOf(p2Id)
                    }
                }.asFlow()
            )
        }
    }

    // ...
}
```

더 복잡한 조회를 실행할 경우, 키-값 인덱싱된 조건이 있는지 확인하고, 해당 값으로 캐시에서 객체를 우선 탐색합니다.

```kotlin
class CachedQueryableRepository<T : Any, ID : Any>(/* ... */) : Repository<T, ID> {
    override fun findAll(criteria: Criteria?, limit: Int?, offset: Long?, sort: Sort?): Flow<T> {
        if (limit != null && limit <= 0) {
            return emptyFlow()
        }

        return flow {
            if (criteria != null && (offset == null || offset == 0L)) {
                val indexNameAndValue = getUniqueIndexNameAndValue(criteria)
                if (indexNameAndValue != null) {
                    val (indexName, value) = indexNameAndValue
                    storage.getIfPresent(indexName, value) { delegator.findOne(criteria) }
                        ?.let { emit(it) }
                    return@flow
                }
            }

            if (criteria != null && limit == null && offset == null && sort == null) {
                if (criteria is Criteria.In) {
                    val key = criteria.key
                    val value = criteria.value

                    if (storage.containsIndex(key)) {
                        val result = mutableListOf<T>()
                        val notCachedKey = mutableListOf<Any?>()

                        storage.getAll(key, value.map { ArrayList<Any?>().apply { add(it) } }).collectIndexed { index, cached ->
                            val current = value[index]
                            if (cached == null) {
                                notCachedKey.add(current)
                            } else {
                                result.add(cached)
                            }
                        }

                        if (notCachedKey.isNotEmpty()) {
                            delegator.findAll(where(key).`in`(notCachedKey))
                                .onEach { storage.add(it) }
                                .collect { result.add(it) }
                        }

                        return@flow emitAll(result.asFlow())
                    }
                }
            }

            return@flow emitAll(
                delegator.findAll(criteria, limit, offset, sort)
                    .onEach { storage.add(it) }
            )
        }
    }

    // ...
}
```

## 배치 처리

배치 처리는 동일한 트랜잭션 내에서 관련된 여러 쿼리를 효율적으로 처리하기 위해, 실행할 쿼리를 미리 수집하는 과정이 필요합니다. 이때, 수집된 여러 쿼리를 관리하는 객체를 **컨텍스트**라고 합니다.

각 레포지토리는 독립적인 컨텍스트를 가지며, 동일한 레포지토리에서 실행되는 쿼리들은 동일한 컨텍스트 내에서 묶입니다.

```kotlin
class FetchContext {
    private val contexts = MultiKeyMap<Any, AggregateContext<*>>()

    fun <T : Any> get(repository: QueryableRepository<T, *>, clazz: KClass<T>): AggregateContext<T> {
        @Suppress("UNCHECKED_CAST")
        return contexts.getOrPut(MultiKey(repository, clazz)) {
            AggregateContext(repository, clazz)
        } as AggregateContext<T>
    }
}

class AggregateContext<T : Any>(
    repository: QueryableRepository<T, *>,
    clazz: KClass<T>,
) {
    private val queryAggregator = QueryAggregator(repository, clazz)

    suspend fun clear() {
        queryAggregator.clear()
    }

    suspend fun clear(entity: T) {
        queryAggregator.clear(entity)
    }

    fun join(criteria: Criteria?, limit: Int? = null): QueryFetcher<T> {
        return QueryFetcher(SelectQuery(criteria, limit), queryAggregator)
    }
}
```

`join` 메서드를 통해 `QueryAggregator`에 실행할 쿼리를 등록하고, 실제 쿼리 실행을 위임하게 됩니다.

```kotlin
class QueryFetcher<T : Any>(
    private val query: SelectQuery,
    private val queryAggregator: QueryAggregator<T>
) {
    init {
        queryAggregator.link(query)
    }

    suspend fun clear() {
        queryAggregator.clear(query)
    }

    suspend fun fetchOne(): T? {
        return fetch().toList().firstOrNull()
    }

    fun fetch(): Flow<T> {
        return queryAggregator.fetch(query)
    }
}
```

`QueryAggregator`는 수집된 쿼리들을 하나로 묶어 배치 처리하고, 그 결과를 저장합니다. 이후, 결과를 개별 쿼리별로 적합하게 분배하여 일관성을 유지합니다.

```kotlin
class QueryAggregator<T : Any>(/* ... */) {
    // ...

    fun link(query: SelectQuery) {
        links.push(query)
    }

    fun fetch(query: SelectQuery): Flow<T> {
        return flow {
            mutex.withLock {
                pop(query)?.onEach { emit(it) } ?: run {
                    val merged = mutableSetOf<SelectQuery>().also {
                        it.addAll(free())
                        it.add(query)
                    }

                    val limit = merged.map { it.limit }.fold(0 as Int?) { acc, cur ->
                        if (cur == null || acc == null) {
                            null
                        } else {
                            acc + cur
                        }
                    }

                    val result = repository.findAll(merge(merged.mapNotNull { it.where }.toSet()), limit = limit).toList()
                    val distributed = distribute(merged, result)

                    distributed.forEach { (key, value) ->
                        if (key != query) {
                            store.put(key, value)
                        } else {
                            latest[query] = value
                            value.onEach { emit(it) }
                        }
                    }
                }
            }
        }
    }

    private suspend fun pop(query: SelectQuery): Collection<T>? {
        return store.getIfPresent(query)?.let {
            store.remove(query)
            latest[query] = it
            it
        }
    }

    private suspend fun free(): Set<SelectQuery> {
        val curr = links.entries().toMutableSet()
        val used = store.entries().map { it.first }

        used.forEach {
            curr.remove(it)
        }

        return curr
    }

    private fun merge(criteria: Set<Criteria>): Criteria {
        return asInOperator(criteria) ?: criteria.reduce { acc, cur -> cur.or(acc) }
    }

    private fun asInOperator(criteria: Set<Criteria>): Criteria? {
        if (criteria.all { it is Criteria.In || it is Criteria.Equals }) {
            val keys = criteria.mapNotNull {
                when (it) {
                    is Criteria.In -> it.key
                    is Criteria.Equals -> it.key
                    else -> null
                }
            }.toSet()
            if (keys.size == 1) {
                return Criteria.In(
                    keys.first(),
                    criteria.map {
                        when (it) {
                            is Criteria.In -> it.value
                            is Criteria.Equals -> listOf(it.value)
                            else -> emptyList()
                        }
                    }.fold<List<Any?>, MutableList<Any?>>(mutableListOf()) { acc, cur ->
                        acc.also { it.addAll(cur) }
                    }.toSet().toList()
                )
            }
        }

        return null
    }

    private fun distribute(criteria: Set<SelectQuery>, values: List<T>): Map<SelectQuery, List<T>> {
        val result = mutableMapOf<SelectQuery, MutableList<T>>()

        criteria.forEach { q ->
            val filter = q.where?.let { parser.parse(it) } ?: { false }
            values.forEach { value ->
                if (filter(value)) {
                    result.getOrPut(q) { mutableListOf() }.add(value)
                }
            }
        }

        return result.mapValues { (key, value) ->
            if (key.limit != null && value.size > key.limit) {
                value.subList(0, key.limit)
            } else {
                value
            }
        }
    }
}
```

## 성능 측정

본 성능 측정은 200명의 가상 사용자가 동시에 시스템을 사용하는 상황을 가정하여, 2019년 MacBook Pro 16인치 i9 16GB 환경에서 모든 의존성 패키지를 단일 머신에서 실행한 결과입니다.

테스트는 제한된 시간 내에 순차적으로 진행되었으며, 이로 인해 스로틀링 현상이 발생할 수 있어 일부 결과에는 노이즈가 포함될 가능성이 있습니다.

계층형 캐시를 적용한 결과, `GET /clients/{client_id}`의 TPS는 236에서 1,968로 734% 향상되었으며, 배치 처리를 통해 `GET /users`의 TPS는 96에서 125로 23% 개선되었습니다.

### 디폴트

| Method | Endpoint                   | TPS    | Avgage | p(50)  | p(90)  | p(95)  |
|--------|-----------------------------|--------|--------|--------|--------|--------|
| GET    | /self                       | 556.29 | 329.36 ms | 212.54 ms | 762.31 ms | 971.68 ms |
| GET    | /clients                    | 13.48  | 7.72 s     | 7.96 s   | 10.44 s   | 10.81 s   |
| GET    | /clients/{client_id}        | 291.94 | 607.88 ms  | 482.49 ms | 1.27 s    | 1.54 s    |
| PATCH  | /clients/{client_id}        | 160.18 | 601.36 ms  | 549.34 ms | 948.27 ms | 1.27 s    |
| DELETE | /clients/{client_id}        | 52.45  | 738.06 ms  | 650.17 ms | 1.38 s    | 2.09 s    |
| GET    | /users                      | 28.31  | 5.08 s     | 4.89 s    | 8.19 s    | 8.83 s    |
| GET    | /users/{user-id}            | 156.74 | 504.32 ms  | 342.5 ms  | 1.12 s    | 1.36 s    |
| PATCH  | /users/{user-id}            | 171.00 | 517.62 ms  | 425.57 ms | 1.07 s    | 1.27 s    |
| DELETE | /users/{user-id}            | 87.54  | 563.38 ms  | 335.46 ms | 1.34 s    | 1.79 s    |

### 계층형 캐시

| Method | Endpoint                   | TPS     | Avgage   | p(50)   | p(90)   | p(95)   |
|--------|-----------------------------|---------|----------|---------|---------|---------|
| GET    | /self                       | 1013.03 | 77.16 ms | 34.48 ms | 200.63 ms | 294.13 ms |
| GET    | /clients                    | 31.54   | 4.51 s   | 4.93 s   | 9.56 s    | 10.25 s   |
| GET    | /clients/{client_id}        | 1622.50 | 107.75 ms| 77.74 ms | 146.7 ms  | 214.95 ms |
| PATCH  | /clients/{client_id}        | 441.33  | 215.88 ms| 197.54 ms| 272.29 ms | 315.04 ms |
| DELETE | /clients/{client_id}        | 229.36  | 251.92 ms| 210.19 ms| 396.56 ms | 445.29 ms |
| GET    | /users                      | 96.42   | 1.04 s   | 496.93 ms| 3.01 s    | 3.3 s     |
| GET    | /users/{user-id}            | 621.05  | 92.27 ms | 81.3 ms  | 108.85 ms | 121.59 ms |
| PATCH  | /users/{user-id}            | 419.67  | 230.3 ms | 200.59 ms| 298.74 ms | 361.97 ms |
| DELETE | /users/{user-id}            | 258.74  | 251.67 ms| 230.25 ms| 365.32 ms | 416.41 ms |

### 배치 처리

| Method | Endpoint                   | TPS     | Avgage   | p(50)   | p(90)   | p(95)   |
|--------|-----------------------------|---------|----------|---------|---------|---------|
| GET    | /self                       | 465.04  | 389.79 ms| 237.6 ms | 956.59 ms | 1.2 s   |
| GET    | /clients                    | 16.90   | 6.8 s    | 6.83 s   | 9.97 s    | 10.44 s |
| GET    | /clients/{client_id}        | 236.20  | 720.63 ms| 574.57 ms| 1.55 s    | 1.89 s  |
| PATCH  | /clients/{client_id}        | 165.78  | 580.64 ms| 465.01 ms| 1.22 s    | 1.48 s  |
| DELETE | /clients/{client_id}        | 42.28   | 854.3 ms | 823.76 ms| 1.29 s    | 1.74 s  |
| GET    | /users                      | 32.94   | 4.46 s   | 4.31 s   | 6.69 s    | 7.36 s  |
| GET    | /users/{user-id}            | 171.88  | 466.96 ms| 338.9 ms | 1.03 s    | 1.23 s  |
| PATCH  | /users/{user-id}            | 155.01  | 561.52 ms| 533.1 ms | 705.94 ms | 952.72 ms|
| DELETE | /users/{user-id}            | 89.88   | 627.73 ms| 401.1 ms | 1.44 s    | 1.92 s  |

### 계층형 캐시와 배치 처리

| Method | Endpoint                   | TPS     | Avgage   | p(50)   | p(90)   | p(95)   |
|--------|-----------------------------|---------|----------|---------|---------|---------|
| GET    | /self                        | 1041.11 | 85.56 ms | 36.04 ms| 225.25 ms| 362.71 ms|
| GET    | /clients                     | 83.47   | 1.34 s   | 1.01 s  | 3.02 s   | 4.07 s  |
| GET    | /clients/{client_id}         | 1968.06 | 85.39 ms | 82.09 ms| 129.72 ms| 148.91 ms|
| PATCH  | /clients/{client_id}         | 245.22  | 403.8 ms | 333.37 ms| 646.13 ms| 775.07 ms|
| DELETE | /clients/{client_id}         | 250.99  | 238.43 ms| 234.29 ms| 331.15 ms| 371.51 ms|
| GET    | /users                       | 125.94  | 760.71 ms| 663.44 ms| 1.28 s   | 1.41 s  |
| GET    | /users/{user-id}             | 526.45  | 115.62 ms| 76.62 ms| 143.79 ms| 424.72 ms|
| PATCH  | /users/{user-id}             | 263.72  | 369.67 ms| 300.79 ms| 527.33 ms| 1.16 s  |
| DELETE | /users/{user-id}             | 174.78  | 559.84 ms| 430.17 ms| 674.69 ms| 1.31 s  |
