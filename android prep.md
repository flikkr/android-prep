# Android Notes and Snippets

---

## Navigation

### Activity / Intents

- Two types of intents: **Explicit** and **Implicit**

  - **Explicit**: We know where to bring the user, e.g. a specific screen in an app
  - **Implicit**: We don't know the destination, e.g. 3rd-party app, webview, share button, etc.

- **Create and launch intent**

```kotlin
holder.button.setOnClickListener {
    val context = holder.view.context
    val intent = Intent(context, DetailActivity::class.java)

    intent.putExtra("letter", holder.button.text.toString()) // add args
    context.startActivity(intent) // start activity
}
```

- **Retrieve args from intent**

`val letterId = intent?.extras?.getString("letter").toString()`

### Activity Lifecycle

- [Lifecycle events](https://developer.android.com/codelabs/basic-android-kotlin-training-activity-lifecycle/img/f6b25a71cec4e401.png):
  - **OnCreate / OnDestroy**: Called once when activity is created, used for initialising data and listeners. `OnDestroy` called when the activity is no longer used and can be cleaned up by the garbage collector, e.g. app is killed.
  - **OnStart / OnStop**: Called after `OnCreate` and can be called multiple times. after executing, the screen is visible. If user launches app and then goes back to home screen, `OnStop` is called, i.e. called when app loses visibility.
  - **OnResume / OnPause**: Called after `OnStart` and adds focus to the screen for user to interact with. `OnPause` removes focus, e.g. user clicks share button and overlay appears, app is still visible but "paused", so `OnPause` is called but not `OnStop`.
- **savedInstanceState**: Used to restore the state of an activity in case of interruption, e.g. screen rotation causing activity to restart.

  - **onSaveInstanceState()**: Called after `OnStop`, used for storing instance state.

  ```kotlin
  const val KEY_REVENUE = "revenue_key"

  override fun onSaveInstanceState(outState: Bundle) {
      outState.putInt(KEY_REVENUE, 66) // add data to save bundle

      super.onSaveInstanceState(outState)

      Log.d(TAG, "onSaveInstanceState Called")
  }
  ```

  - **Restore state in OnCreate**:

  ```kotlin
  override fun onCreate(savedInstanceState: Bundle?) {
      super.onCreate(savedInstanceState)

      if (savedInstanceState != null) {
          revenue = savedInstanceState.getInt(KEY_REVENUE, 0)
      }
  }
  ```

### Fragment Lifecycle

- Similar to Activity lifecycle ([see image](https://developer.android.com/codelabs/basic-android-kotlin-training-fragments-navigation-component/img/8dc30a4c12ab71b.png))
- Inflating layout happens in `onCreateView` instead of `onCreate` for Activities.
- Binding views happens in `onViewCreated`

### Navigation Component

- Navigate using the NavController

```kotlin
val action = LetterListFragmentDirections
                .actionLetterListFragmentToWordListFragment(
                    letter = holder.button.text.toString()
                )
holder.view.findNavController().navigate(action)
```

- E.g. usage

```kotlin
val navHostFragment = supportFragmentManager
    .findFragmentById(R.id.nav_host_fragment) as NavHostFragment
val navController = navHostFragment.navController

override fun onSupportNavigateUp(): Boolean {
   return navController.navigateUp() || super.onSupportNavigateUp()
}
```

### View Model

- Create class:

```kotlin
class GameFragment : Fragment() {
    private val viewModel: GameViewModel by viewModels()

    ...
}
```

- Create view model:

```kotlin
class GameViewModel : ViewModel(){
    private var _score = 0
    val score: Int get() = _score

    ...
}
```

- ViewModel should store state of UI, so move state variables there.

### LiveData

- Can be mutable or immutable using `LiveData<T>` or `MutableLiveDate<T>`
- Observe LiveData in Fragment/Activity (for Fragment, add in `onViewCreated()`)

```kotlin
private val viewModel: GameViewModel by viewModels()

override fun onViewCreated(view: View, savedInstanceState: Bundle) {
    // other declarations
    ...

    viewModel.foo.observe(viewLifecycleOwner, {
        newValue -> binding.textViewFoo.text = newValue
    })
}
```

---

## Coroutines

| Type           | Description                                                                                                                                                                                |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Job            | A cancelable unit of work, such as one created with the launch() function.                                                                                                                 |
| CoroutineScope | Functions used to create new coroutines such as launch() and async() extend CoroutineScope.                                                                                                |
| Dispatcher     | Determines the thread the coroutine will use. The Main dispatcher will always run coroutines on the main thread, while dispatchers like Default, IO, or Unconfined will use other threads. |

- Example usage :

```kotlin
import kotlinx.coroutines.*

fun main() {
    repeat(3) {
        GlobalScope.launch {
            println("Hi from ${Thread.currentThread()}")
        }
    }
}
```

---

## Internet

### Retrofit / Moshi

- Use Retrofit to handle network requests. Use annotations to describe REST actions.

- Create API class as Singleton object using the `object` keyword in Kotlin. `lazy` initializes the object as soon as it is first used. Each time we call `MarsApi.retrofitService`, the caller will access the same singleton Retrofit object that implements `MarsApiService` which is created on the first access.

- We can use Moshi to parse the JSON response and convert them to Kotlin objects

#### Network Service

```kotlin
private const val BASE_URL =
   "https://android-kotlin-fun-mars-server.appspot.com"

/**
* Build the Moshi object with Kotlin adapter factory that Retrofit will be using.
*/
private val moshi = Moshi.Builder()
   .add(KotlinJsonAdapterFactory())
   .build()

/**
* The Retrofit object with the Moshi converter.
*/
private val retrofit = Retrofit.Builder()
   .addConverterFactory(MoshiConverterFactory.create(moshi))
   .baseUrl(BASE_URL)
   .build()

/**
* A public interface that exposes the [getPhotos] method
*/
interface MarsApiService {
   /**
    * Returns a [List] of [MarsPhoto] and this method can be called from a Coroutine.
    * The @GET annotation indicates that the "photos" endpoint will be requested with the GET
    * HTTP method
    */
   @GET("photos")
   suspend fun getPhotos() : List<MarsPhoto>
}

/**
* A public Api object that exposes the lazy-initialized Retrofit service
*/
object MarsApi {
    val retrofitService: MarsApiService by lazy { 
       retrofit.create(MarsApiService::class.java) 
    }
}
```

#### ViewModel

```kotlin
class OverviewViewModel : ViewModel() {

   // The internal MutableLiveData that stores the status of the most recent request
   private val _status = MutableLiveData<String>()
   private val _photos = MutableLiveData<List<MarsPhoto>>()

   // The external immutable LiveData for the request status
   val status: LiveData<String> = _status
   val photos: LiveData<List<MarsPhoto>> = _photos

   /**
    * Call getMarsPhotos() on init so we can display status immediately.
    */
   init { getMarsPhotos() }

   /**
    * Gets Mars photos information from the Mars API Retrofit service and updates the
    * [MarsPhoto] [List] [LiveData].
    */
   private fun getMarsPhotos() {
       viewModelScope.launch {
           try {
               val listResult = MarsApi.retrofitService.getPhotos()
               _status.value = "Success: Mars properties retrieved"
           } catch (e: Exception) {
               _status.value = "Failure: ${e.message}"
           }
       }
   }
}
```

#### Model

```kotlin
data class MarsPhoto(
   val id: String, 
   @Json(name = "img_src") val imgSrcUrl: String // @Json for using a different var name
)
```

### Loading Images

- Use binding adapters to load custom attributes to views.  `imageUrl` is the custom attribute. See [this link](https://github.com/google-developer-training/android-basics-kotlin-mars-photos-app/tree/main/app/src/main/java/com/example/android/marsphotos) for an example project.

```xml
<ImageView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:imageUrl="@{product.imageUrl}"/>
```

- Create `BindingAdapters.kt` to use custom attributes using **Coil**
```kotlin
@BindingAdapter("listData")
fun bindRecyclerView(recyclerView: RecyclerView, data: List<MarsPhoto>?) {
    val adapter = recyclerView.adapter as PhotoGridAdapter
    adapter.submitList(data)
}

/**
 * Uses the Coil library to load an image by URL into an [ImageView]
 */
@BindingAdapter("imageUrl")
fun bindImage(imgView: ImageView, imgUrl: String?) {
    imgUrl?.let {
        val imgUri = imgUrl.toUri().buildUpon().scheme("https").build()
        imgView.load(imgUri) {
            placeholder(R.drawable.loading_animation)
            error(R.drawable.ic_broken_image)
        }
    }
}

/**
 * This binding adapter displays the [MarsApiStatus] of the network request in an image view.  When
 * the request is loading, it displays a loading_animation.  If the request has an error, it
 * displays a broken image to reflect the connection error.  When the request is finished, it
 * hides the image view.
 */
@BindingAdapter("marsApiStatus")
fun bindStatus(statusImageView: ImageView, status: MarsApiStatus?) {
    when (status) {
        MarsApiStatus.LOADING -> {
            statusImageView.visibility = View.VISIBLE
            statusImageView.setImageResource(R.drawable.loading_animation)
        }
        MarsApiStatus.ERROR -> {
            statusImageView.visibility = View.VISIBLE
            statusImageView.setImageResource(R.drawable.ic_connection_error)
        }
        MarsApiStatus.DONE -> {
            statusImageView.visibility = View.GONE
        }
    }
}
```

---

## Views

### View Binding

- View binding allows us to retrieve all the views from a layout without using `findViewById`. Instead, we access a binding object that contains the views as properties. E.g. a `MainActivity` layout gives us access to `ActivityMainBinding`. We then use it like this:
```kotlin
class MainActivity : AppCompatActivity() {

    lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)

        // Instead of passing the resource ID of the layout, R.layout.activity_main, this specifies the root of the hierarchy of views in your app, binding.root.
        setContentView(binding.root)
    }
}
```

```kotlin
// Old way with findViewById()
val myButton: Button = findViewById(R.id.my_button)
myButton.text = "A button"

// Better way with view binding
val myButton: Button = binding.myButton
myButton.text = "A button"

// Best way with view binding and no extra variable
binding.myButton.text = "A button"
```
