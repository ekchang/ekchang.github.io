---
layout: post
title: Data Binding - First party solution to findViewById
date: 2016-05-22
categories: blog
---

There are plenty of resources introducing the data binding library in Android--see [Yigit's talk at the Bay Area Android Dev Group][realm-yigit] or watch [Jacob's talk from Droidcon NYC 2015][droidcon-jacob]. Not many people are willing to buy into the whole MVVM architecture, especially if you're already working on a large app. It doesn't seem worth it to rip out the internals of your carefully constructed MVX setup to make room for this framework. 

It probably isn't.

But I still think there's room to add the data binding library to every app. It works really nicely as a replacement for `findViewById` and [Butterknife][butterknife] (sorry Jake).

### The problem with findViewById

```java
class ExampleActivity extends Activity {
  TextView title;
  TextView subtitle;
  TextView footer;

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    title = (TextView) findViewById(R.id.title);
    subtitle = (TextView) findViewById(R.id.subtitle);
    footer = (TextView) findViewById(R.id.footer);
  }
}
```

Everyone is familiar with this common nuisance. Whenever you inflate an Activity or Fragment, you need to call `findViewById(int id)`, cast it to the correct View you're interested in, and if your view has the slightest bit of complexity involved, you need to hold a reference to the view in your class. This scales poorly, is prone to error, and adds extra clutter to your `onCreate` methods.

[Butterknife][butterknife] aimed to solve this problem a while back:

```java
class ExampleActivity extends Activity {
  @BindView(R.id.title) TextView title;
  @BindView(R.id.subtitle) TextView subtitle;
  @BindView(R.id.footer) TextView footer;

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
    // TODO Use fields...
  }
}
```

It solves the automatic type casting, but we still have to hold each field manually. We clutter our Activity classes with these view references, turning our Activity into a big ViewHolder.

Just introducing the data binding library can remove this clutter for you:

```java
class ExampleActivity extends Activity {
  SimpleActivityBinding binding;

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    binding = DataBindingUtil.setContentView(this, R.layout.simple_activity);
    // TODO access any View with ids declared in the layout
    binding.title.setText("Hi I'm a TextView!");
    binding.subtitle.setText("I'm a subtitle.");
    binding.footer.setText("I'm a footer.");
  }
}
```

The generated `ViewDataBinding` object acts as the ViewHolder for you, doing automatic type casting for you. No need to jump into the MVVM aspect of it--the binding is incredibly invaluable as a ViewHolder in itself. All you need to do for this migration is to wrap your root layout in `<layout>` tags and move all the namespace definitions.

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout 
  xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:tools="http://schemas.android.com/tools"
  xmlns:app="http://schemas.android.com/apk/res-auto"
  >
  <LinearLayout
      android:id="@+id/container"
      android:layout_width="match_parent"
      android:layout_height="match_parent"
      >
    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        />
    <TextView
        android:id="@+id/subtitle"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        />
    <TextView
        android:id="@+id/footer"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        />
  </LinearLayout>
</layout>
```

Every view with an id defined will be captured in the generated `ViewDataBinding` for you, with no additional casting or declaration necessary. Adding another view to this layout with an id defined will be immediately accessible in the `binding` field in the example above. It really is a first party solution to `findViewById`.

[realm-yigit]: https://realm.io/news/data-binding-android-boyar-mount/
[droidcon-jacob]: https://youtu.be/WdUbXWztKNY
[butterknife]: https://github.com/jakewharton/butterknife
