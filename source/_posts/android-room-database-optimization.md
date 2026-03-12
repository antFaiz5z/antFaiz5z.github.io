---
title: "Room 数据库迁移与性能优化"
date: 2023-06-01 22:20:44
description: "从基础到生产环境实践 Room 基础回顾"
comments: true
toc: true
categories:
  - Android
  - Database
tags:
  - Android
  - Room
  - Database
  - Performance
---

# Room 数据库迁移与性能优化

> 从基础到生产环境实践

## 目录

1. [Room 基础回顾](#1-room-基础回顾)
2. [数据库迁移策略](#2-数据库迁移策略)
3. [性能优化技巧](#3-性能优化技巧)
4. [常见问题解决](#4-常见问题解决)
5. [最佳实践总结](#5-最佳实践总结)

---

## 1. Room 基础回顾

### 核心组件

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Room 架构                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐        │
│    │    DAO      │────▶│  Database   │────▶│   Entity    │        │
│    └─────────────┘     └─────────────┘     └─────────────┘        │
│         │                   │                                      │
│         │              ┌─────────────┐                              │
│         └─────────────▶│  Room       │                              │
│                        │  Runtime    │                              │
│                        └─────────────┘                              │
│                                                                      │
│    Entity: 数据模型 (表)                                             │
│    DAO: 数据访问对象 (CRUD)                                          │
│    Database: 数据库持有者                                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 基本实现

```kotlin
// Entity
@Entity(tableName = "users")
data class User(
    @PrimaryKey val id: Long,
    val name: String,
    val email: String,
    val createdAt: Long = System.currentTimeMillis()
)

// DAO
@Dao
interface UserDao {
    
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
    
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUserById(id: Long): User?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: User)
    
    @Delete
    suspend fun deleteUser(user: User)
    
    @Query("DELETE FROM users")
    suspend fun deleteAllUsers()
}

// Database
@Database(
    entities = [User::class],
    version = 1,
    exportSchema = true
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

---

## 2. 数据库迁移策略

### 简单版本升级

```kotlin
@Database(
    entities = [User::class, Post::class],
    version = 2,  // 升级版本号
    exportSchema = true
)
abstract class AppDatabase : RoomDatabase() {
    
    companion object {
        private const val DATABASE_NAME = "my_database"
    }
    
    abstract fun userDao(): UserDao
    
    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    DATABASE_NAME
                )
                    // ✅ 添加迁移
                    .addMigrations(MIGRATION_1_2)
                    .build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

### 迁移示例

```kotlin
// 添加新表
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("""
            CREATE TABLE IF NOT EXISTS posts (
                id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
                title TEXT NOT NULL,
                content TEXT NOT NULL,
                userId INTEGER NOT NULL,
                createdAt INTEGER NOT NULL,
                FOREIGN KEY(userId) REFERENCES users(id) ON DELETE CASCADE
            )
        """.trimIndent())
    }
}

// 添加新列
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("""
            ALTER TABLE users ADD COLUMN phone TEXT
        """.trimIndent())
    }
}

// 重命名表
val MIGRATION_3_4 = object : Migration(3, 4) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users RENAME TO users_new")
        database.execSQL("""
            CREATE TABLE users (
                id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
                name TEXT NOT NULL,
                email TEXT NOT NULL,
                phone TEXT,
                createdAt INTEGER NOT NULL
            )
        """.trimIndent())
        database.execSQL("INSERT INTO users SELECT * FROM users_new")
        database.execSQL("DROP TABLE users_new")
    }
}
```

### 自动迁移 (Room 2.4+)

```kotlin
@Database(
    entities = [User::class],
    version = 2,
    exportSchema = true
)
@AutoMigration(from = 1, to = 2)
abstract class AppDatabase : RoomDatabase()
```

---

## 3. 性能优化技巧

### 1) 避免主线程查询

```kotlin
class Repository(private val userDao: UserDao) {
    
    // ✅ 协程: 自动切换到 IO 线程
    fun getUsers(): Flow<List<User>> = userDao.getAllUsers()
    
    // ❌ 错误: 在主线程同步查询
    fun getUsersSync(): List<User> {
        return userDao.getAllUsers()  // 会阻塞主线程！
    }
}
```

### 2) 使用索引

```kotlin
@Entity(
    tableName = "users",
    indices = [
        // ✅ 添加索引加速查询
        Index(value = ["email"], unique = true),
        Index(value = ["createdAt"])
    ]
)
data class User(
    @PrimaryKey val id: Long,
    val name: String,
    val email: String,
    val createdAt: Long
)
```

### 3) 只查询需要的字段

```kotlin
@Dao
interface UserDao {
    
    // ✅ 推荐: 只查询需要的字段
    @Query("SELECT id, name FROM users")
    fun getAllUserNames(): Flow<List<UserName>>
    
    // ❌ 不好: 查询所有字段
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
    
    // ✅ 分页查询
    @Query("SELECT * FROM users LIMIT :limit OFFSET :offset")
    suspend fun getUsersPaginated(limit: Int, offset: Int): List<User>
}

// 只包含需要的字段
data class UserName(val id: Long, val name: String)
```

### 4) 使用 Paging 库

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY createdAt DESC")
    fun getUsersPaged(): PagingSource<Int, User>
}

class Repository(private val userDao: UserDao) {
    fun getUsersPaged(): Flow<PagingData<User>> {
        return Pager(
            config = PagingConfig(
                pageSize = 20,
                enablePlaceholders = false,
                prefetchDistance = 5
            ),
            pagingSourceFactory = { userDao.getUsersPaged() }
        ).flow
    }
}

// Compose 中使用
@Composable
fun UserList(viewModel: UserViewModel = hiltViewModel()) {
    val pager = viewModel.users.collectAsLazyPagingItems()
    
    LazyColumn {
        items(pager) { user ->
            UserItem(user = user!!)
        }
    }
}
```

### 5) Room + Flow 协程优化

```kotlin
@Dao
interface UserDao {
    
    // ✅ 使用 Flow 替代 LiveData
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
    
    // ✅ Flow 带 distinctUntilChanged 避免重复
    @Query("SELECT * FROM users WHERE id = :id")
    fun getUserById(id: Long): Flow<User?>
}

// 使用 map 转换
fun getUserNames(): Flow<List<String>> {
    return userDao.getAllUsers()
        .map { users -> users.map { it.name } }  // 转换
        .distinctUntilChanged()  // 避免重复
}
```

---

## 4. 常见问题解决

### 问题 1: Entity 与表字段不匹配

```kotlin
// Entity 中使用不同的列名
@Entity(tableName = "users")
data class User(
    @PrimaryKey val id: Long,
    
    // ✅ 使用 ColumnInfo 指定列名
    @ColumnInfo(name = "user_name")
    val name: String,
    
    @ColumnInfo(name = "user_email")
    val email: String
)
```

### 问题 2: 关系型数据查询

```kotlin
// 定义关系
data class UserWithPosts(
    @Embedded val user: User,
    @Relation(
        parentColumn = "id",
        entityColumn = "userId"
    )
    val posts: List<Post>
)

@Dao
interface UserDao {
    
    // ✅ 查询用户及其所有帖子
    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    fun getUserWithPosts(userId: Long): Flow<UserWithPosts>
    
    // 或者一次性获取
    @Transaction
    @Query("SELECT * FROM users")
    fun getAllUsersWithPosts(): Flow<List<UserWithPosts>>
}
```

### 问题 3: 多数据库实例

```kotlin
// ✅ 正确: 使用单例模式
@Database(...)
abstract class AppDatabase : RoomDatabase() {
    
    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(...)
                    .build()
                INSTANCE = instance
                instance
            }
        }
    }
}

// ❌ 错误: 每次都创建新实例
fun badGetDatabase(context: Context): AppDatabase {
    return Room.databaseBuilder(...)
        .build()  // 每次创建新实例！
}
```

---

## 5. 最佳实践总结

### Database 封装

```kotlin
class DatabaseManager private constructor(
    context: Context
) {
    private val database = Room.databaseBuilder(
        context.applicationContext,
        AppDatabase::class.java,
        "app_database"
    )
        .addMigrations(*ALL_MIGRATIONS)
        .fallbackToDestructiveMigration()  // ⚠️ 仅开发时使用
        .build()
    
    val userDao: UserDao get() = database.userDao()
    val postDao: PostDao get() = database.postDao()
    
    companion object {
        @Volatile
        private var INSTANCE: DatabaseManager? = null
        
        fun getInstance(context: Context): DatabaseManager {
            return INSTANCE ?: synchronized(this) {
                val instance = DatabaseManager(context)
                INSTANCE = instance
                instance
            }
        }
    }
}

// 使用 Hilt 依赖注入
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    fun provideDatabase(context: Context): AppDatabase {
        return Room.databaseBuilder(...)
            .build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
}
```

### Flow vs LiveData 选择

| 场景 | 推荐 |
|------|------|
| Compose 中使用 | Flow + collectAsStateWithLifecycle |
| 需要在 Activity/Fragment 中观察 | LiveData |
| 需要跨进程通信 | LiveData |
| 纯 Kotlin 项目 | Flow |
| 与协程集成 | Flow |

---

## 性能优化检查清单

| 优化项 | 状态 |
|--------|------|
| 添加索引 | ✅ |
| 只查询需要的字段 | ✅ |
| 使用 Paging 分页 | ✅ |
| 使用 Flow 而非 LiveData | ✅ |
| 单例数据库实例 | ✅ |
| 添加迁移策略 | ✅ |

---

## 总结

Room 性能核心：
1. **索引**: 为常用查询字段添加索引
2. **分页**: 大数据集使用 Paging 库
3. **协程**: 使用 Flow + 协程
4. **单例**: 数据库实例只创建一次
5. **迁移**: 使用 Migration 管理版本

---

**相关文章**：
- [LiveData 为什么不够好 - 迁移到 Flow 完全指南](./android-livedata-to-flow.md)
- [Android View 与 Compose 互操作避坑指南](./android-view-compose-interop.md)
