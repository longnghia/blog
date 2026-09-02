---
date: "2026-03-09T23:58:53+07:00"
draft: false
title: "Paging 3 with Room and Compose: Three Patterns"
summary: "Which of PagingSource, RemoteMediator, or plain Room paging you actually need — and the code for each."
categories:
  - Code
tags:
  - android
  - compose
  - paging3
  - room
  - kotlin
---

{{< gpt >}}

Paging 3 gets confusing because most tutorials show one setup without saying which problem it solves. There are really three, and picking the wrong one means either writing a `RemoteMediator` you didn't need or fighting Paging to do offline caching it wasn't configured for.

| Your situation                                    | What you need                   |
| ------------------------------------------------- | ------------------------------- |
| Paging an API, no local cache                     | `PagingSource`                  |
| Paging an API, cached in Room, works offline      | `RemoteMediator` + Room         |
| Paging data already in Room                       | Room's generated `PagingSource` |

The last one needs the least code and is the one people most often over-engineer.

## Dependencies

```gradle
implementation "androidx.paging:paging-runtime:3.2.1"
implementation "androidx.paging:paging-compose:3.2.1"

// for the Room-backed patterns
implementation "androidx.room:room-runtime:2.6.1"
implementation "androidx.room:room-ktx:2.6.1"
```

## Pattern 1: paging an API directly

A `PagingSource` answers three questions: how to load a page, what key identifies the next and previous pages, and what to do on error.

```kotlin
class UserPagingSource(
    private val api: UserApi
) : PagingSource<Int, User>() {

    override suspend fun load(
        params: LoadParams<Int>
    ): LoadResult<Int, User> {

        val page = params.key ?: 1

        return try {
            val response = api.getUsers(page)

            LoadResult.Page(
                data = response.users,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.users.isEmpty()) null else page + 1
            )

        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(
        state: PagingState<Int, User>
    ): Int? {
        return state.anchorPosition?.let { position ->
            state.closestPageToPosition(position)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(position)?.nextKey?.minus(1)
        }
    }
}
```

A `null` key means "no more pages in that direction", which is how Paging knows to stop.

`getRefreshKey` is what makes pull-to-refresh land the user roughly where they were instead of at the top. It reads `anchorPosition` — the item currently on screen — and derives a key near it.

`Pager` turns the source into a `Flow`:

```kotlin
class UserRepository(private val api: UserApi) {

    fun getUsers() = Pager(
        config = PagingConfig(pageSize = 20),
        pagingSourceFactory = { UserPagingSource(api) }
    ).flow
}
```

`cachedIn(viewModelScope)` in the ViewModel is not optional — without it the flow restarts on every configuration change and refetches page 1:

```kotlin
class UserViewModel(
    private val repo: UserRepository
) : ViewModel() {

    val users = repo.getUsers()
        .cachedIn(viewModelScope)
}
```

## Pattern 2: API into Room, with `RemoteMediator`

When you want offline access and paging that survives process death, the list is served from Room and `RemoteMediator` fills Room from the API as the user scrolls. Room provides the `PagingSource`; the mediator only handles syncing.

```text
Retrofit API → RemoteMediator → Room → PagingSource → Pager → ViewModel → LazyColumn
```

The entity and DAO are ordinary Room, except that the DAO returns a `PagingSource` — Room generates the implementation:

```kotlin
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: Int,
    val name: String
)

@Dao
interface UserDao {

    @Query("SELECT * FROM users ORDER BY id")
    fun pagingSource(): PagingSource<Int, UserEntity>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(users: List<UserEntity>)

    @Query("DELETE FROM users")
    suspend fun clearAll()
}
```

The mediator is the substance of this pattern:

```kotlin
@OptIn(ExperimentalPagingApi::class)
class UserRemoteMediator(
    private val api: UserApi,
    private val db: AppDatabase
) : RemoteMediator<Int, UserEntity>() {

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, UserEntity>
    ): MediatorResult {

        val page = when (loadType) {

            LoadType.REFRESH -> 1

            LoadType.PREPEND -> {
                return MediatorResult.Success(endOfPaginationReached = true)
            }

            LoadType.APPEND -> {
                val lastItem = state.lastItemOrNull()
                if (lastItem == null) 1
                else (lastItem.id / state.config.pageSize) + 1
            }
        }

        return try {

            val response = api.getUsers(page)

            db.withTransaction {

                if (loadType == LoadType.REFRESH) {
                    db.userDao().clearAll()
                }

                val entities = response.users.map {
                    UserEntity(it.id, it.name)
                }

                db.userDao().insertAll(entities)
            }

            MediatorResult.Success(
                endOfPaginationReached = response.users.isEmpty()
            )

        } catch (e: Exception) {
            MediatorResult.Error(e)
        }
    }
}
```

Points worth pausing on:

- **`PREPEND` returns `endOfPaginationReached = true`.** Page 1 is the top of the list; there is nothing before it. Returning anything else makes Paging keep asking.
- **`REFRESH` clears the table inside the same transaction as the insert.** Deleting outside the transaction leaves the UI briefly showing an empty list if the network call then fails.
- **Deriving the page from `lastItem.id`** works when IDs are dense and sequential. If they aren't — and in most real APIs they aren't — you need a `RemoteKeys` table storing the next key per item instead. This is the most common reason a `RemoteMediator` silently stops loading.

The `Pager` takes both, and the `pagingSourceFactory` is now the DAO:

```kotlin
@OptIn(ExperimentalPagingApi::class)
class UserRepository(
    private val db: AppDatabase,
    private val api: UserApi
) {

    fun getUsers() = Pager(
        config = PagingConfig(pageSize = 20),
        remoteMediator = UserRemoteMediator(api, db),
        pagingSourceFactory = { db.userDao().pagingSource() }
    ).flow
}
```

## Pattern 3: paging Room only

If the data is already local, you don't need a `RemoteMediator` and you don't need to write a `PagingSource`. Room generates one from a query.

A common case: an `items` table, a `favorites` table, and a list that shows items with their favorite status. The instinct is to query both and combine in Kotlin. Don't — do it in SQL and let paging happen inside SQLite.

```kotlin
@Entity(tableName = "items")
data class ItemEntity(
    @PrimaryKey val id: Long,
    val title: String
)

@Entity(
    tableName = "favorites",
    indices = [Index("itemId")],
    primaryKeys = ["itemId"]
)
data class FavoriteEntity(
    val itemId: Long
)
```

The projection is a plain data class, not an entity:

```kotlin
data class ItemWithFavorite(
    val id: Long,
    val title: String,
    val isFavorite: Boolean
)
```

```kotlin
@Dao
interface ItemDao {

    @Query("""
        SELECT items.id,
               items.title,
               CASE WHEN favorites.itemId IS NOT NULL
                    THEN 1 ELSE 0 END AS isFavorite
        FROM items
        LEFT JOIN favorites
        ON items.id = favorites.itemId
        ORDER BY items.id
    """)
    fun pagingItems(): PagingSource<Int, ItemWithFavorite>
}

@Dao
interface FavoriteDao {

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun addFavorite(favorite: FavoriteEntity)

    @Query("DELETE FROM favorites WHERE itemId = :id")
    suspend fun removeFavorite(id: Long)
}
```

```kotlin
class ItemRepository(private val db: AppDatabase) {

    fun getItems() = Pager(
        config = PagingConfig(pageSize = 20),
        pagingSourceFactory = { db.itemDao().pagingItems() }
    ).flow
}
```

No API, no mediator, and Paging only ever materializes one page at a time.

### Favourite toggles update the list for free

Room invalidates the `PagingSource` whenever a table the query touches changes. That includes `favorites`, because it's in the `LEFT JOIN`:

```text
favorites INSERT/DELETE → Room invalidates → Paging reloads → Compose recomposes
```

So `addFavorite()` is the whole implementation of the toggle. No manual refresh, no state to keep in sync.

### `EXISTS` instead of `LEFT JOIN`

On large tables this usually plans better — it can stop at the first match rather than joining every row:

```sql
SELECT
  i.id,
  i.title,
  EXISTS(
      SELECT 1 FROM favorites f
      WHERE f.itemId = i.id
  ) AS isFavorite
FROM items i
ORDER BY i.id
```

The `Index("itemId")` on `FavoriteEntity` above matters for both versions. Without it, every row scans the favorites table.

For very large or very hot lists, some apps go further and maintain a database `VIEW` — or a materialized `item_with_favorite` table — so the DAO query stays trivial.

## The Compose side

Identical for all three patterns, which is the nice part — swapping the repository implementation doesn't touch the UI.

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {

    val users = viewModel.users.collectAsLazyPagingItems()

    LazyColumn {
        items(users.itemCount) { index ->
            users[index]?.let { user ->
                Text(text = user.name)
            }
        }

        when (users.loadState.append) {
            is LoadState.Loading -> {
                item { CircularProgressIndicator() }
            }
            else -> {}
        }
    }
}
```

Indexing into `users[index]` can return `null` — that's a placeholder for a row Paging hasn't loaded yet, not an error.

The two load states you care about are separate:

```kotlin
when (users.loadState.refresh) {
    is LoadState.Loading -> CircularProgressIndicator()
    is LoadState.Error -> Text("Error loading users")
    else -> {}
}
```

`loadState.refresh` is the initial or pulled-to-refresh load and belongs *outside* the list — it's a full-screen spinner. `loadState.append` is loading the next page and belongs as the last item *inside* it.

## Summary

| Component         | Role                                  |
| ----------------- | ------------------------------------- |
| `PagingSource`    | Defines how a page is loaded          |
| `RemoteMediator`  | Syncs a remote source into a local one |
| `Pager`           | Creates the paging stream             |
| `PagingData`      | The paginated data container          |
| `LazyPagingItems` | Compose adapter                       |
| `LazyColumn`      | The UI list                           |

Start with pattern 3 if the data is local and pattern 1 if it isn't. Reach for `RemoteMediator` only when you specifically need offline reads of remote data — it's the pattern with the most places to get something subtly wrong.
