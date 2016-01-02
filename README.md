# TechStacks Kotlin Android App

To demonstrate Kotlin Native Types in action we've ported the Swift 
[TechStacks iOS App](https://github.com/ServiceStackApps/TechStacksApp) 
to a native Kotlin Android App to showcase the responsiveness and easy-of-use of leveraging 
[Kotlin Add ServiceStack Reference](https://github.com/ServiceStack/ServiceStack/wiki/Kotlin-Add-ServiceStack-Reference) 
in Android Projects. 

The Android TechStacks App can be 
[downloaded for free from the Google Play Store](https://play.google.com/store/apps/details?id=test.servicestack.net.techstackskotlin):

[![](https://raw.githubusercontent.com/ServiceStack/Assets/master/img/release-notes/techstacks-kotlin-app.png)](https://play.google.com/store/apps/details?id=test.servicestack.net.techstackskotlin)

### Data Binding

As there's no formal data-binding solution in Android we've adopted a lightweight iOS-inspired [Key-Value-Observable-like data-binding solution](https://github.com/ServiceStack/ServiceStack/wiki/Swift-Add-ServiceStack-Reference#observing-data-changes) in Android TechStacks in order to maximize knowledge-sharing and ease porting between native Swift iOS and Java/Kotlin Android Apps. 

Similar to the Swift TechStacks iOS App, all web service requests are encapsulated in a single 
[App.kt](https://github.com/ServiceStackApps/TechStacksKotlinApp/blob/master/src/TechStacks/app/src/main/java/servicestack/net/techstackskotlin/App.kt) 
class and utilizes Async Service Client API's in order to maintain a non-blocking and responsive UI. 

### Registering for Data Updates

In iOS, UI Controllers register for UI and data updates by implementing `*DataSource` and `*ViewDelegate` 
protocols, following a similar approach, Android Activities and Fragments register for Async Data callbacks 
by implementing the Custom interface `AppDataListener` below:

```kotlin
interface AppDataListener {
    fun onUpdate(data: AppData, dataType: DataType)
}
```

Where Activities or Fragments can then register itself as a listener when they're first created:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    App.data.addListener(this)
}
```

### Data Binding Async Service Responses

Then in `onCreateView` MainActivity calls the `AppData` singleton to fire off all async requests required to 
populate it's UI:

```kotlin
override fun onCreateView(inflater:LayoutInflater?, container:ViewGroup?, 
                          savedInstanceState:Bundle?): View? {
    App.data.loadAppOverview()
    ...
}
```

Where `loadAppOverview()` makes an async call to the `AppOverview` Service, storing the result in an AppData 
instance variable before notifying all registered listeners that `DataType.AppOverview` has been updated:

```kotlin
fun loadAppOverview(): AppData {
    if (appOverviewResponse != null) {
        onUpdate(DataType.AppOverview)
    }
    client.getAsync(AppOverview(), AsyncSuccess<AppOverviewResponse> {
        appOverviewResponse = it
        onUpdate(DataType.AppOverview)
    })
    return this
}
```

> Returning `this` allows expression chaining, reducing the boilerplate required to fire off multiple requests

Calling `onUpdate()` simply invokes the list of registered listeners with itself and the  enum DataType of what was changed, i.e:

```kotlin
fun onUpdate(dataType: DataType) {
    for (listener in listeners) {
        listener.onUpdate(this, dataType)
    }
}
```

The Activity can then update its UI within the `onUpdate()` callback by re-binding its UI Controls when 
relevant data has changed, in this case when `AppOverview` response has returned:

```kotlin
override fun onUpdate(data: App.AppData, dataType: App.DataType) {
    when (dataType) {
        App.DataType.AppOverview -> {
            val spinner = categorySpinner
            if (spinner != null) {
                val categories = data.appOverviewResponse!!.AllTiers.map { it.Title }
                spinner.adapter = ArrayAdapter(activity, android.R.layout.simple_spinner_item, categories)
            }

            val list = topRatedListView
            if (list != null) {
                refreshTopTechnologies(data, list)
            }
        }
    }
}
```

In this case the `MainActivity` home screen re-populates the Technology Category **Spinner** (aka Picker) 
and the Top Technologies **ListView** controls by assigning a new Android `ArrayAdapter`. 


### Images and Custom Binary Requests

The TechStacks Android App can take advantage of the Custom Service Client API's to download images 
asynchronously. As images can be fairly resource and bandwidth intensive they're stored in a simple 
Dictionary Cache to minimize any unnecessary CPU and network resources, i.e:

```kotlin
internal var imgCache = HashMap<String, Bitmap>()
fun loadImage(imgUrl: String, callback: ImageResult) {
    val img = imgCache[imgUrl]
    if (img != null) {
        callback.success(img)
        return
    }

    client.getAsync(imgUrl, AsyncSuccess<ByteArray> {
        val img = AndroidUtils.readBitmap(it)
        imgCache.put(imgUrl, img)
        callback.success(img)
    })
}
```
