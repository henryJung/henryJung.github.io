---
layout: post title: Databinding 사용 시, 경우에 따라 View 바인딩 하는 방법 tags: [android, kotlin,development]
comments: true
---

Databinding 라이브러리 사용 시, 사용되는 경우(Activity, Fragment, Adapter, CustomView...)에 따라 xml 레이아웃을 바인딩하는 방법에 차이가 있어서 간단히
정리해보았습니다.

모든 내용은, Android developers 래퍼런스의 Generated binding classes 가이드를 참고하였습니다.

### Activity

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    val binding = DataBindingUtil.setContentView(this, layoutId)
}
```

### Fragment, Adapter

```kotlin
val binding = ListItemBinding.inflate(layoutInflater, viewGroup, false)
// or 
val binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false)
```

```kotlin
//Fragment 예시 
override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
    val binding = DataBindingUtil.inflate(inflater, R.layout.fragment_main, container, false)
    //or 
    val binding = FragmentMainBinding.inflate(inflater, container, false)
    val view = binding.root
    return view
}
```

```kotlin
//Adapter 예시 
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
    //View를 넘기는 경우
    val view = LayoutInflater.from(parent.context).inflate(R.layout.item_recyclerview, parent, false)
    return ViewHolderItem(view)
    //뷰홀더에서 아래와 같이 binding 처리 필요 
    // val binding = DataBindingUtil.bind(view)

    //binding을 넘기는 경우
    val binding: ItemRecyclerviewBinding =
        DataBindingUtil.inflate(LayoutInflater.from(parent.context), R.layout.item_recyclerview, parent, false)
    //or
    val binding = DataBindingUtil.bind(view) //뷰홀더에서 처리하는 binding과 동일
    return ViewHolderItem(binding)
}
 ```

### CustomView

ViewGroup을 사용하는 경우, 즉 ViewGroup을 상속받는 CustomView의 경우는, binding.inflate(layoutInflater, viewGroup, attachToRoot) 함수를
사용합니다.

```kotlin
class HeaderView(context: Context?, attrs: AttributeSet?, defStyleAttr: Int) :
    RelativeLayout(context, attrs, defStyleAttr) {
    constructor(context: Context?) : this(context, null)
    constructor(context: Context?, attrs: AttributeSet?) : this(context, attrs, 0)

    //view_header.xml파일명을 기반으로 자동생성된 binding 객체 ViewHeaderBinding
    private val binding: ViewHeaderBinding = ViewHeaderBinding.inflate(LayoutInflater.from(context), this, false)

    init {
        TODO()
    }
}
```

여기에서 중요한 점은,

binding 변수에 inflate 하는 binding 타입이, DataBindingUtil이 아니라 xml명을 기반으로 자동 생성된 ViewDataBinding 객체를 사용하여 inflate 해준다는 점과, (
예제에서, view_header.xml파일명을 기반으로 자동 생성된 binding 객체 ViewHeaderBinding에 해당)

binding.inflate(layoutInflater, viewGroup, attachToRoot)의 파라미터 중

attachToRoot 파라미터를 true로 설정하면, root의 자식 View로 **자동**으로 추가되고 (xml파일 내에 커스텀뷰를 정의해서 사용하는 경우)

**false로 하는 경우**에는, addView(view) 를 사용해서 뷰를 추가하는 경우입니다.

또는, 이미 구성된(inflate) View를 따로 바인딩하는 경우에는 아래와 같이 할 수 있습니다.

```kotlin
val binding: ViewHeaderBinding = ViewHeaderBinding.bind(viewRoot)
```

Others 바인딩 클래스를 미리 알 수 없는 경우에는, DataBindingUtil 클래스를 사용하여 바인딩할 수 있습니다.

```kotlin
val viewRoot = LayoutInflater.from(this).inflate(layoutId, parent, attachToParent)
val binding: ViewDataBinding? = DataBindingUtil.bind(viewRoot)
```