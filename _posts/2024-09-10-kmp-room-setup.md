---
title: KMP (Kotlin Multiplatform) AndroidX Room setup and more
date: 2024-09-11 00:01:00 +0200
categories: [ Kotlin, KMP ]
tags: [ Kotlin, KMP ]
---

### Intro

<img src="/assets/img/kmp/kmp.png" alt ="" class="center" >

Kotlin Multiplatform (KMP) enables you to share code across various platforms such as Android, iOS, and JVM. Managing local data consistently across these platforms can be challenging. AndroidX Room, primarily used in Android development, can be integrated into a KMP project to handle local databases, providing a unified approach to data management. For some of us Android developers, this is a familiar and efficient library.

### Why Use AndroidX Room in KMP?

1. Unified Data Model: Define and manage data models consistently for Android and JVM, simplifying shared code management.

2. Shared Business Logic: Implement data access logic once and use it across platforms, reducing duplication and ensuring consistency.

3. Consistent API: Room’s type-safe API helps manage data effectively on Android and JVM, while shared code maintains consistency.

4. Development Efficiency: Avoid writing platform-specific database code for Android and JVM, integrating seamlessly with existing Room setups.

5. Platform-Specific Solutions: For iOS, use alternatives like SQLite or Core Data while keeping data models aligned with your shared code.

6. It's KMP ready

### Setup

We need to declare the required dependencies

```toml
[versions]
...
kotlin = "2.0.20"
ksp = "2.0.20-1.0.25"
androidx-room = "2.7.0-alpha07"
androidx-sqlite = "2.5.0-alpha07"
kotlinx-datetime = "0.6.1"

[libraries]
...
androidx-sqlite-bundled = { module = "androidx.sqlite:sqlite-bundled", version.ref = "androidx-sqlite" }
androidx-room-runtime = { module = "androidx.room:room-runtime", version.ref = "androidx-room" }
androidx-room-compiler = { module = "androidx.room:room-compiler", version.ref = "androidx-room" }
kotlinx-datetime = { module = "org.jetbrains.kotlinx:kotlinx-datetime", version.ref = "kotlinx-datetime" }

[plugins]
...
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
androidx-room = { id = "androidx.room", version.ref = "androidx-room" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

First, declare the required plugins in your project's `build.gradle.kts` file:

```kotlin
plugins {
    ...
    alias(libs.plugins.kotlin.multiplatform).apply(false)
    alias(libs.plugins.androidx.room).apply(false)
    alias(libs.plugins.ksp).apply(false)
}
```

Second, declare the required dependencies in your `shared's` or any module of your choice where the DB will live `build.gradle.kts` file:
```kotlin
plugins {
    ...
    alias(libs.plugins.androidx.room)
    alias(libs.plugins.ksp)
}


kotlin {
    ...

    sourceSets {
        commonMain.dependencies {
            implementation(libs.androidx.sqlite.bundled)
            implementation(libs.androidx.room.runtime)
            implementation(libs.kotlinx.datetime)
        }
    }
}

dependencies {
    //keep in mind that i am not compiling for iOS x86, if you do need it, just add it below
    add("kspAndroid", libs.androidx.room.compiler)
    add("kspIosSimulatorArm64", libs.androidx.room.compiler)
    add("kspIosArm64", libs.androidx.room.compiler)
    add("kspJvm", libs.androidx.room.compiler)
}


room {
    schemaDirectory("$projectDir/schemas")
}
```

Define your entities and DAOs in your shared module. Here’s an example:

```kotlin
@Entity(
    tableName = "cached_general_test_model",
    indices = [Index(value = ["testId", "originalFileType", "createdAt"])],
)
data class CachedGeneralTestModel(
    @ColumnInfo(name = "createdAt")
    val createdAt: Long,
    @ColumnInfo(name = "fileSize")
    val fileSize: Int,
    @ColumnInfo(name = "originalFileType")
    val originalFileType: String,
    @ColumnInfo(name = "source")
    val source: String?,
    @PrimaryKey
    @ColumnInfo(name = "testId")
    val testId: String
)

@Entity(tableName = "cached_test_date")
data class CachedTestDateModel(
    @PrimaryKey
    @ColumnInfo(name = "id")
    val id: String,

    @ColumnInfo(name = "dateOfInsertion")
    val dateOfInsertion: Instant,
)
```

Create a converter for Instant to Long:
```kotlin
class DateConverter {
    @TypeConverter
    fun fromLongToDate(timeInMillis: Long) = Instant.fromEpochMilliseconds(timeInMillis)

    @TypeConverter
    fun fromDateToLong(instant: Instant) = instant.toEpochMilliseconds()
}
```

Define DAOs for your entities:

```kotlin
@Dao
interface CachedGeneralTestDAO {
    @Query("SELECT * FROM cached_general_test_model")
    suspend fun getAllCachedGeneralTest(): List<CachedGeneralTestModel>

    @Query("SELECT dateOfInsertion FROM cached_daily_picks_date LIMIT 1")
    suspend fun getTopMostDate(): Long?

    @Insert(onConflict = REPLACE)
    suspend fun insertCachedGeneralTest(cachedGeneralTestModels: List<CachedGeneralTestModel>)

    @Insert(onConflict = REPLACE)
    suspend fun insertCachedGeneralTestDate(cachedTestDateModels: List<CachedTestDateModel>)
}
```

Since we'll use Write-Ahead logging we will relax the synchronization mode:
```kotlin
internal class TestDatabaseCallback : RoomDatabase.Callback() {
    override fun onOpen(connection: SQLiteConnection) {
        connection.apply {
            execSQL("PRAGMA synchronous = NORMAL")
        }
    }
}
```

Define your database class:

```kotlin

@Database(
    entities = [CachedGeneralTestModel::class, CachedTestDateModel::class],
    version = 1,
    exportSchema = true,
)
@TypeConverters(DateConverter::class)
@ConstructedBy(TestDatabaseConstructor::class)
abstract class TestDatabase : RoomDatabase() {
    abstract fun cachedGeneralTestDAO(): CachedGeneralTestDAO
}

@Suppress("NO_ACTUAL_FOR_EXPECT")
internal expect object TestDatabaseConstructor : RoomDatabaseConstructor<TestDatabase>

const val dbFileName = "test.db"

fun <T : RoomDatabase> RoomDatabase.Builder<T>.setDefaults(): RoomDatabase.Builder<T> = this.apply {
    setJournalMode(RoomDatabase.JournalMode.WRITE_AHEAD_LOGGING) //enabling WAL https://www.sqlite.org/wal.html
    setDriver(BundledSQLiteDriver())
    addCallback(TestDatabaseCallback())
    addFallbackInDebugOnly()
    setQueryCoroutineContext(Dispatchers.IO)
}

private fun <T : RoomDatabase> RoomDatabase.Builder<T>.addFallbackInDebugOnly(): RoomDatabase.Builder<T> {
    if (BuildKonfig.isDebug) { // we are using BuildKonfig as a demonstration reference
        fallbackToDestructiveMigration(true)
    }

    return this
}
```

Create an expect function for database creation:
```kotlin
expect fun createTestDatabase(): TestDatabase
```

For iOS:
`iosMain`
```kotlin

actual fun createTestDatabase(): TestDatabase {
    val dbFile = "${fileDirectory()}/$dbFileName"
    return Room.databaseBuilder<TestDatabase>(
        name = dbFile,
    )
        .setDefaults()
        .build()
}

@OptIn(ExperimentalForeignApi::class)
private fun fileDirectory(): String {
    val documentDirectory: NSURL? = NSFileManager.defaultManager.URLForDirectory(
        directory = NSDocumentDirectory,
        inDomain = NSUserDomainMask,
        appropriateForURL = null,
        create = false,
        error = null,
    )
    return requireNotNull(documentDirectory).path!!
}
```

For Android, you can include SQLCipher for encryption in release mode (keep in mind that enabling the cipher in debug mode won't make the DB readable by the inspector):
```toml
[versions]
...
android-database-sqlcipher = "4.5.4"
[libraries]
...
android-database-sqlcipher = { module = "net.zetetic:android-database-sqlcipher", version.ref = "android-database-sqlcipher" }
```

For Android:
`androidMain`

```kotlin
androidMain {
    dependencies {
        implementation(libs.android.database.sqlcipher)
    }
}
```

Define database creation with SQLCipher:
```kotlin
actual fun createTestDatabase(): TestDatabase {
    val dbFile = applicationContext.getDatabasePath(dbFileName)
    return Room.databaseBuilder<TestDatabase>(
        context = applicationContext,
        name = dbFile.absolutePath,
    )
        .setDefaults()
        .addCipherInReleaseOnly()
        .build()
}

private fun <T : RoomDatabase> RoomDatabase.Builder<T>.addCipherInReleaseOnly(
    cipherFactory: SupportFactory = SupportFactory(SQLiteDatabase.getBytes(applicationContext.packageName.toCharArray())), //or another more secure key, this is for demonstration only
    buildConfigType: BuildKonfig = BuildKonfig, // we are using BuildKonfig as a demonstration reference
): RoomDatabase.Builder<T> {
    if (!buildConfigType.isDebug) {
        openHelperFactory(cipherFactory)
    }

    return this
}
```

For JVM:
`jvmMain`

```kotlin
private val userHome get() = System.getProperty("user.home")
private const val APP_NAME = "TestApp"

actual fun createTestDatabase(): TestDatabase {
    val parentFolder = File(userHome + "/${APP_NAME}")
    if (!parentFolder.exists()) {
        parentFolder.mkdirs()
    }
    val databasePath = File(parentFolder, dbFileName)
    return Room.databaseBuilder<TestDatabase>(
        name = databasePath.absolutePath,
    )
        .setDefaults()
        .build()
}
```

### Closing Thoughts

Integrating AndroidX Room into a Kotlin Multiplatform project provides a unified approach to local data management on Android and JVM platforms. Room’s type-safe API and ease of integration make it a strong choice for these targets.

I've also shown how to setup [SQLDelight](/posts/sql-delight-kmp/).

I miss the auto complete when writing Room queries in Android Studio, as of now it doesn't work, but at least it provides SQL syntax highlight. 

In my experience, both SQLDelight and Room offer powerful solutions for data management in KMP projects. While SQLDelight provides robust SQL support and excellent multiplatform integration, Room offers familiarity and a type-safe API. I’ve enjoyed working with both and found each tool to be effective in different contexts. Ultimately, the choice between SQLDelight and Room depends on your specific project needs, preferences and knowledge, use the tool you know the best.

Until the next article, arrivederci!
