# RxAndroid

生命週期的掌控：

Observable 應用在 Android 上，常遇到 Activity/Fragment 不在前景，卻存取 View 造成的問題。

當 Activty/Fragment 生命週期結束或者指定特定生命期間結束時，自動 `unsubscribe()`。

```java
AppObservable.bindActivity()
AppObservable.bindFragment()

LifecycleObservable.bindActivityLifecycle()
LifecycleObservable.bindFragmentLifecycle()
```

`AppObservable.bindActivity()`/`AppObservable.bindFragment()` 目前只能作到 `observeOn(AndroidSchedulers.mainThread())` 以及杜絕不應該的期間作 `subscribe()`。

如果要生命週期結束，把一些 subscriptions 取消：

自己 `unsubscribe()`: 

```java
class SimpleActivity extends Activity {
    CompsotionSubscription mSubscriptions = new CompositeSubscription();
    
    @Override
    protected void onResume() {
        super.onResume();

        bind(Observable.just("Hello, world"), s -> textView.setText(s));
    }

    protected <T> void bind(Observable<T> obs, Action1<T> onNext) {
        mCompositeSubscription.add(AppObservable.bindActivity(this, obs).subscribe(onNext));
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

        mCompositeSubscription.unsubscribe();
    }
}
```

`LifecycleObservable` + `RxActivity`:

```java
class SimpleActivity extends RxActivity {

    @Override
    public void onResume() {
        LifecycleObservable.bindActivityLifecycle(lifecycle(),
            AppObservable.bindActivity(this, Observable.just("Hello, world")))
        ).subscribe(s -> textView.setText(s));
    }
}
```

`LifecycleObservable` 好處是會在對應的生命期間取消。

`LifecycleObservable` 哪時候訂閱哪時候取消對照表：

```java
CREATE -> LifecycleEvent.DESTROY;
START -> LifecycleEvent.STOP;
RESUME -> LifecycleEvent.PAUSE;
PAUSE -> LifecycleEvent.STOP;
STOP -> LifecycleEvent.DESTROY;
```

*註：筆者不是很清楚，為什麼不用 overloading: `AppObservable.bind(Activity/Frgment/v4.Fragment)` 來取代 `AppObservable.bindFragment(Fragment)`,
`AppObservable.bindFragment(v4.Fragment)`,
`AppObservable.bindActivity(Activity)`*

## RxBinding

主要以 RxView 為中心 ，當 View 顯示時 `subscribe()` 離開時 `unsubscribe()`

Before：

```java
class HogeActivity extends Activity {

    @InjectView(R.id.text)
    private TextView mTextView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.inject(this);

        AppObservable.bindActivity(this, Observable.just("hoge"))
        .subscribeOn(Schedulers.io())
        .subscribe(setTextAction());
    }

    Action1<String> setTextAction() {
        return text -> mTextView.setText(text);
    }
}
```

After:

```java
class HogeActivity extends Activity {
    // ...

    private Rx<TextView> mRxTextView

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.inject(this);

        mRxTextView = RxView.of(mTextView);
        Subscription s = mRxTextView.bind(Observable.just("hoge"), RxActions.setText());
    }
}
```