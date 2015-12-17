# TechStacks Android App

To demonstrate Java Native Types in action we've ported the Swift [TechStacks iOS App](https://github.com/ServiceStackApps/TechStacksApp) to a native Java Android App to showcase the responsiveness and easy-of-use of leveraging Java Add ServiceStack Reference in Android Projects. 

The Android TechStacks App can be [downloaded for free from the Google Play Store](https://play.google.com/store/apps/details?id=servicestack.net.techstacks):

[![](https://raw.githubusercontent.com/ServiceStack/Assets/master/img/release-notes/techstacks-android-app.jpg)](https://play.google.com/store/apps/details?id=servicestack.net.techstacks)

### Data Binding
As there's no formal data-binding solution in Android we've adopted a lightweight iOS-inspired [Key-Value-Observable-like data-binding solution](https://github.com/ServiceStack/ServiceStack/wiki/Swift-Add-ServiceStack-Reference#observing-data-changes) in Android TechStacks in order to maximize knowledge-sharing and ease porting between native Swift iOS and Java Android Apps. 

Similar to the Swift TechStacks iOS App, all web service requests are encapsulated in a single [App.java](https://github.com/ServiceStack/ServiceStack.Java/blob/master/src/AndroidClient/techstacks/src/main/java/servicestack/net/techstacks/App.java) class and utilizes Async Service Client API's in order to maintain a non-blocking and responsive UI. 

### Registering for Data Updates
In iOS, UI Controllers register for UI and data updates by implementing `*DataSource` and `*ViewDelegate` protocols, following a similar approach, Android Activities and Fragments register for Async Data callbacks by implementing the Custom interface `AppDataListener` below:

```java
public static interface AppDataListener
{
    public void onUpdate(AppData data, DataType dataType);
}
```

Where Activities or Fragments can then register itself as a listener when they're first created:
```java
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    App.getData().addListener(this);
}
```

### Data Binding Async Service Responses
Then in `onCreateView` MainActivity calls the `AppData` singleton to fire off all async requests required to populate it's UI:
```java
public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle state) {
    App.getData().loadAppOverview();
    ...
}
```

Where `loadAppOverview()` makes an async call to the `AppOverview` Service, storing the result in an AppData instance variable before notifying all registered listeners that `DataType.AppOverview` has been updated:
```java
public AppData loadAppOverview(){
    client.getAsync(new AppOverview(), new AsyncResult<AppOverviewResponse>() {
        @Override
        public void success(AppOverviewResponse response){
            appOverviewResponse = response;
            onUpdate(DataType.AppOverview);
        }
    });
    return this;
}
```
> Returning `this` allows expression chaining, reducing the boilerplate required to fire off multiple requests

Calling `onUpdate()` simply invokes the list of registered listeners with itself and the  enum DataType of what was changed, i.e:
```java
public void onUpdate(DataType dataType){
    for (AppDataListener listener : listeners){
        listener.onUpdate(this, dataType);
    }
}
```

The Activity can then update its UI within the `onUpdate()` callback by re-binding its UI Controls when relevant data has changed, in this case when `AppOverview` response has returned:
```java
@Override
public void onUpdate(App.AppData data, App.DataType dataType) {
    switch (dataType) {
        case AppOverview:
            Spinner spinner = (Spinner)getActivity().findViewById(R.id.spinnerCategory);
            ArrayList<String> categories = map(data.getAppOverviewResponse().getAllTiers(), 
                new Function<Option, String>() {
                    @Override public String apply(Option option) {
                        return option.getTitle();
                    }
                });
            spinner.setAdapter(new ArrayAdapter<>(getActivity(),
                android.R.layout.simple_spinner_item, categories));

            ListView list = (ListView)getActivity().findViewById(R.id.listTopRated);
            ArrayList<String> topTechnologyNames = map(getTopTechnologies(data),
                new Function<TechnologyInfo, String>() {
                    @Override public String apply(TechnologyInfo technologyInfo) {
                        return technologyInfo.getName() + " (" + technologyInfo.getStacksCount() + ")";
                    }
                });
            list.setAdapter(new ArrayAdapter<>(getActivity(),
                android.R.layout.simple_list_item_1, topTechnologyNames));
            break;
    }
}
```

In this case the `MainActivity` home screen re-populates the Technology Category **Spinner** (aka Picker) and the Top Technologies **ListView** controls by assigning a new Android `ArrayAdapter`. 

### Functional Java Utils
The above example also introduces the `map()` functional util we've also included in the **net.servicestack:client** dependency to allow usage of Functional Programming techniques to transform, query and filter data given Android's Java 7 lack of any language or library support for Functional Programming itself. Unfortunately lack of closures in Java forces more boilerplate than otherwise would be necessary as it needs to fallback to use anonymous Type classes to capture delegates. Android Studio also recognizes this pattern as unnecessary noise and will automatically collapse the code into a readable closure syntax, with what the code would've looked like had Java supported closures, e.g:

![Android Studio Collapsed Closure](https://raw.githubusercontent.com/ServiceStack/Assets/master/img/release-notes/androidstudio-collapse-closure.png)

### [Func.java](https://github.com/ServiceStack/ServiceStack.Java/blob/master/src/AndroidClient/client/src/main/java/net/servicestack/client/Func.java) API

The [Func.java](https://github.com/ServiceStack/ServiceStack.Java/blob/master/src/AndroidClient/client/src/main/java/net/servicestack/client/Func.java) static class contains a number of common functional API's providing a cleaner and more robust alternative to working with Data than equivalent imperative code. We can take advantage of **static imports** in Java to import the namespace of all utils with the single import statement below:
```java
import static net.servicestack.client.Func.*;
```

Which will let you reference all the Functional utils below without a Class prefix:
```java
ArrayList<R> map(Iterable<T> xs, Function<T,R> f)
ArrayList<T> filter(Iterable<T> xs, Predicate<T> predicate)
void each(Iterable<T> xs, Each<T> f)
T first(Iterable<T> xs)
T first(Iterable<T> xs, Predicate<T> predicate)
T last(Iterable<T> xs)
T last(Iterable<T> xs, Predicate<T> predicate)
boolean contains(Iterable<T> xs, Predicate<T> predicate)
ArrayList<T> skip(Iterable<T> xs, int skip)
ArrayList<T> skip(Iterable<T> xs, Predicate<T> predicate)
ArrayList<T> take(Iterable<T> xs, int take)
ArrayList<T> take(Iterable<T> xs, Predicate<T> predicate)
boolean any(Iterable<T> xs, Predicate<T> predicate)
boolean all(Iterable<T> xs, Predicate<T> predicate)
ArrayList<T> expand(Iterable<T>... xss)
T elementAt(Iterable<T> xs, int index)
ArrayList<T> reverse(Iterable<T> xs)
reduce(Iterable<T> xs, E initialValue, Reducer<T,E> reducer)
E reduceRight(Iterable<T> xs, E initialValue, Reducer<T,E> reducer)
String join(Iterable<T> xs, String separator)
ArrayList<T> toList(Iterable<T> xs)
```

### Images and Custom Binary Requests
The TechStacks Android App also takes advantage of the Custom Service Client API's to download images asynchronously. As images can be fairly resource and bandwidth intensive they're stored in a simple Dictionary Cache to minimize any unnecessary CPU and network resources, i.e:
```java
HashMap<String,Bitmap> imgCache = new HashMap<>();
public void loadImage(final String imgUrl, final ImageResult callback) {
    Bitmap img = imgCache.get(imgUrl);
    if (img != null){
        callback.success(img);
        return;
    }

    client.getAsync(imgUrl, new AsyncResult<byte[]>() {
        @Override
        public void success(byte[] imgBytes) {
            Bitmap img = AndroidUtils.readBitmap(imgBytes);
            imgCache.put(imgUrl, img);
            callback.success(img);
        }
    });
}
```

The TechStacks App uses the above API to download screenshots and load their Bitmaps in `ImageView` UI Controls, e.g:

```java
String imgUrl = result.getScreenshotUrl();
final ImageView img = (ImageView)findViewById(R.id.imgTechStackScreenshotUrl);
data.loadImage(imgUrl, new App.ImageResult() {
    @Override public void success(Bitmap response) {
        img.setImageBitmap(response);
    }
});
```

