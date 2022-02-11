---
title: Android ViewBinding 사용
subtitle: Activity, Fragment, Dialog, View 및 RecyclerView Holder에서 View Binding을 한 번 사용하는 방법을 설명하려고 합니다.
gh-repo: henryJung/CleanMvvm
thumbnail-img: /assets/img/viewbinding/viewbinding_thubmnail.jpeg
tags: [android, aac, development]
---

## 개요
**viewBinding**은 뷰와 상호 작용하는 코드를 보다 쉽게 작성할 수 있는 기능입니다. 모듈에서 **viewBinding**이 활성화되면 해당 모듈에 있는 각 XML 레이아웃 파일에 대한 바인딩 클래스를 생성합니다. 바인딩 클래스의 인스턴스에는 해당 레이아웃에 ID가 있는 모든 보기에 대한 직접 참조가 포함되어 있습니다.

대부분의 경우 보기 바인딩이 `findViewById`를 대체합니다.

**viewBinding**을 사용할 때 일반적으로 각 레이아웃 파일에 대해 바인딩 클래스가 생성됩니다. 바인딩 클래스는 특정 뷰에 대한 모든 참조를 저장합니다.

**viewBinding**은 null-safe하고 빠릅니다. 이를 통해 개발자는 프로그래밍 중에 일반적인 오류를 피할 수 있습니다.

![viewbinding](/assets/img/viewbinding/viewbinding_thubmnail.jpeg){: .mx-auto.d-block :}

### 뷰 바인딩의 장점
- null 안전을 지원합니다. 이 기능은 개발자가 존재하지 않는 뷰 또는 ID를 호출하는 것을 방지합니다. 결과적으로 앱이 갑자기 충돌하는 것을 방지합니다.
- 상용구 코드를 줄이는 데 도움이 됩니다.
- type 안전을 용이하게 합니다. 생성된 바인딩 클래스는 레이아웃 파일에 선언된 뷰와 일치합니다. 다시 한 번, 이 기능은 응용 프로그램이 충돌하는 것을 방지합니다.

## 설정
**viewBinding은 모듈별로 활성화**됩니다. 모듈에서 보기 바인딩을 활성화하려면 다음 예제와 같이 모듈 수준 `build.gradle` 파일에서 viewBinding 빌드 옵션을 true로 설정합니다.

```groovy
android {
    ...
    buildFeatures {
        viewBinding = true
    }
}
```

바인딩 클래스를 생성하는 동안 레이아웃 파일을 무시하려면 해당 레이아웃 파일의 루트 보기에 `tools:viewBindingIgnore="true"` 속성을 추가합니다.

```xml
<ConstraintLayout
        ...
        tools:viewBindingIgnore="true" >
    ...
</ConstraintLayout>
```

프로젝트에 대해 활성화되면 뷰 바인딩은 모든 레이아웃에 대한 바인딩 클래스를 자동으로 생성합니다. 바인딩 클래스의 이름은 XML 파일의 이름을 Pascal 대소문자로 변환하고 끝에 **"Binding"**이라는 단어를 추가하여 생성합니다.

이 작업이 완료되면 `Fragment`, A`ctivity`, `Dialog`, `View` 및 `RecyclerViewHolder`와 같은 레이아웃을 확장할 때마다 바인딩 클래스를 사용할 수 있습니다.

## ViewBinding을 위한 확장 메서드
ViewBinding을 사용하기 위한 확장 메서드가 있습니다. 단계별로 설명하려고 합니다. 전체 코드는 아래와 같습니다.

### ViewBindingExtensions.kt
```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.viewbinding.ViewBinding
import java.lang.reflect.ParameterizedType
internal fun <V : ViewBinding> Class<*>.getBinding(layoutInflater: LayoutInflater): V {
    return try {
        @Suppress("UNCHECKED_CAST")
        getMethod(
            "inflate",
            LayoutInflater::class.java
        ).invoke(null, layoutInflater) as V
    } catch (ex: Exception) {
        throw RuntimeException("The ViewBinding inflate function has been changed.", ex)
    }
}
internal fun <V : ViewBinding> Class<*>.getBinding(
    layoutInflater: LayoutInflater,
    container: ViewGroup?
): V {
    return try {
        @Suppress("UNCHECKED_CAST")
        getMethod(
            "inflate",
            LayoutInflater::class.java,
            ViewGroup::class.java,
            Boolean::class.java
        ).invoke(null, layoutInflater, container, false) as V
    } catch (ex: Exception) {
        throw RuntimeException("The ViewBinding inflate function has been changed.", ex)
    }
}
internal fun Class<*>.checkMethod(): Boolean {
    return try {
        getMethod(
            "inflate",
            LayoutInflater::class.java
        )
        true
    } catch (ex: Exception) {
        false
    }
}
internal fun Any.findClass(): Class<*> {
    var javaClass: Class<*> = this.javaClass
    var result: Class<*>? = null
    while (result == null || !result.checkMethod()) {
        result = (javaClass.genericSuperclass as? ParameterizedType)
            ?.actualTypeArguments?.firstOrNull {
                if (it is Class<*>) {
                    it.checkMethod()
                } else {
                    false
                }
            } as? Class<*>
        javaClass = javaClass.superclass
    }
    return result
}inline fun <reified V : ViewBinding> ViewGroup.toBinding(): V {
    return V::class.java.getMethod(
        "inflate",
        LayoutInflater::class.java,
        ViewGroup::class.java,
        Boolean::class.java
    ).invoke(null, LayoutInflater.from(context), this, false) as V
}

********************************************************************

internal fun <V : ViewBinding> BindingActivity<V>.getBinding(): V {
    return findClass().getBinding(layoutInflater)
}
internal fun <V : ViewBinding> BindingFragment<V>.getBinding(
    inflater: LayoutInflater,
    container: ViewGroup?
): V {
    return findClass().getBinding(inflater, container)
}
internal fun <V : ViewBinding> BindingSheetDialog<V>.getBinding(
    inflater: LayoutInflater,
    container: ViewGroup?
): V {
    return findClass().getBinding(inflater, container)
}
internal fun <V : ViewBinding> BindingComponent<V>.getBinding(
    inflater: LayoutInflater,
    container: ViewGroup?
): V {
    return findClass().getBinding(inflater, container)
}
```

ViewHolder에서 ViewBinding 사용
RecyclerView 어댑터와 함께 사용할 바인딩 클래스의 인스턴스를 설정하려면 생성된 바인딩 클래스 개체를 홀더 클래스 생성자에 전달해야 합니다.

`RecyclerView` 행 항목에 대한 `row_character` XML 파일이 있고 생성된 클래스는 RowCharacterBinding 입니다.

### row_character.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<com.google.android.material.card.MaterialCardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/card"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:cardElevation="4dp"
    app:cardUseCompatPadding="false">   
    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">     
        <com.google.android.material.textview.MaterialTextView
            android:id="@+id/tvName"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginStart="8dp"
            android:layout_marginTop="16dp"
            android:layout_marginEnd="8dp"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            tools:text="Mr.Sanchez" />   
    </androidx.constraintlayout.widget.ConstraintLayout>
</com.google.android.material.card.MaterialCardView>
```
우리는 ViewHolder 클래스를 확장하는 `BindingViewHolder` 클래스를 가지고 있습니다.

### BindingViewHolder.kt
```kotlin
import android.content.Context
import androidx.recyclerview.widget.RecyclerView
import androidx.viewbinding.ViewBinding

open class BindingViewHolder<VB : ViewBinding>(val binding: VB) :
    RecyclerView.ViewHolder(binding.root) {
    val context: Context = binding.root.context
}
```

`CharactersViewHolder` 클래스의 사용 예는 다음과 같습니다.

### CharactersViewHolder.kt
```kotlin
inner class CharactersViewHolder(binding: RowCharacterBinding) :
    BindingViewHolder<RowCharacterBinding>(binding) {
    fun bind(item: Character) {
        ....
        binding.tvName.text = item.name
        ....
        binding.root.setOnClickListener {
            ....
        }
    }
}
```

`ViewBindingExtensions.kt` 파일에서 **"toBinding()"** 메서드를 사용합니다.
```kotlin
inline fun <reified V : ViewBinding> ViewGroup.toBinding(): V {
    return V::class.java.getMethod(
        "inflate",
        LayoutInflater::class.java,
        ViewGroup::class.java,
        Boolean::class.java
    ).invoke(null, LayoutInflater.from(context), this, false) as V
}
```

마지막으로 `onCreateViewHolder`를 사용하여 생성된 바인딩 클래스를 `ViewHolder` 클래스에 **"parent.toBinding()"**을 전달합니다.
```kotlin
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
    return CharactersViewHolder(parent.toBinding())
}
```

## Activity에서 ViewBinding 사용
Activity와 함께 사용할 바인딩 클래스의 인스턴스를 설정하려면 BindingActivity에서 다음 단계를 수행합니다.

`BindingActivity()` 클래스는 `AppCompatActivity()` 클래스를 상속받고 있습니다.

```kotlin
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import androidx.viewbinding.ViewBinding
open class BindingActivity<VB : ViewBinding> : AppCompatActivity() {
     .......
}
```

`ViewBindingExtensions.kt` 파일에서 **"getBinding()"** 메소드를 생성하여 사용합니다.

```kotlin
internal fun <V : ViewBinding> BindingActivity<V>.getBinding(): V {
    return findClass().getBinding(layoutInflater)
}
```
`BindingActivity.kt` 파일의 setContentView()에 **"getBinding()"**을 전달합니다.

### BindingActivity.kt

```kotlin
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import androidx.viewbinding.ViewBinding
open class BindingActivity<VB : ViewBinding> : AppCompatActivity() {
    lateinit var binding: VB
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        if (::binding.isInitialized.not()) {
            binding = getBinding()
            setContentView(binding.root)
        }
    }
}
```

이 작업이 완료되면 이제 바인딩 클래스의 인스턴스를 사용하여 `BindingActivity()` 에서 확장된 뷰를 참조하고 활용할 수 있습니다.

### MainActivity.kt
```kotlin
class MainActivity : BindingActivity<ActivityMainBinding>() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)        ......        
        binding.tvName.text = "NAME"
    }
}
```

## Fragment에서 ViewBinding 사용
Fragment와 함께 사용할 바인딩 클래스의 인스턴스를 설정하려면 BindingFragment에서 다음 단계를 수행하십시오.

`BindingFragment()` 클래스는 `Fragment()` 클래스를 상속받습니다.

```kotlin
open class BindingFragment<VB : ViewBinding> : Fragment() {
    .....
}
```

`ViewBindingExtensions.kt` 파일에서 **"getBinding()"** 메소드를 생성하여 사용합니다.

```kotlin
internal fun <V : ViewBinding> BindingFragment<V>.getBinding(
    inflater: LayoutInflater,
    container: ViewGroup?
): V {
    return findClass().getBinding(inflater, container)
}
```

`BindingFragment.kt` 파일의 onCreateView()에 **"getBinding()"**을 전달합니다.

### BindingFragment.kt
```kotlin
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.fragment.app.Fragment
import androidx.viewbinding.ViewBinding
open class BindingFragment<VB : ViewBinding> : Fragment() {
    private var _binding: VB? = null
    val binding: VB
        get() = _binding
            ?: throw RuntimeException("Should only use binding after onCreateView and before onDestroyView")
    protected fun requireBinding(): VB = requireNotNull(_binding)
    final override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = getBinding(inflater, container)
        return binding.root
    }
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

이 작업이 완료되면 이제 바인딩 클래스의 인스턴스를 사용하여 `BindingFragment()` 에서 확장된 뷰를 참조하고 활용할 수 있습니다.

### CharactersFragment.kt
```kotlin
class CharactersFragment : BindingFragment<FragmentCharactersBinding>() {    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)        ......        
        binding.tvName.text = "NAME"
    }
}
```

## Dialog에서 ViewBinding 사용

`BottomSheetDialogFragment` 와 함께 사용할 바인딩 클래스의 인스턴스를 설정하려면 `BindingSheetDialog`에서 다음 단계를 수행하십시오.

`BindingSheetDialog()` 클래스는 `BottomSheetDialogFragment()` 클래스를 상속받습니다.

```kotlin
open class BindingSheetDialog<VB : ViewBinding> : BottomSheetDialogFragment() {
    .......
}
```

`ViewBindingExtensions.kt` 파일에서 **"getBinding()"** 메소드를 생성하여 사용합니다.

```kotlin
internal fun <V : ViewBinding> BindingSheetDialog<V>.getBinding(
    inflater: LayoutInflater,
    container: ViewGroup?
): V {
    return findClass().getBinding(inflater, container)
}
```

`BindingSheetDialog.kt` 파일에서 **"getBinding()"** onCreateView()를 전달합니다.

```kotlin
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.viewbinding.ViewBinding
import com.google.android.material.bottomsheet.BottomSheetDialogFragment
open class BindingSheetDialog<VB : ViewBinding> : BottomSheetDialogFragment() {
    private var _binding: VB? = null
    protected val binding: VB
        get() = _binding
            ?: throw RuntimeException("Should only use binding after onCreateView and before onDestroyView")
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = getBinding(inflater, container)
        return binding.root
    }
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

## View/Layout에서 ViewBinding 사용
레이아웃과 함께 사용할 바인딩 클래스의 인스턴스를 설정하려면 레이아웃에서 다음 단계를 수행하십시오.

`BindingComponent()` 클래스는 `ConstraintLayout()` 클래스를 상속받고 있습니다.

```kotlin
open class BindingComponent<VB : ViewBinding> @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : ConstraintLayout(context, attrs) {
    .....
}
```

`ViewBindingExtensions.kt` 파일에서 **"getBinding()"** 메소드를 생성하여 사용합니다.
```kotlin
internal fun <V : ViewBinding> BindingComponent<V>.getBinding(
    inflater: LayoutInflater,
    container: ViewGroup?
): V {
    return findClass().getBinding(inflater, container)
}
```

`BindingComponent.kt` 파일에 **"getBinding()"**을 전달합니다.
```kotlin
val Context.inflater get() = LayoutInflater.from(this)

********************************************************************

open class BindingComponent<VB : ViewBinding> @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : ConstraintLayout(context, attrs) {
    lateinit var binding: VB
    init {
        initBinding()
    }
    private fun initBinding() {
        if (::binding.isInitialized.not()) {
            binding = getBinding(context.inflater, container = this)
        }
    }
}
```

## findViewById와의 차이점

**ViewBinding**은 `findViewById`를 사용하는 것보다 중요한 이점이 있습니다.

- **Null safety**: ViewBinding은 뷰에 대한 직접 참조를 생성하므로 잘못된 뷰 ID로 인한 **NullPointException**의 위험이 없습니다. 또한 View가 레이아웃의 일부 구성에만 있는 경우 바인딩 클래스에서 해당 참조를 포함하는 필드는 @Nullable로 표시됩니다.

- **Type safety**: 각 바인딩 클래스의 필드에는 XML 파일에서 참조하는 View와 일치하는 유형이 있습니다. 이것은 클래스 캐스트 예외의 위험이 없음을 의미합니다.

이러한 차이점은 레이아웃과 코드 간의 비호환성으로 인해 런타임이 아닌 **컴파일 시간**에 빌드가 실패한다는 것을 의미합니다.

### DataBinding과의 비교
ViewBinding과 DataBinding은 모두 View를 직접 참조하는 데 사용할 수 있는 바인딩 클래스를 생성합니다. 그러나 ViewBinding은 더 간단한 사용 사례를 처리하기 위한 것이며 DataBinding에 비해 다음과 같은 이점을 제공합니다.

- **더 빠른 컴파일**: ViewBinding에는 주석 처리가 필요하지 않으므로 컴파일 시간이 더 빠릅니다.
- **사용 용이성**: ViewBinding에는 특별히 태그가 지정된 XML 레이아웃 파일이 필요하지 않으므로 앱에서 더 빠르게 채택할 수 있습니다. 모듈에서 ViewBinding을 활성화하면 해당 모듈의 모든 레이아웃에 자동으로 적용됩니다.

반대로 ViewBinding에는 DataBinding에 비해 다음과 같은 제한 사항이 있습니다.
- ViewBinding은 레이아웃 변수나 레이아웃 표현식을 지원하지 않으므로 XML 레이아웃 파일에서 직접 동적 UI 콘텐츠를 선언하는 데 사용할 수 없습니다.
- ViewBinding은 양방향 데이터 바인딩을 지원하지 않습니다. 

이러한 고려 사항 때문에 어떤 경우에는 프로젝트에서 ViewBinding과 DataBinding을 모두 사용하는 것이 가장 좋습니다. 고급 기능이 필요한 레이아웃에서는 데이터 바인딩을 사용하고 필요하지 않은 레이아웃에서는 ViewBinding을 사용할 수 있습니다.


[원본 글](https://medium.com/innovance-company-blog/android-viewbinding-using-in-activity-fragment-dialog-view-recyclerviewholder-756b6b5c15cb)
