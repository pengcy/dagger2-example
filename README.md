## Overview

This code sample shows how to use Dagger 2 with Retrofit, OkHttp, and Gson. 

For more context, see https://github.com/codepath/android_guides/wiki/Dependency-Injection-with-Dagger-2.

### Scopes
Think a scope as constraint which encapsulates components and defines the component lifecycle. ```@Singleton``` is a default scope provided out of box. This ```UserScope``` is a custom scope.

UserScope.java
```
@Documented
@Retention(value=RetentionPolicy.RUNTIME)
@Scope
public @interface UserScope {
}
```

### Modules
Think modules as things that are needed to make up a component, things that will be injected into Activities, Fragments, Presenters, ViewModels, Respositories, etc.

AppModule.java
```
@Module
public class AppModule {
    Application mApplication;
    public AppModule(Application application) {
        mApplication = application;
    }
    @Provides
    @Singleton
    Application providesApplication() {
        return mApplication;
    }
}
```

NetModule.java
```
@Module
public class NetModule {
    String mBaseUrl;
    public NetModule(String baseUrl) {
        this.mBaseUrl = baseUrl;
    }

    @Provides
    @Singleton
    SharedPreferences providesSharedPreferences(Application application) {
        return PreferenceManager.getDefaultSharedPreferences(application);
    }

    @Provides
    @Singleton
    Cache provideOkHttpCache(Application application) {
        int cacheSize = 10 * 1024 * 1024; // 10 MiB
        Cache cache = new Cache(application.getCacheDir(), cacheSize);
        return cache;
    }

    @Provides  // Dagger will only look for methods annotated with @Provides
    @Singleton
    Gson provideGson() {
        GsonBuilder gsonBuilder = new GsonBuilder();
        gsonBuilder.setFieldNamingPolicy(FieldNamingPolicy.LOWER_CASE_WITH_UNDERSCORES);
        return gsonBuilder.create();
    }

    @Provides
    @Singleton
    OkHttpClient provideOkHttpClient(Cache cache) {
        OkHttpClient okHttpClient = new OkHttpClient();
        okHttpClient.setCache(cache);
        return okHttpClient;
    }

    @Provides
    @Singleton
    Retrofit provideRetrofit(Gson gson, OkHttpClient okHttpClient) {
        Retrofit retrofit = new Retrofit.Builder()
                .addConverterFactory(GsonConverterFactory.create(gson))
                .baseUrl(mBaseUrl)
                .client(okHttpClient)
                .build();
        return retrofit;
    }
}
```

GitHubModule.java
```
@Module
public class GitHubModule {
    @Provides
    @UserScope
    public GitHubApiInterface providesGitHubInterface(Retrofit retrofit) {
        return retrofit.create(GitHubApiInterface.class);
    }
}
```

### Components
Think components as groups of things/modules that are needs for an application, a activity, a fragment, etc.

NetComponent.java
```
@Singleton
@Component(modules={AppModule.class, NetModule.class})
public interface NetComponent {
    // downstream components need these exposed
    Retrofit retrofit();
    OkHttpClient okHttpClient();
    SharedPreferences sharedPreferences();
}
```

GitHubComponent.java
```
@UserScope
@Component(dependencies = NetComponent.class, modules = GitHubModule.class)
public interface GitHubComponent {
    void inject(MainActivity activity);
}
```

### Data layer

Repository.java
```
public class Repository {
    String name;
    String fullName;
    String description;

    public String getName() {
        return name;
    }

    public String getFullName() {
        return fullName;
    }

    public String getDescription() {
        return description;
    }
}
```

GitHubApiInterface.java
```
public interface GitHubApiInterface {
    @GET("/users/{user}/repos")
    Call<ArrayList<Repository>> getRepository(@Path("user") String userName);
}
```
### Initializing the components

MyApp.java
```
public class MyApp extends Application {
    private NetComponent mNetComponent;
    private GitHubComponent mGitHubComponent;

    @Override
    public void onCreate() {
        super.onCreate();

        // specify the full namespace of the component
        // Dagger_xxxx (where xxxx = component name)
        mNetComponent = DaggerNetComponent.builder()
                .appModule(new AppModule(this))
                .netModule(new NetModule("https://api.github.com"))
                .build();

        mGitHubComponent = DaggerGitHubComponent.builder()
                .netComponent(mNetComponent)
                .gitHubModule(new GitHubModule())
                .build();

    }

    public NetComponent getNetComponent() {
        return mNetComponent;
    }

    public GitHubComponent getGitHubComponent() {
        return mGitHubComponent;
    }
}
```

### Injecting objects and use them in an Activity

```
@Inject
SharedPreferences mSharedPreferences;
@Inject
Retrofit mRetrofit;
@Inject
GitHubApiInterface mGitHubApiInterface;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ((MyApp) getApplication()).getGitHubComponent().inject(this);
}
```

MainActivity.java
```
public class MainActivity extends AppCompatActivity {

    @Inject
    SharedPreferences mSharedPreferences;

    @Inject
    Retrofit mRetrofit;

    @Inject
    GitHubApiInterface mGitHubApiInterface;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        FloatingActionButton fab = (FloatingActionButton) findViewById(R.id.fab);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(final View view) {
                    Call<ArrayList<Repository>> call = mGitHubApiInterface.getRepository("codepath");

                    call.enqueue(new Callback<ArrayList<Repository>>() {
                        @Override
                        public void onResponse(Response<ArrayList<Repository>> response, Retrofit retrofit) {
                            if (response.isSuccess()) {
                                Log.i("DEBUG", response.body().toString());
                                Snackbar.make(view,"Data retrieved", Snackbar.LENGTH_LONG)
                                        .setAction("Action",null).show();
                            } else {
                                Log.i("ERROR", String.valueOf(response.code()));
                            }

                        }

                        @Override
                        public void onFailure(Throwable t) {

                        }
                    });
                }

            });

        ((MyApp) getApplication()).getGitHubComponent().inject(this);
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(R.menu.menu_main, menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        // Handle action bar item clicks here. The action bar will
        // automatically handle clicks on the Home/Up button, so long
        // as you specify a parent activity in AndroidManifest.xml.
        int id = item.getItemId();

        //noinspection SimplifiableIfStatement
        if (id == R.id.action_settings) {
            return true;
        }

        return super.onOptionsItemSelected(item);
    }
}
```
