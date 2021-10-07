# Room Test 3 AutoMigration

Create: 2021년 10월 7일 오후 2:51  
Tag: android, db, test

이글은 다음과 같이 구성되어 있습니다.  
1. [Room Test 1 Dao Test](Room%20Test%201%20Dao%20Test%20046fa740a1f74274bda4ed5f8d6d4cf9.md)  
2. [Room Test 2 Migration Test](Room%20Test%202%20Migration%20Test%205e70dc89031448e1b5de529c223f4c61.md)  
3. [Room Test 3 AutoMigration](Room%20Test%203%20AutoMigration%20e87ce181d16c45ccb35db64b16b5be22.md)  


## AutoMigration Test

이전글에서 우리는 room test와 migration 및 migration test 방법에 대하여 알아 봤어요.    
이번에는 `2.4.0-alphaXX` 에서 나온 [Automated migrations](https://developer.android.com/training/data-storage/room/migrating-db-versions#automated)에 대하여 알아 볼께요.

room Migration에서 단점이라고 관려된 쿼리를 작성하고 별도의 Migration 객체 작업을 해줘야만 합니다.  
이 작업을 이제는 1줄로 끝낼수도 있어요.(꼭 그렇다는건 아니라는 함정이 있어요.)

## Gradle

```groovy
dependencies {
		def room_version = "2.4.0-alpha05"
    implementation "androidx.room:room-runtime:$room_version"
    kapt "androidx.room:room-compiler:$room_version"
    implementation "androidx.room:room-ktx:$room_version"
    testImplementation "androidx.room:room-testing:$room_version"
		androidTestImplementation "androidx.room:room-testing:$room_version"
}
```

migration test를 위한 설정

```kotlin
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments += [
                    "room.schemaLocation":"$projectDir/schemas".toString(),
                    "room.incremental":"true",
                    "room.expandProjection":"true"]
            }
        }
    }
}
```

## Auto Migration 

[https://github.com/eastar-dev/RoomTest/tree/room-automigration-1-2-column-add](https://github.com/eastar-dev/RoomTest/tree/room-automigration-1-2-column-add)

Migration 객체가 삭제되고 RoomDatabase정의시 @Database항목에 autoMigrations이 추가 됩니다.  
자세한건 아래 코드를 보면서 따라 올께요.

아래와 같은 코드를 test 할꺼에요.

기존의 DB version 1 → 2변경되면서 location과 photo가 추가됐어요.

=== "변경전"

    ```kotlin
    @Entity(tableName = "USERS")
    data class UserEntity(
        @PrimaryKey(autoGenerate = true)
        val id: Long = 0,
        val name: String = "",
        val level: Level,
        val date: LocalDateTime = LocalDateTime.now().withNano(0),
    )
    ```

=== "변경후"

    ```kotlin
    @Entity(tableName = "USERS")
    data class UserEntity(
        @PrimaryKey(autoGenerate = true)
        val id: Long = 0,
        val name: String = "",
        val level: Level,
        val date: LocalDateTime = LocalDateTime.now().withNano(0),
        val location: String? = "geo:${Random.latitude},${Random.longitude}",
        val photo: String? = "https://picsum.photos/id/${Random.nextInt(100)}/100/100",
    )
    ```


update code
=== "AutoMigration을 이용할 경우" 

    ```kotlin hl_lines="3"
    @Database(
        entities = [UserEntity::class],
        autoMigrations = [AutoMigration(from = 1, to = 2)],
        version = 2,
        exportSchema = true
    )
    @TypeConverters(DateConverter::class)
    abstract class TestDatabase : RoomDatabase() {
        abstract fun getUserDao(): UserDao
    }
    ```

=== "Migration Class를 이용할 경우"

    ```kotlin
    val MIGRATION_1_2 = object : Migration(1, 2) {
        override fun migrate(database: SupportSQLiteDatabase) {
            database.execSQL("ALTER TABLE USERS ADD COLUMN location TEXT")
            database.execSQL("ALTER TABLE USERS ADD COLUMN photo TEXT")
        }
    }
    ```

test code
=== "AutoMigration을 이용할 경우"

    ```kotlin hl_lines="20"
    class MigrationTest {
        companion object {
            const val TEST_DB = "migration-test"
        }

        @get:Rule
        val helper: MigrationTestHelper = MigrationTestHelper(
            InstrumentationRegistry.getInstrumentation(),
            TestDatabase::class.java,
        )

            @Test
        fun migration_1_2() {
            //given
            val level = "Level1"
            helper.createDatabase(TEST_DB, 1).use {
                it.execSQL("INSERT INTO USERS ( name,level,date ) VALUES ('eastar', '$level' ,0);")
            }
            //when
            helper.runMigrationsAndValidate(TEST_DB, 2, true).use {
                //then
                val cursor = it.query("SELECT level, location FROM USERS")
                cursor.moveToFirst()
                val actualLevel = cursor.getString(0)//level
                val actualLocation = cursor.getString(1)//location
                cursor.close()
                assertThat(actualLevel).isEqualTo(level)
                assertThat(actualLocation).isNull()
            }
        }
    }
    ```

=== "Migration Class를 이용할 경우"

    ```kotlin
    class MigrationTest {
        companion object {
            const val TEST_DB = "migration-test"
        }

        @get:Rule
        val helper: MigrationTestHelper = MigrationTestHelper(
            InstrumentationRegistry.getInstrumentation(),
            TestDatabase::class.java,
        )

            @Test
        fun migration_1_2() {
            //given
            val level = "Level1"
            helper.createDatabase(TEST_DB, 1).use {
                it.execSQL("INSERT INTO USERS ( name,level,date ) VALUES ('eastar', '$level' ,0);")
            }
            //when
            helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2).use {
                //then
                val cursor = it.query("SELECT level, location FROM USERS")
                cursor.moveToFirst()
                val actualLevel = cursor.getString(0)//level
                val actualLocation = cursor.getString(1)//location
                cursor.close()
                assertThat(actualLevel).isEqualTo(level)
                assertThat(actualLocation).isNull()
            }
        }
    }
    ```

!!! warning 

    migration test 는 기본적으로 DB의 구조만 test 해요. DB의 내용이 재대로 변경 됐는지는 직접  
    asset를 사용해서 확인 해야해요

??? tip "내부가 궁금해요"

    runMigrationsAndValidate 내부를 확인해보면  
    container.addMigrations(getAutoMigrations(mSpecs)); 함수를 통해서 자동으로  
    List<Migration>을 넣어 주고 있네요

    외부에서 사용 할수는 없네요  
    private List<Migration> getAutoMigrations(List<AutoMigrationSpec> userProvidedSpecs)  
    정의되어 있어요

    ```kotlin
    @SuppressLint("RestrictedApi")
    public SupportSQLiteDatabase runMigrationsAndValidate(String name, int version,
            boolean validateDroppedTables, Migration... migrations) throws IOException {
        File dbPath = mInstrumentation.getTargetContext().getDatabasePath(name);
        if (!dbPath.exists()) {
            throw new IllegalStateException("Cannot find the database file for " + name + ". "
                    + "Before calling runMigrations, you must first create the database via "
                    + "createDatabase.");
        }
        SchemaBundle schemaBundle = loadSchema(version);
        RoomDatabase.MigrationContainer container = new RoomDatabase.MigrationContainer();
        container.addMigrations(getAutoMigrations(mSpecs));
        container.addMigrations(migrations);
    ```

### Automatic migration specifications

[AutoMigrationSpec](https://developer.android.com/training/data-storage/room/migrating-db-versions#automigrationspec) 에 대하여 알아 볼께요.

migration 작업시 모호한 스키마가 있을때 사용하는 방법이에요.  
좀더 상세한 migration을 정의한 클레스라고 생각하면 됩니다.  
여기서 모호한 작업이라는것은

```kotlin
@DeleteTable
@RenameTable
@DeleteColumn
@RenameColumn
```

를 이야기 합니다.

삭제 TEST 시에는 아래 코드만 가지고는 오류가 발생한다

```kotlin
AutoMigration(from = 1, to = 2)
```

!!! error "친절한 가이드가 나온다"
    ```kotlin
    AutoMigration Failure in ‘AutoMigrationSpec_1_2’: Column ‘date’ in table ‘USERS’ has
    been either removed or renamed. Please annotate ‘AutoMigrationSpec_1_2’ with the @RenameColumn
    or @RemoveColumn annotation to specify the change to be performed:
    1) RENAME:
        @RenameColumn(
                tableName = "USERS",
                fromColumnName = "date",
                toColumnName = <NEW_COLUMN_NAME>
        )
    2) DELETE:
        @DeleteColumn=(
                tableName = "USERS",
                columnName = "date"
        )
    ```


### Single column delete

[https://github.com/eastar-dev/RoomTest/tree/room-automigration-1-2-single-column](https://github.com/eastar-dev/RoomTest/tree/room-automigration-1-2-single-column)

아래와 같이 어느 값이 삭제될지에 대한  
@DeleteColumn(tableName = "USERS", columnName = "date")  
값이 필요하다.

```kotlin
@Database(
    entities = [UserEntity::class],
    autoMigrations = [AutoMigration(from = 1, to = 2, spec = TestDatabase.AutoMigrationSpec_1_2::class)],
    version = 2,
    exportSchema = true
)
@TypeConverters(DateConverter::class)
abstract class TestDatabase : RoomDatabase() {
    abstract fun getUserDao(): UserDao

    @DeleteColumn(tableName = "USERS", columnName = "date")
    internal class AutoMigrationSpec_1_2 : AutoMigrationSpec {
        override fun onPostMigrate(db: SupportSQLiteDatabase) {
            Log.e("Invoked once auto migration is done")
        }
    }
}
```

### Delete 2 column

[https://github.com/eastar-dev/RoomTest/tree/room-automigration-1-2-double-column-delete](https://github.com/eastar-dev/RoomTest/tree/room-automigration-1-2-double-column-delete)

2개 이상의 column을 지우고 싶으면 아래와 같이 수정하면된다.

```kotlin
@Database(
    entities = [UserEntity::class],
    autoMigrations = [AutoMigration(from = 1, to = 2, spec = TestDatabase.AutoMigrationSpec_1_2::class)],
    version = 2,
    exportSchema = true
)
@TypeConverters(DateConverter::class)
abstract class TestDatabase : RoomDatabase() {
    abstract fun getUserDao(): UserDao

    @DeleteColumn.Entries(
        DeleteColumn(tableName = "USERS", columnName = "date"),
        DeleteColumn(tableName = "USERS", columnName = "level")
    )
    internal class AutoMigrationSpec_1_2 : AutoMigrationSpec {
        override fun onPostMigrate(db: SupportSQLiteDatabase) {
            Log.e("Invoked once auto migration is done")
        }
    }
}
```

## 총평

이상으로 Auto Migration 에 대하여 알아 봤다.

    @DeleteTable  
    @RenameTable  
    @DeleteColumn  
    @RenameColumn  

등이 지원되는 수정에서는 비교적 간단한 방법으로 작업을 할수 있었지만  
여전히 복잡한 작업은 수동 migration 작업이 필요해요.  
좀더 발전해서 column type변경같은 것도 지원해 준다면,  
좋을 것 같네요.

👏 👏 👏 

수고 하셨습니다.


이글은 다음과 같이 구성되어 있습니다.  
1. [Room Test 1 Dao Test](Room%20Test%201%20Dao%20Test%20046fa740a1f74274bda4ed5f8d6d4cf9.md)  
2. [Room Test 2 Migration Test](Room%20Test%202%20Migration%20Test%205e70dc89031448e1b5de529c223f4c61.md)  
3. [Room Test 3 AutoMigration](Room%20Test%203%20AutoMigration%20e87ce181d16c45ccb35db64b16b5be22.md)  


## TL;DR

Sample Code

[GitHub - eastar-dev/RoomTest: room migrations](https://github.com/eastar-dev/RoomTest)