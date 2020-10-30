## 使用Hilt实现依赖注入

### 添加依赖项
 - 将 hilt-android-gradle-plugin 插件添加到项目的根级 build.gradle 文件中：
 
 ```
 buildscript {
    ...
    dependencies {
        ...
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.28-alpha'
    }
}
 ```
 - 将app的module下面的gradle文件添加下面依赖:
 
 ```
 ...
apply plugin: 'kotlin-kapt'//添加compiler的插件
apply plugin: 'dagger.hilt.android.plugin'//添加dagger2的插件
android {
    ...
}
dependencies {
        implementation "com.google.dagger:hilt-android:2.28-alpha"
        kapt "com.google.dagger:hilt-android-compiler:2.28-alpha"
}
```
- 如果要使用java8的功能，在app的module中的gradle文件添加如下配置:

```
android {
  ...
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
  }
}
```

### hilt使用

#### hilt应用类
- 将主应用的application添加`@HiltAndroidApp`注解，此时hilt才能在每个需要application的地方提供依赖项。

```
@HiltAndroidApp
class ExampleApplication : Application() { ... }

```
完成上面的步骤后，这一hilt组件会附加到Application对象的生命周期，并为其提供依赖项。它也是应用的父组件，其他组件可以访问它提供的依赖项。
   
#### hilt依赖注入android类


- 添加完主应用的`@HiltAndroidApp`注解后，需要为我们的几大组件添加依赖项注解，比如fragment添加了`@AndroidEntryPoint`注释后，则需要依赖的activity也需要添加`@AndroidEntryPoint`注释，目前android中的类支持添加依赖注释有:
   - Application（通过使用 @HiltAndroidApp）
   - Activity
   - Fragment
   - View
   - Service
   - BroadcastReceiver
   
比如activity中添加依赖项:

```
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() { ... }
```

如果想在activity这种组件中获取别的类的依赖，可以使用`@Inject`来获取依赖:

```
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() {

  @Inject lateinit var analytics: AnalyticsAdapter
  ...
}
```

当我们看到`AnalyticsAdapter`能直接通过`@Inject`获取它的依赖项的时候，是不是很好奇这是怎么拿到的，所以前提条件下得告诉hilt是怎么添加依赖项的。

#### @Inject提供类的依赖项
- 我们在上面能拿到AnalyticsAdapter依赖项，是因为在`AnalyticsAdapter`类中提前在构造器中定义了依赖项，所以在用的时候能直接拿到注入的依赖项:

```
class AnalyticsAdapter @Inject constructor(
  private val service: AnalyticsService
) { ... }
```
在上面代码中，带有注释的构造器是该类的依赖项，在上面例子中，AnalyticsService也是一个依赖项，所以我们在定义AnalyticsService的时候也需要提供它的依赖项，那么下面如何提供AnalyticsService的依赖项呢，这就需要讲到了hilt的自定义模块。

#### hilt模块
在上面例子中，AnalyticsService是一个接口，无法通过构造器提供依赖项，我们可以通过带有@Binds注释的抽象方法提供依赖项，此处的抽象方法主要通过返回值提供依赖项的接口类型，方法的传参来确定提供那种接口实例：

##### Binds提供接口依赖项

```
interface AnalyticsService {
  fun analyticsMethods()
}

//接口实例需要在构造器处提供依赖项
class AnalyticsServiceImpl @Inject constructor(
  ...
) : AnalyticsService { ... }

@Module
@InstallIn(ActivityComponent::class)
abstract class AnalyticsModule {

  @Binds
  abstract fun bindAnalyticsService(
    analyticsServiceImpl: AnalyticsServiceImpl//方法参数表示提供依赖项的实例
  ): AnalyticsService//方法的返回值表示提供依赖项的类型
}
```
- 在上面定义了AnalyticsService接口，AnalyticsServiceImpl在构造函数中提供了依赖项，后面在`AnalyticsModule`中通过@Binds在方法上提供了AnalyticsService类型的依赖项，方法的参数作为接口的依赖项实例，返回值作为依赖项的类型。
- Hilt 模块 AnalyticsModule 带有 @InstallIn(ActivityComponent::class) 注释，因为您希望 Hilt 将该依赖项注入 ExampleActivity。此注释意味着，AnalyticsModule 中的所有依赖项都可以在应用的所有 Activity 中使用。

##### Provides提供第三方实例
如果我们提供实例不是通过接口的形式提供，而是比如提供retrofit这种对象的时候，此时需要通过在hilt的@module模块中通过provides注解提供依赖项，比如下面的写法:

```
@Module
@InstallIn(ActivityComponent::class)
object AnalyticsModule {

  @Provides
  fun provideAnalyticsService(
  ): AnalyticsService {
      return Retrofit.Builder()
               .baseUrl("https://example.com")
               .build()
               .create(AnalyticsService::class.java)
  }
}
```
在上面的`provideAnalyticsService`方法中通过provides提供retrofit的依赖项，并且此处在InstallIn注解规定此依赖项是作用在activity上面的，如果我们不只是在activity这种android特定类中使用的话，我们可以通过`ApplicationComponent`注解来定义，这样retrofit对象作用在app中是全局的，任何地方都可以作为依赖项。

#### 为同一个类型定义多种实例
通常在我们okhttp中配置拦截器的时候，生成OkHttpClient对象的时候，可以指定不同的应用拦截器和网络拦截器，首先我们通过hilt的限定符定义好拦截器的名字:

```
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptorOkHttpClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class OtherInterceptorOkHttpClient
```
有了上面的两种限定符后，我们就可以通过限定符通过注解的形式添加两种不同的拦截器:

```
@Module
@InstallIn(ApplicationComponent::class)
object NetworkModule {

  @AuthInterceptorOkHttpClient
  @Provides
  fun provideAuthInterceptorOkHttpClient(
    authInterceptor: AuthInterceptor
  ): OkHttpClient {
      return OkHttpClient.Builder()
               .addInterceptor(authInterceptor)
               .build()
  }

  @OtherInterceptorOkHttpClient
  @Provides
  fun provideOtherInterceptorOkHttpClient(
    otherInterceptor: OtherInterceptor
  ): OkHttpClient {
      return OkHttpClient.Builder()
               .addInterceptor(otherInterceptor)
               .build()
  }
}
```
在上面代码中，通过我们定义的限定符，然后使用限定符作为注解的形式给两种拦截器添加依赖项，然后每个方法返回自己的okhttpClient对象。紧接着我们通过限定符作为方法注解来添加retrofit对象:

```
@Module
@InstallIn(ActivityComponent::class)
object AnalyticsModule {

  @Provides
  fun provideAnalyticsService(
    @AuthInterceptorOkHttpClient okHttpClient: OkHttpClient
  ): AnalyticsService {
      return Retrofit.Builder()
               .baseUrl("https://example.com")
               .client(okHttpClient)
               .build()
               .create(AnalyticsService::class.java)
  }
}
```
看到了没，上面在实例化retrofit的AnalyticsService时候，通过给方法的入参添加限定符注解，则完成了AuthInterceptor类型的拦截的OkHttpClient的依赖项。

如果要在activity中使用这种拦截器的依赖项的时候呢:

```
@AndroidEntryPoint
class ExampleActivity: AppCompatActivity() {

  @AuthInterceptorOkHttpClient
  @Inject lateinit var okHttpClient: OkHttpClient
}
```
如果在普通类中定义该依赖项呢:

```
class ExampleServiceImpl @Inject constructor(
  @AuthInterceptorOkHttpClient private val okHttpClient: OkHttpClient
){
}
```
普通类中就直接通过构造器中定义该限定符的注解就可以实现依赖项。

#### hilt中预定义的依赖项
比如我们在某个类中需要拿到application的context，可以通过hilt预定义的@ApplicationContext注解来实现

```
class InjectApplication constructor(@ApplicationContext context: ApplicationContext) {
}
```
还有我们的@ActivityContext也是照样可以拿到activity的上下文。

关于更多的组件作用域和更多的注解进入android官网[hilt实现依赖注入](https://developer.android.google.cn/training/dependency-injection/hilt-android)




