---
layout: post
title: Android Multi ViewType RecyclerView
subtitle: RecyclerView의 itemView를 model의 viewType을 가져와 그려주도록 합니다.
gh-repo: henryJung/CleanMvvm
tags: [android, recyclerView, development]
comments: true
---

RecyclerView에 대한 itemView를 표현 할 때, 여러가지 타입의 뷰를 그려주고 싶어 할 때가 있습니다. RecyclerView는 Adapter가 관리하는 Data Set의 특정 데이터 항목에 대하여 미리 정의된 View를 통해 스크롤이 있는 List 형식으로 표현할 수 있습니다. 이번 포스팅에서는 한 개의 RecyclerView에서 여러 Type의 View를 정의해놓고 데이터의 타입에 따라 각각 다른 ViewType을 적용시키는 방법을 알아보겠습니다. 

여기에서는 간단하게 Photo와 Book 두가지 타입에 구성되어 있는 List를 표현하겠습니다.

---
#### 1. ListItem.kt
```kotlin
interface ListItem {
    val viewType: ViewType

    fun getKey() = hashCode()
}

// List에 표현 할 viewType을 설정합니다.
enum class ViewType {
    EMPTY,
    PHOTO,
    RANDOM_PHOTO,
    BOOK,
}
```
List에 표현 할 Item은 ListItem을 반드시 참조해야합니다.

#### 2. model(Photo, Book) 
```kotlin
data class Photo(
    val id: String,
    val author: String,
    val width: Int,
    val height: Int,
    val url: String,
    val downloadUrl: String,
): ListItem {
    override val viewType: ViewType
        get() = ViewType.PHOTO
}

data class Book(
    val title: String,
    val subtitle: String,
    val isbn13: String,
    val price: String,
    val image: String,
    val url: String
) : ListItem {
    override val viewType: ViewType
        get() = ViewType.BOOK
}
```
각 data class는 `ListItem`을 참조하고 미리 정의한 viewType을 지정합니다.

#### 3. BaseViewHolder.kt
```kotlin
abstract class BaseViewHolder(
    private val binding: ViewDataBinding,
    private val handler: BaseRecyclerHandler? = null
) : RecyclerView.ViewHolder(binding.root) {

    protected var item: ListItem? = null

    open fun bind(item: ListItem) {
        this.item = item
        binding.setVariable(BR.item, this.item)
        binding.setVariable(BR.handler, handler)
    }
}
```
`BaseViewHolder`는 ListAdapter에 들어갈 기본 뷰홀더가 됩니다. `BaseViewHolder`에서는 기본적으로 item, handler에 대한 binding을 담고 있습니다.

#### 4. ClickType.kt
```kotlin
enum class ClickType {
    Random, Photo, Book
}
```
`ClickType`은 정의한 ItemView에 대한 공통 click event를 처리하기 위한 Type을 지정합니다.  

#### 5. BaseRecyclerHandler.kt
```kotlin
open class BaseRecyclerHandler(private val context: Context) {

    open fun onClickButton(type: ClickType, item: ListItem? = null) {
        when (type) {
            ClickType.Random -> {

            }
            ClickType.Photo -> {
                context.toast("Photo Click")
            }
            ClickType.Book -> {
                context.toast("Book Click")
            }
        }
    }
}
```
handler를 통한 click event를 해당 class를 통해서 처리하게 됩니다.

#### 6. ViewHolderGenerator.kt
```kotlin
object ViewHolderGenerator {

    fun get(
        parent: ViewGroup,
        viewType: Int,
        adapter: RecyclerView.Adapter<BaseViewHolder>,
        handler: BaseRecyclerHandler
    ): BaseViewHolder {
        return when (viewType) {
            ViewType.PHOTO.ordinal -> {
                PhotoViewHolder(parent.inflate(R.layout.item_photo), handler)
            }
            ViewType.RANDOM_PHOTO.ordinal -> {
                RandomPhotoViewHolder(parent.inflate(R.layout.item_photo_random), handler)
            }
            ViewType.BOOK.ordinal -> {
                BookViewHolder(parent.inflate(R.layout.item_book), handler)
            }
            ViewType.EMPTY.ordinal -> {
                EmptyFooterViewHolder(parent.inflate(R.layout.item_empty_footer), handler)
            }
            else -> {
                ItemViewHolder(parent.inflate(R.layout.item_empty))
            }
        }
    }

    class ItemViewHolder(binding: ItemEmptyBinding) : BaseViewHolder(binding)

    fun <T : ViewDataBinding> ViewGroup.inflate(layout: Int): T {
        return DataBindingUtil.inflate(
            LayoutInflater.from(context),
            layout,
            this,
            false
        )
    }
}
```
`ViewHolderGenerator`에서는 앞에서 정의한 ViewType에 대한 ViewHolder를 정의하는 class입니다.

#### 7. BaseListAdapter.kt
```kotlin
class BaseListAdapter(
    context: Context,
    private val handler: BaseRecyclerHandler = BaseRecyclerHandler(context)
) : ListAdapter<ListItem, BaseViewHolder>(DiffCallback()) {

    override fun getItemViewType(position: Int): Int {
        val item = getItem(position)
        return item?.viewType?.ordinal ?: -1
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): BaseViewHolder {
        return ViewHolderGenerator.get(parent, viewType, this, handler)
    }

    override fun onBindViewHolder(holder: BaseViewHolder, position: Int) {
        val item = getItem(position)
        if (item != null) {
            holder.bind(item)
        }
    }

    class DiffCallback<T : ListItem> : DiffUtil.ItemCallback<T>() {
        override fun areItemsTheSame(oldItem: T, newItem: T): Boolean =
            oldItem == newItem

        override fun areContentsTheSame(oldItem: T, newItem: T): Boolean =
            oldItem.viewType == newItem.viewType && oldItem.hashCode() == newItem.hashCode()
    }
}
```
`BaseListAdapter`에서는 ListItem을 담기 위한 adapter입니다.

#### 8. ViewHolder
```kotlin
class PhotoViewHolder(
    binding: ItemPhotoBinding,
    handler: BaseRecyclerHandler
) : BaseViewHolder(binding, handler)

class BookViewHolder(
    binding: ItemBookBinding,
    handler: BaseRecyclerHandler
) : BaseViewHolder(binding, handler)
```
각 ViewHolder class를 정의합니다.

#### 9. item xml을 정의합니다.
item_photo.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

        <variable
            name="item"
            type="com.devpub.domain.model.Photo" />

        <variable
            name="handler"
            type="com.devpub.cleanmvvm.ui.common.list.BaseRecyclerHandler" />

        <import type="com.devpub.cleanmvvm.ui.common.list.viewholder.ClickType"/>
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="250dp"
        android:foreground="?attr/selectableItemBackground"
        android:onClick="@{() -> handler.onClickButton(ClickType.Photo, item)}"
        tools:ignore="UnusedAttribute">

        <ImageView
            android:id="@+id/imageView"
            android:layout_width="0dp"
            android:layout_height="0dp"
            android:scaleType="centerCrop"
            android:transitionName="image"
            app:imageUrl="@{item.downloadUrl}"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

        <View
            android:layout_width="0dp"
            android:layout_height="0dp"
            android:background="@drawable/gradient"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

        <TextView
            android:id="@+id/linkTextView"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginStart="8dp"
            android:layout_marginEnd="8dp"
            android:layout_marginBottom="8dp"
            android:autoLink="web"
            android:background="?attr/selectableItemBackground"
            android:text="@{item.url}"
            android:textColorLink="@color/white"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            tools:text="http://www.naver.com" />

        <TextView
            android:id="@+id/authorTextView"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginStart="8dp"
            android:layout_marginEnd="8dp"
            android:layout_marginBottom="4dp"
            android:text="@{item.author}"
            android:textColor="@color/white"
            app:layout_constraintBottom_toTopOf="@+id/linkTextView"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            tools:text="title" />

        <TextView
            android:id="@+id/imageTextView"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginStart="8dp"
            android:layout_marginEnd="8dp"
            android:layout_marginTop="4dp"
            android:text="@{item.url}"
            android:textColor="@color/text_gray"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            tools:text="title" />

    </androidx.constraintlayout.widget.ConstraintLayout>

</layout>
```
item_book.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name="item"
            type="com.devpub.domain.model.Book" />

        <variable
            name="handler"
            type="com.devpub.cleanmvvm.ui.common.list.BaseRecyclerHandler" />

        <import type="com.devpub.cleanmvvm.ui.common.list.viewholder.ClickType"/>
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="?attr/selectableItemBackground"
        android:paddingStart="8dp"
        android:onClick="@{() -> handler.onClickButton(ClickType.Book, item)}"
        android:paddingEnd="8dp"
        android:paddingBottom="8dp">

        <ImageView
            android:id="@+id/imageView"
            android:layout_width="70dp"
            android:layout_height="120dp"
            android:scaleType="centerCrop"
            android:transitionName="image"
            app:imageUrl="@{item.image}"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

        <ImageView
            android:id="@+id/arrowImageView"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:scaleType="centerCrop"
            android:src="@drawable/ic_arrow_right"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

        <TextView
            android:id="@+id/idTextView"
            style="@style/text"
            android:layout_width="0dp"
            android:layout_marginStart="16dp"
            android:layout_marginTop="8dp"
            android:layout_marginEnd="8dp"
            android:text="@{item.isbn13}"
            android:textColor="@color/text_gray"
            app:layout_constraintEnd_toStartOf="@+id/arrowImageView"
            app:layout_constraintStart_toEndOf="@+id/imageView"
            app:layout_constraintTop_toTopOf="parent"
            tools:text="id" />

        <TextView
            android:id="@+id/titleTextView"
            style="@style/text.title"
            android:layout_width="0dp"
            android:text="@{item.title}"
            app:layout_constraintEnd_toEndOf="@+id/idTextView"
            app:layout_constraintStart_toStartOf="@+id/idTextView"
            app:layout_constraintTop_toBottomOf="@+id/idTextView"
            tools:text="title" />

        <TextView
            android:id="@+id/subTitleTextView"
            style="@style/text.light"
            android:layout_width="0dp"
            android:layout_marginTop="4dp"
            android:text="@{item.subtitle}"
            app:layout_constraintEnd_toEndOf="@+id/titleTextView"
            app:layout_constraintStart_toStartOf="@+id/titleTextView"
            app:layout_constraintTop_toBottomOf="@+id/titleTextView"
            tools:text="subTitle" />

        <TextView
            android:id="@+id/priceTextView"
            style="@style/text.title"
            android:layout_width="0dp"
            android:layout_marginTop="8dp"
            android:layout_marginEnd="8dp"
            android:text="@{item.price}"
            android:textStyle="bold"
            app:layout_constraintEnd_toEndOf="@+id/titleTextView"
            app:layout_constraintStart_toStartOf="@+id/titleTextView"
            app:layout_constraintTop_toBottomOf="@+id/subTitleTextView"
            tools:text="$3.3" />

        <TextView
            android:id="@+id/linkTextView"
            style="@style/text"
            android:layout_marginTop="4dp"
            android:autoLink="web"
            android:background="?attr/selectableItemBackground"
            android:text="@{item.url}"
            android:textColorLink="@color/primaryColor"
            app:layout_constraintStart_toStartOf="@+id/titleTextView"
            app:layout_constraintTop_toBottomOf="@+id/priceTextView"
            tools:text="http://www.naver.com" />
    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```

#### 10. ViewBindingAdapter.kt
```kotlin
object ViewBindingAdapter {

    @JvmStatic
    @BindingAdapter("show")
    fun ProgressBar.bindShow(uiState: UiState) {
        isVisible = uiState is UiState.Loading
    }

    @JvmStatic
    @BindingAdapter("toast")
    fun View.bindToast(uiState: UiState) {
        if (uiState is UiState.Error) {
            uiState.error?.message?.let { errorMessage ->
                Toast.makeText(context, errorMessage, Toast.LENGTH_SHORT).show()
            }
        }
    }

    @JvmStatic
    @BindingAdapter("adapter", "listItems")
    fun RecyclerView.bindListItems(adapter: PagingDataAdapter<*,*>, uiState: UiState) {
        this.adapter = adapter
        if (adapter is BaseListAdapter && uiState is UiState.Success<*>) {
            adapter.submitData(
                uiState.data as List<ListItem>
            )
        }
    }
}

@BindingAdapter("imageUrl")
fun setImageUrl(imageView: ImageView, url: String) {
    imageView.load(url) {
        crossfade(true)
        placeholder(R.color.hintTextColor)
    }
}

@BindingAdapter("visible")
fun setVisible(view: View, visible: Boolean) {
    view.isVisible = visible
}
```
`BindingAdapter`를 정의합니다. 이 곳에서는 adpater 초기화 및 submitData를 정의합니다.

이렇게 되면 RecyclerView에 표현 될 UI는 다음과 같습니다.
![multi viewType_result](/assets/img/multiviewtype/viewtype_result.png){: .mx-auto.d-block :}