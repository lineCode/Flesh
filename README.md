# Flesh（果肉）

果肉一款福利满满的app，数据源[mzitu][3]，MD风格的界面。

如果你是一位想学习一下Kotlin的同学，那么绝对不要错过Flesh。如Kotlin所说它与Java完美兼容，所以这里有Kotlin调用Java，同时也有Java调用Kotlin。在学习的同时还将有另一种体验。

国际惯例，先上福利。[Release1.0](https://github.com/Kerr1Gan/Flesh/releases/download/170929/flesh-release.apk)

![fuli](art/fuli.gif)

特点
--------
1. 列表显示图片，点击查看更多。
2. 快速跳转至顶部，底部，指定位置。
3. 收藏，查看历史记录。
4. 设置壁纸。
5. 离线缓存。

组成
--------
1. 语言：Kotlin，Java
2. 网络请求：HttpUrlConnection
3. 数据库：Sqlite
4. 数据源：Jsoup
5. 第三方库：Glide

概述
--------
**1)** 网络请求

网络框架并没有使用RxRetrofit等，为了保证精简高效直接使用的HttpUrlConnection
+ get 
```kotlin
val request = AsyncNetwork()
request.request(Constants.HOST_MOBILE_URL, null)
request.setRequestCallback(object : IRequestCallback {
    override fun onSuccess(httpURLConnection: HttpURLConnection?, response: String) {
        //todo
    }
})
```
+ post
```kotlin
val request = AsyncNetwork()
request.request(Constants.HOST_MOBILE_URL, mutableMapOf())
request.setRequestCallback(object : IRequestCallback {
    override fun onSuccess(httpURLConnection: HttpURLConnection?, response: String) {
        //todo
    }
})
```
**2)** 数据库

数据库没有使用第三方框架，直接使用的sql语句。
```sql
CREATE TABLE tb_class_page_list ( 
                    _id           INTEGER PRIMARY KEY ASC AUTOINCREMENT,
                    href          STRING  UNIQUE,
                    description   STRING,
                    image_url     STRING,
                    id_class_page INTEGER REFERENCES tb_class_page (_id) ON DELETE CASCADE ON UPDATE CASCADE,
                    [index]       INTEGER);
```
**3)** 读写缓存

Serializable的效率远低于Parcelable，所以采用Parcelable实现的缓存机制，速度快了大概7，8倍。
+ 读取缓存
```kotlin
val helper = PageListCacheHelper(container?.context?.filesDir?.absolutePath)
val pageModel: Any? = helper.get(key)
```
+ 写入缓存
```kotlin
val helper = PageListCacheHelper(context.filesDir.absolutePath)
helper.put(key, object)
```
+ 删除缓存
```kotlin
val helper = PageListCacheHelper(context.filesDir.absolutePath)
helper.remove(key)
```
**4)** jsoup获取数据

由于数据是用从html页面中提取的，所以速度偏慢，为了不影响体验做了一套缓存机制，来做到流畅体验。
```java
Document doc = Jsoup.parse(html);
Elements elements = body.getElementsByTag("a");
String text = elements.get(0).text();
String imageUrl = elements.get(0).attr("src");
...
```
**5)** 组件
+ Activity fragment替代activity来显示新界面

    因为activity需要在Manifiest中注册，所以当有多个activity的时候，就需要编写很长的Manifiest文件，严重影响了Manifiest的可读性，界面的风格也比较笨重。所以一个新页面就注册一个activity不太合适，我们通过用activity做为容器添加不同的Fragment来达到注册一个activity启动多个不同页面的效果。生命周期由activity管理，更方便简洁。
    ```kotlin
    open class BaseFragmentActivity : BaseActionActivity() {

        companion object {

            private const val EXTRA_FRAGMENT_NAME = "extra_fragment_name"
            private const val EXTRA_FRAGMENT_ARG = "extra_fragment_arguments"

            @JvmOverloads
            @JvmStatic
            fun newInstance(context: Context, fragment: Class<*>, bundle: Bundle? = null,
                                        clazz: Class<out Activity> = getActivityClazz()): Intent {
                val intent = Intent(context, clazz)
                intent.putExtra(EXTRA_FRAGMENT_NAME, fragment.name)
                intent.putExtra(EXTRA_FRAGMENT_ARG, bundle)
                return intent
            }

            protected open fun getActivityClazz(): Class<out Activity> {
                return BaseFragmentActivity::class.java
            }
        }

        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)
            setContentView(R.layout.activity_base_fragment)
            val fragmentName = intent.getStringExtra(EXTRA_FRAGMENT_NAME)
            var fragment: Fragment? = null
            if (TextUtils.isEmpty(fragmentName)) {
                //set default fragment
    //            fragment = makeFragment(MainFragment::class.java!!.getName())
            } else {
                val args = intent.getBundleExtra(EXTRA_FRAGMENT_ARG)
                try {
                    fragment = makeFragment(fragmentName)
                    if (args != null)
                        fragment!!.arguments = args
                } catch (e: Exception) {
                    e.printStackTrace()
                }

            }

            if (fragment == null) return

            supportFragmentManager
                    .beginTransaction()
                    .replace(R.id.fragment_container, fragment)
                    .commit()
        }

        fun makeFragment(name: String): Fragment? {
            try {
                val fragmentClazz = Class.forName(name)
                return fragmentClazz.newInstance() as Fragment
            } catch (e: Exception) {
                e.printStackTrace()
            }

            return null
        }

    }
    ```


**6)** 测试

性能测试，为了避免干扰，我们使用AndroidTest进行测试。
```java
@Test
public void useAppContext() throws Exception {
        // Context of the app under test.
        final Context appContext = InstrumentationRegistry.getTargetContext();
        List<PageModel.ItemModel> itemModels = new ArrayList<PageModel.ItemModel>();
        for (int i = 0; i < 10000; i++) {
            itemModels.add(new PageModel.ItemModel("", "", ""));
        }
        //db begin
        appContext.deleteDatabase("test4");
        SQLiteDatabase db = DatabaseManager.getInstance(appContext).getHelper(appContext, "test4").getWritableDatabase();
        long start = System.currentTimeMillis();
        db.beginTransaction();
        LikeTableImpl impl = new LikeTableImpl();
        for (int i = 0; i < 10000; i++) {
            impl.addLike(db, "page" + i, "", "", "");
        }
        db.setTransactionSuccessful();
        db.endTransaction();
        Log.e("cache speed", "db save time " + (System.currentTimeMillis() - start));

        start = System.currentTimeMillis();
        db.beginTransaction();
        impl.getAllLikes(db);
        db.setTransactionSuccessful();
        db.endTransaction();
        Log.e("cache speed", "db read time " + (System.currentTimeMillis() - start));
        db.close();
        //db end

        //parcel begin
        start = System.currentTimeMillis();
        PageModel model = new PageModel(itemModels);
        PageListCacheHelper helper = new PageListCacheHelper(appContext.getCacheDir().getAbsolutePath());
        helper.put("test", model);
        Log.e("cache speed", "parcel save time " + (System.currentTimeMillis() - start));

        start = System.currentTimeMillis();
        helper.get("test");
        Log.e("cache speed", "parcel read time " + (System.currentTimeMillis() - start));
        //parcel end

        //Serializable
        List<TestItemModel> testModels = new ArrayList<TestItemModel>();
        for (int i = 0; i < 10000; i++) {
            testModels.add(new TestItemModel("", "", ""));
        }
        start = System.currentTimeMillis();
        ObjectOutputStream os = new ObjectOutputStream(new FileOutputStream(new File(appContext.getCacheDir().getAbsolutePath(), "serializable")));
        os.writeObject(testModels);
        Log.e("cache speed", "serializable save time " + (System.currentTimeMillis() - start));
        os.close();
        start = System.currentTimeMillis();
        ObjectInputStream is = new ObjectInputStream(new FileInputStream(new File(appContext.getCacheDir().getAbsolutePath(), "serializable")));
        is.readObject();
        Log.e("cache speed", "serializable read time " + (System.currentTimeMillis() - start));
        is.close();
    }
```
测试结果：(单位ms)

1000条数据
```
db save time 73
db read time 9
parcel save time 76
parcel read time 59
serializable save time 150
serializable read time 96
```
10000条数据
```
db save time 684
db read time 100
parcel save time 141
parcel read time 136
serializable save time 1479
serializable read time 996
```
50000条数据
```
db save time 3348
db read time 890
parcel save time 493
parcel read time 498
serializable save time 7571
serializable read time 4975
```
从结果上可以看到数据库和parcel的存储各有优劣，而Serializable则是劣势明显。由于项目上的各种原因加上我要直接存储对象，所以最后使用了Parcel的存储方式来实现无网络状态下显示数据。并没有使用数据库，但是数据库应该是最优解，可以用一些对象型数据库框架例如[GreenDao][5]等等。

ProGuard
--------
```pro
-keep class org.jsoup.**{*;}
-keep public class com.ecjtu.netcore.jsoup.SoupFactory{*;}
-keep public class * extends com.ecjtu.netcore.jsoup.BaseSoup{*;}
-keep public class com.ecjtu.netcore.Constants{static <fields>;}
-keep public class com.ecjtu.netcore.model.**{*;}
-keep public class com.ecjtu.netcore.network.BaseNetwork{public <methods>;}
-keep public class * extends com.ecjtu.netcore.network.BaseNetwork{ public <methods>; }
-keep public interface com.ecjtu.netcore.network.IRequestCallback{*;}
-keep public class * extends android.support.design.widget.CoordinatorLayout$Behavior{*;}
```

Contributing
------------
contributors submmit pull requests.

Thanks
------
* The **Glide** [image loading framework][1] Flesh's image loader is based on.
* The **Bugly** [app monitoring tools][2] Flesh's log collector.
* Everyone who has contributed code and reported issues!

Author
------
KerriGan - mnsync@outlook.com or ethanxiang95@gmail.com

License
-------
[Apache2][4]

Disclaimer
---------
Only available for study and communication.If the flesh violate your rights,we can delete immediately violate to your rights and interests content.

[1]: https://github.com/bumptech/glide
[2]: https://bugly.qq.com/v2/
[3]: http://www.mzitu.com
[4]: https://github.com/Kerr1Gan/Flesh/blob/master/LICENSE
[5]: http://greenrobot.org/greendao/
