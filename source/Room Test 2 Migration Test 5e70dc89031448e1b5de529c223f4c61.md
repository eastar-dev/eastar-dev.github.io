# Room Test 2 Migration Test

Create: 2021년 10월 5일 오후 5:08  
Tag: android, db, test

이글은 [Room Test 1 Dao Test](Room%20Test%201%20Dao%20Test%20046fa740a1f74274bda4ed5f8d6d4cf9.md) 과 연결되어 있습니다.

## Migration Test

room database에서는 버전이 있고 이 버전은 프로그램의 변경에 따라  
이전 사용자들을 업그레이드 하면서 변경해요. 이걸 마이그레이션 이라 해요

room Migration Test는 다른 test와 좀다르게 불편한 부분이 있고 주의해야 할점도 있어요.

이 글에서 핵심은 Migration Test고 이 부분에 대해서 좀 더심도 있게 다룰 생각이에요.

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

## Android Instrumented Test

### comumn add test

아래와 같은 코드를 test 할꺼에요.

기존의 DB version 1 → 2변경되면서 location과 photo가 추가됐어요.

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE USERS ADD COLUMN location TEXT")
        database.execSQL("ALTER TABLE USERS ADD COLUMN photo TEXT")
    }
}
```

Dao에선 UI Thread에서 접근이 불가능하기 때문에 여기서는 Coroutine를 사용했어요.  
따라서 다음과 같은 Instrumented Test Code를 만들수 있어요.

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
        helper.createDatabase(TEST_DB, 1).use {
            it.execSQL("INSERT INTO USERS ( name,level,date ) VALUES ('eastar', 'Level1' ,0);")
        }
        //when
        helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2).use {
            //then
            val cursor = it.query("SELECT COUNT(*) FROM USERS")
            cursor.moveToFirst()
            val actual = cursor.getLong(0)
            cursor.close()
            assertThat(actual).isEqualTo(1)
        }
    }
}
```

!!! warning 
    migration test 는 기본적으로 DB의 구조만 test 해요.  
    DB의 내용이 재대로 변경 됐는지는 직접  
    assert를 사용해서 확인 해야해요


### comumn change test

이번에는 난이도가 있는 변경 작업을 할려고해요

코드는 길지만 내용은 간단해요.  
date의 값이 long를 사용해서 저장 했는데  
String를 사용해서 변경 하는 내용입니다.

```kotlin
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        /** copy form data/schemas/dev.eastar.roomtest.data.db.TestDatabase/3.json */
        val tempTable =
            """CREATE TABLE IF NOT EXISTS `USERS_3` (`id` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, `name` TEXT NOT NULL, `level` TEXT NOT NULL, `date` TEXT NOT NULL, `location` TEXT, `photo` TEXT)"""
        database.execSQL(tempTable)
        val cursor =
            database.query(
                """SELECT
                      id,
                      name,
                      level,
                      date,
                      location,
                      photo
                      FROM USERS"""
            )

        while (cursor.moveToNext()) {
            val id = cursor.getLong(0)
            val name = cursor.getString(1)
            val level = cursor.getString(2)
            val date = cursor.getLong(3)
            val location = cursor.getStringOrNull(4)
            val photo = cursor.getStringOrNull(5)

            database.execSQL(
                """INSERT INTO USERS_3 (id, name, level, date, location, photo )
                     VALUES (
                     $id,
                     '$name',
                     '$level',
                     '${Instant.ofEpochMilli(date).atOffset(ZoneOffset.UTC).toLocalDateTime()}',
                     ${location?.let { "'$location'" } ?: "NULL"},
                     ${photo?.let { "'$photo'" } ?: "NULL"}
                    );"""
            )
        }
        database.execSQL("DROP TABLE USERS")
        database.execSQL("ALTER TABLE USERS_3 RENAME TO USERS")
    }
}
```
migration은 table 구조만 검증해주기 때문에 내용은 직접 검증해야 해요.  
그리고 생각보다 Table column type도 일일이 챙겨야 해요


!!! note

    migrateion 에서 삭제와 변경을 지원 하지 않기 때문에  
    dummyTable → 복사 → 원본삭제 → dummyTable이름 변경  
    이라는 엄청난 과정을 거쳐야 합니다.


??? tip "CREATE TABLE QUERY는 어디에서 가져올까?"
    ```
    val tempTable = """CREATE TABLE IF NOT EXISTS 
        `USERS_3` (`id` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, 
            `name` TEXT NOT NULL, 
            `level` TEXT NOT NULL, 
            `date` TEXT NOT NULL, 
            `location` TEXT, 
            `photo` TEXT)"""
    ```
    ![Untitled](Room%20Test%202%20Migration%20Test%205e70dc89031448e1b5de529c223f4c61/Untitled.png)
    
    아래쪽이 제가 사용하는 table entity인데 생각하는것과 생성된 table이 다를 수 있으니 복사해서 사용하기를 권장드려요.
    
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
    

test code

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
    fun migration_2_3() {
        //given
        helper.createDatabase(TEST_DB, 2).use {
            /** 0 => UTC 1970-01-01 00:00:00.000 */
            it.execSQL("INSERT INTO USERS ( name,level,date,location,photo ) VALUES ('eastar', 'Level1' ,0,NULL,NULL);")
        }
        //when
        helper.runMigrationsAndValidate(TEST_DB, 3, true, MIGRATION_2_3).use {
            //then
            val cursor = it.query("SELECT id,name,level,date,location,photo FROM USERS")
            cursor.moveToFirst()
            val id = cursor.getLong(0)//id
            val date = cursor.getString(3)//date
            val location = cursor.getStringOrNull(4)//location
            cursor.close()
            assertThat(id).isEqualTo(1)
            assertThat(DateConverter.text2LocalDateTime(date))
                .isEqualTo(DateConverter.text2LocalDateTime("1970-01-01T00:00:00.000"))
            assertThat(location).isNull()
        }
    }
}
```

### All Migration Test

이번 test 할내용은 1→3까지 모든 migration을 Test 하는 방법을 확인 해볼께요.

test code

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
    @Throws(IOException::class)
    fun migrateAll() {
        helper.createDatabase(TEST_DB, 1).use { }

        Room.databaseBuilder(
            InstrumentationRegistry.getInstrumentation().targetContext,
            TestDatabase::class.java,
            TEST_DB
        ).addMigrations(MIGRATION_1_2, MIGRATION_2_3).build().apply {
            openHelper.writableDatabase
            close()
        }
    }

}
```

!!! warning 
    migration test 는 기본적으로 DB의 구조만 test 해요.  
    DB의 내용이 재대로 변경 됐는지는 직접  
    assert를 사용해서 확인 해야해요

## Unit Test

Unit Test에서 test를 할려면 크게 다르지 않아요 다만. Robolectric을 사용해야 합니다.  
덤으로 실행중 오류가 발생하는것도 포함이에요.

### Robolectric

gradle

```kotlin
//robolectric
dependencies {
    testImplementation 'org.robolectric:robolectric:4.5.1'
    testImplementation 'androidx.test.ext:junit-ktx:1.1.3'
}
android {
    testOptions {
        unitTests {
            includeAndroidResources = true
        }
    }
}
```

unit test code

```kotlin
@Config(sdk = [30])
@RunWith(RobolectricTestRunner::class)
class TestDatabaseUnitTest {
   ...
```

제가 알고 있는 가장 간단한 Robolectric사용법이에요

### unit test code

!!! example "android test 랑 동일합니다."
    
    ```kotlin
    @Config(sdk = [30])
    @RunWith(RobolectricTestRunner::class)
    class MigrationUnitTest {
        companion object {
            const val TEST_DB = "migration-test"
        }
    
        @get:Rule
        val helper: MigrationTestHelper =
            MigrationTestHelper(
                InstrumentationRegistry.getInstrumentation(),
                TestDatabase::class.java,
            )
    
        @Test
        @Throws(IOException::class)
        fun migrateAll() {
            helper.createDatabase(TEST_DB, 1).use { }
    
            Room.databaseBuilder(
                InstrumentationRegistry.getInstrumentation().targetContext,
                TestDatabase::class.java,
                TEST_DB
            ).addMigrations(MIGRATION_1_2, MIGRATION_2_3).build().apply {
                openHelper.writableDatabase
                close()
            }
        }
    
        @Test
        fun migration_1_2() {
            //given
            helper.createDatabase(TEST_DB, 1).use {
                it.execSQL("INSERT INTO USERS ( name,level,date ) VALUES ('eastar', 'Level1' ,0);")
            }
            //when
            helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2).use {
                //then
                val cursor = it.query("SELECT COUNT(*) FROM USERS")
                cursor.moveToFirst()
                val actual = cursor.getLong(0)
                cursor.close()
                assertThat(actual).isEqualTo(1)
            }
        }
    
        @Test
        fun migration_2_3() {
            //given
            helper.createDatabase(TEST_DB, 2).use {
                /** 0 => UTC 1970-01-01 00:00:00.000 */
                it.execSQL("INSERT INTO USERS ( name,level,date,location,photo ) VALUES ('eastar', 'Level1' ,0,NULL,NULL);")
            }
            //when
            helper.runMigrationsAndValidate(TEST_DB, 3, true, MIGRATION_2_3).use {
                //then
                val cursor = it.query("SELECT id,name,level,date,location,photo FROM USERS")
                cursor.moveToFirst()
                val id = cursor.getLong(0)//id
                val date = cursor.getString(3)//date
                val location = cursor.getStringOrNull(4)//location
                cursor.close()
                assertThat(id).isEqualTo(1)
                assertThat(DateConverter.text2LocalDateTime(date))
                    .isEqualTo(DateConverter.text2LocalDateTime("1970-01-01T00:00:00.000"))
                assertThat(location).isNull()
            }
        }
    }
    ```
    

!!! fail "다만 아래와 같은 오류가 발생합니다."
    ```
    Cannot find the schema file in the assets folder. Make sure to include the exported json schemas in your test assert inputs. See https://developer.android.com/training/data-storage/room/migrating-db-versions#export-schema for details. Missing file: dev.eastar.roomtest.data.db.TestDatabase/1.json
    java.io.FileNotFoundException: Cannot find the schema file in the assets folder. Make sure to include the exported json schemas in your test assert inputs. See https://developer.android.com/training/data-storage/room/migrating-db-versions#export-schema for details. Missing file: dev.eastar.roomtest.data.db.TestDatabase/1.json
        at androidx.room.testing.MigrationTestHelper.loadSchema(MigrationTestHelper.java:484)
        at androidx.room.testing.MigrationTestHelper.createDatabase(MigrationTestHelper.java:238)
        at dev.eastar.roomtest.data.db.MigrationUnitTest.migrateAll(MigrationUnitTest.kt:33)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.base/java.lang.reflect.Method.invoke(Method.java:566)
        at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:59)
        at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
        at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:56)
        at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
        at org.junit.rules.TestWatcher$1.evaluate(TestWatcher.java:61)
        at org.junit.runners.ParentRunner$3.evaluate(ParentRunner.java:306)
        at org.robolectric.RobolectricTestRunner$HelperTestRunner$1.evaluate(RobolectricTestRunner.java:575)
        at org.robolectric.internal.SandboxTestRunner$2.lambda$evaluate$0(SandboxTestRunner.java:278)
        at org.robolectric.internal.bytecode.Sandbox.lambda$runOnMainThread$0(Sandbox.java:89)
        at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
        at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
        at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
        at java.base/java.lang.Thread.run(Thread.java:834)
    ```

    요약하면 exported json schemas가 안된것 같으니 다음 주소에 가서 자세한건 확인해봐라.  
    못찾은 파일은 ~~ .json파일이다. 란 내용입니다.  
    해당 주소에 가서보면 이미 설정한 gradle setting에 대한 이야기를 하고 있네요.
    
        Cannot find the schema file in the assets folder. 
        Make sure to include the exported json schemas in your test assert inputs. 
        See https://developer.android.com/training/data-storage/room/migrating-db-versions#export-schema for details. 
        Missing file: dev.eastar.roomtest.data.db.TestDatabase/1.json



!!! warning "아래 내용을 꼭 넣어야 되요. 첨부파일"
    
    [MigrationTestHelper.java](Room%20Test%202%20Migration%20Test%205e70dc89031448e1b5de529c223f4c61/MigrationTestHelper.java)

파일에 출처는 아래 코드를 추적해서 디컴파일한 코드에요.  
권장은 [https://cs.android.com/](https://cs.android.com/) 에서 코드를 찾으셔서 넣는게 좋습니다.

```kotlin
import androidx.room.testing.MigrationTestHelper
```

이 파일을 그냥 넣으시면 안되고  
unit test에서 getAssets()을 통해 1.json 파일을 읽을 수 없어서 나오는 오류이기 때문에
해당 부분을 수정해서 넣습니다. 
아래는 변경한 부분은 이에요.

```kotlin
private SchemaBundle loadSchema(Context context, int version) throws IOException {
//  InputStream input = context.getAssets().open(mAssetsFolder + "/" + version + ".json");
    InputStream input = new FileInputStream("./schemas/" + mAssetsFolder + "/" + version + ".json");
    return SchemaBundle.deserialize(input);
}
```

변경된 파일을 test코드에 넣으시고 import를 변경해주시면 됩니다.

```kotlin
//import androidx.room.testing.MigrationTestHelper
import dev.eastar.room.testing.MigrationTestHelper
```

이제 UnitTest를 돌려 볼까요?

![Untitled](Room%20Test%202%20Migration%20Test%205e70dc89031448e1b5de529c223f4c61/Untitled%201.png)

👏 👏 👏 

수고 하셨습니다.

## TL;DR

Sample Code

[GitHub - eastar-dev/RoomTest: room migrations](https://github.com/eastar-dev/RoomTest)