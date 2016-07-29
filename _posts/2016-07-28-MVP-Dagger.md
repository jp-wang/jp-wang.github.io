---
layout: post
title: Common framework for MVP + Dagger
published: true
---

(by [@jpwang](https://github.com/jp-wang) and original post is [here](https://github.com/jp-wang/jp-wang.github.io/blob/master/_posts/2016-07-28-MVP-Dagger.md))

### Dagger

**Dagger** is a fully static, compile-time dependency injection framework for both Java and Android.

**Dagger** aims to address many of the development and performance issues that have plagued reflection-based solutions. For more information, please refer to [here](http://google.github.io/dagger/).We won't talk too much here.

### MVP

**MVP**(Model-View-Presenter) pattern is a derivative from the well known `MVC`(Model-View-Controller) which for a while now is gaining importance in the development of Android applications.

#### What does MVP really means?

The MVP allows **seprate the presentation layer from the logic**, so that the things about how the interface works are separate from how we represent it on screen. 

Keep in mind: MVP is not an architecture pattern, it's only responsible for the presentation layer.

#### How to use MVP?

As it's talked above, MVP is just a design pattern to decoupling the presentation layer. So there are a lot of ways to implement it. In this topic, we will show one implementation by combining the real project.

So let's start!!

### Implementation

Consider about below diagram:

```
sequenceDiagram
View->>Presenter: doLogin
Presenter->>Model: goLogin!
Model->>Presenter: login successfully
Presenter->>View: Show successful message
```


View is not talking to Model anymore, instead, Presenter is the mid-layer to communicate things with each other.

Further, all of these layers should be platform independent especially on View. Most of time, we are using Activity/Fragment to display screen UI on android platform as View layer. If the Presenter calls Activity directly, it has too strong reference to the special platform, and its hard to make our goal: presenter layer is seprated from the logic of how the UI interface works. By instead, we should define an abstract interface for View layer so that it doesn't care about if it's implemented by Activity or Fragment or anything else.

View

```java
public interface IBaseView {
}
```

Presenter

```java
public abstract class BasePresenter<T extends IBaseView> {
    private T view;
    
    public IPresenter(T view) {
        this.view = view;
    }
    
    protected final T getView() {
        return view;
    }
    
}
```

ViewImpl

```java
public abstract BaseActivity<P extends BasePresenter> extends Activity {
    @Inject
    P presenter;
    
    protected final P getPresenter() {
        return this.presenter;
    }
    
    @Override
    public void oncreate(Bundle onSavedStateInstance) {
        
    }
}
```

Now we have general framework setup with MVP pattern!
