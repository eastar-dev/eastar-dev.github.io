# Room Test 1 Dao Test

Create: 2021년 9월 30일 오후 4:25  
Tag: android, db, test

이글은 [Room Test 2 Migration Test](Room%20Test%202%20Migration%20Test%205e70dc89031448e1b5de529c223f4c61.md) 과 연결되어 있습니다.


## Room?

Room 은 Jetpack libraries중에서 DB관련 라이브러리 이며,   
주요 기능으로 DB사용의 장점과, 쉬운 Data 객체 변환이 특징인 라이브러리에요.  

글을 작성하는 시점을 기준으로 Stable Release 2.3.0, Alpha 2.4.0-alpha05이 나와있어요.

2.4.0-alphaXX 부터 [`AutoMigration`](https://developer.android.com/reference/androidx/room/AutoMigration) 을 지원해요.  
자세한건 아래 링크를 참조

[AutoMigration 참조 링크](https://developer.android.com/jetpack/androidx/releases/room)

## Gradle

```groovy
dependencies {
	def room_version = "2.4.0-alpha05"
    implementation "androidx.room:room-runtime:$room_version"
    kapt "androidx.room:room-compiler:$room_version"
    implementation "androidx.room:room-ktx:$room_version"
}
```

## Room Test

[](https://developer.android.com/training/data-storage/room/testing-db)

Room을 test 하기위한 방법은 2가지가 있어요.

첫번째는 android device를 사용하는 test 방법이고 InstrumentedTest 인데  
다른 한가지는 Robolectric을 이용한 Unit Test 방법이에요.

이 중 안드로이드에서 권장하는 안.하.는 방법은 UnitTest에요   
하지만 InstrumentedTest 방법은 device가 필요해요.

이건 CI툴에 자동화를 돌리기 어렵운 단점이 있어요.

이 글에선 android device없이 어디까지 room testing이 가능한가 해볼꺼에요.

### Dao Test

room에서 test case를 만들 만한것이 무엇이 있을까요?

기본적인 database 생성  및 CRUD 등이 있겠네요.  
Room은 CRUD의 대부분을 Dao class에서 할수 있어요.

샘플에서 [요기 부분](https://github.com/eastar-dev/RoomTest/blob/master/data/src/main/java/dev/eastar/roomtest/data/db/UserDao.kt)이에요.  
여기를 test하는 코드를 만들면 Room test 에서 절반은 했다고 생각하시면 되요.

### Migration Test

room database에서는 버전이 있고 이 버전은 프로그램의 변경에 따라  
이전 사용자들을 업그레이드 하면서 변경해요. 이걸 마이그레이션 이라 해요

room Migration Test는 다른 test와 좀다르게 불편한 부분이 있고 주의해야 할점도 있어요.

다음 글에서 좀더 상세하게 Migration Test를 다룰꺼에요.

## Android Instrumented Test

아래와 같은 코드를 test 할꺼에요

```kotlin
@Dao
interface UserDao {

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(entity: UserEntity): Long

    @Delete
    suspend fun deleteUser(entity: UserEntity): Int

    @Update
    suspend fun updateUser(entity: UserEntity): Int

    @Query("SELECT * FROM USERS WHERE id = :id")
    suspend fun getUser(id: Long): UserEntity?

    @Query("SELECT * FROM USERS")
    fun getAllUsers(): Flow<List<UserEntity>>
}
```

Dao에선 UI Thread에서 접근이 불가능하기 때문에 여기서는 Coroutine를 사용했어요.  
따라서 다음과 같은 Instrumented Test Code를 만들수 있어요.

```kotlin
@RunWith(AndroidJUnit4::class)
class TestDatabaseTest {
    private lateinit var userDao: UserDao
    private lateinit var db: TestDatabase

    @Before
    fun createDb() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        db = Room.inMemoryDatabaseBuilder(
            context, TestDatabase::class.java
        ).build()
        userDao = db.getUserDao()
    }

    @After
    @Throws(IOException::class)
    fun closeDb() {
        db.close()
    }

    @Test
    @Throws(Exception::class)
    fun writeUserAndReadInList() = runBlocking {
        //givens
        val user = UserEntity(0, "hello", Level.Level1, LocalDateTime.now())

        //when
        val id = userDao.insertUser(user)
        val actual = userDao.getUser(id)

        //then
        assertThat(actual).let {
            it.isNotNull()
            it.isEqualTo(user.copy(id = id))
        }
    }
}
```

!!! tip

    inMemoryDatabase를 이용해서 test를 하고  
    test block을 coroutine `runBlocking`으로 감싸서 test 합니다.

!!! etc

    ```kotlin
    import com.google.common.truth.Truth.assertThat
    ```

## Unit Test

Unit Test에서 test를 할려면 크게 다르지 않아요 다만. Robolectric을 사용해야 합니다.

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
}
```

제가 알고 있는 가장 간단한 Robolectric사용법이에요

### unit test code

```kotlin
@Config(sdk = [30])
@RunWith(RobolectricTestRunner::class)
class TestDatabaseUnitTest {
    private lateinit var userDao: UserDao
    private lateinit var db: TestDatabase

    @Before
    fun createDb() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        db = Room.inMemoryDatabaseBuilder(
            context, TestDatabase::class.java
        ).build()
        userDao = db.getUserDao()
    }

    @After
    @Throws(IOException::class)
    fun closeDb() {
        db.close()
    }

    @Test
    @Throws(Exception::class)
    fun writeUserAndReadInList() = runBlocking {
        //givens
        val user = UserEntity(0, "hello", Level.Level1, LocalDateTime.now().withNano(0))

        //when
        val id = userDao.insertUser(user)
        val actual = userDao.getUser(id)

        //then
        Truth.assertThat(actual).let {
            it.isNotNull()
            it.isEqualTo(user.copy(id = id))
        }
    }
}
```

## TL;DR

Sample Code

[GitHub - eastar-dev/RoomTest: room migrations](https://github.com/eastar-dev/RoomTest)