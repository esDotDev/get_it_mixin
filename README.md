# get_it_mixin

This package offers a set of mixin types that make it very easy to bind your Widgets views to the data stored in GetIt.

>When I write of binding, I mean a mechanism that will automatically rebuild a widget that if data it depends on changes 

Several users have been asking for data binding support similar to that offered by `provider`. At the same time, I have to admit, I became quite intrigued by `flutter_hooks` from [Remi Rousselet](https://github.com/rrousselGit/), and started to think about how I could create something similar for `GetIt`. **I'm very thankful for Remi's work. I took more than one inspiration from his code.**

In order to keep `GetIt` free of Flutter dependencies I chose to write a separate package using mixins.

To be clear: you can achieve the same using different Flutter Builders but it will make your code less readable and more verbose.

## Getting started
>For this readme I expect that you know how to work with [GetIt](https://pub.dev/packages/get_it)

First, lets create some Model class that we want to access within our View:

```Dart
class Model extends ChangeNotifier {
  String _country;
  set country(String val) {
    _country = val;
    notifyListeners();
  }
  String get country => _country;

  String _emailAddress;
  set country(String val) {
    _emailAddress = val;
    notifyListeners();
  }
  String get emailAddress => _emailAddress;

  final ValueNotifier<String> name;
  final Model nestedModel;

  Stream<String> userNameUpdates; 
  Future get initializationReady;
}
```

With the Model in place we can explore how to access it in various ways using `get_it_mixin`.

### Reading Data

When you add the `GetItMixin` to your `StatelessWidget` you get some new functions that you can use inside your Widgets. 

The easiest ones are `get()` and `getX()` which simmply lookup data from `GetIt` similar to calling `GetIt.I<Type>()`.

```Dart
class TestStateLessWidget extends StatelessWidget with GetItMixin {

  @override
  Widget build(BuildContext context) {
    final email = get<Model>().emailAddress;
    return Column(
      children: [
        Text(email),
        Text(getX((Model x) => x.country, instanceName: 'secondModell')),
      ],
    );
  }
}
```

As you can see `get()` is used exactly like using `GetIt` directly with all its parameters. `getX()` does the same but offers a selector function that returns a property on the referenced object. Most of the time you will probably only use `get()`, but the selector function can be used for any data processing reqired before you can use the value.

**get() and getX() can be called multiple times inside a Widget and also outside the `build()` function.**

### Watching Data
The following functions will return a value and rebuild the widget every-time the value changes. **Important: This function can only be called inside the `build()` method and you can only watch an object once. Also, all of these function have to be called in the same order on every `build` (meaning they can't be called conditionally), otherwise the mixin gets confused.**

Imagine you have an object registered with `GetIt` that implements `ValueListenableBuilder<String> currentUserName` and we want to bind it to a Widget, that rebuilds any time the `currentUserName` changes. 

We could do this adding a `ValueListenableBuilder`:

```Dart
class TestStateLessWidget1 extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        ValueListenableBuilder<String>(
          valueListenable: GetIt.I<ValueListenable<String>>(instanceName: 'currentUser'),
          builder: (context, val,_) {
            return Text(val);
          }
        ),
      ],
    );
  }
}
```

With the mixin we can now write this:

```Dart
class TestStateLessWidget1 extends StatelessWidget with GetItMixin {
  @override
  Widget build(BuildContext context) {
    final currentUser = 
       watch<ValueListenable<String>, String>(instanceName: 'currentUser');

    return Column(
      children: [
         Text(currentUser)
      ],
    );
  }
}
```

Unfortunately we have to provide a second generic parameter because Dart can't infer the type of the return value. Luckily we will see with the following functions there is a way to help the compiler.

#### WatchX
In a real app it's way more probable that your business object wont be the `ValueListenable` itself but it will have some properties that might be `ValueListenables` themselves. Like the `name` property of our `Model` class. To react to changes to of such properties you can use `watchX()`:

```Dart
class TestStateLessWidget1 extends StatelessWidget with GetItMixin {
  @override
  Widget build(BuildContext context) {
    final name = watchX((Model x) => x.name);
    /// if the valueListenable is nested deeper in you object
    final innerName = watchX((Model x) => x.nestedModel.name);

    return Column(
      children: [
        Text(name),
        Text(innerName),
      ],
    );
  }
}
```

This widget will rebuild whenever one of the watched `ValueListenables` changes.

You might be wondering why I did not pass the type `Model` as generic Parameter to `watchX()` as the signature looks like this:

```Dart
R watchX<T, R>(
    ValueListenable<R> Function(T x) select, {
    String instanceName,
  }) =>
```
Passing the generic types are not required here due to type inference. If you pass `T` inside the `select` function the compiler is able to infer `R`! 

#### watchOnly & watchXonly
Another popular pattern is where a business object implements `Listenable`, like `ChangeNotifier`, and it will notify its listeners whenever one of its properties changes. 

If you want to rebuild whenever Model triggers `notifyListener` you can simple call `watchOnly`:

```Dart
final model = watchOnly((Model x) => x);
```

For more granular rebuilds `watchOnly()` lets you define which a property you want to observe and will only rebuild when the value changes. `watchXonly()` does the same but for nested `Listenables`.

```Dart
class TestStateLessWidget1 extends StatelessWidget with GetItMixin {
  @override
  Widget build(BuildContext context) {
    final country = watchOnly((Model x) => x.country);
    /// if the watched property is nested deeper in you object
    final innerEmail = watchXOnly((Model x) => x.nestedModel,(Model o)=>o.emailAddress);

    return Column(
      children: [
        Text(country),
        Text(innerEamil),
      ],
    );
  }
}
```

This Widget will rebuild when either `model.country` or `model.nestedModel.emailAddress` changes. If `model.emailAddress` changes, it won't trigger a rebuild, despite it calling `notifyListeners` internally.


#### Streams and Futures
To update your widget when a Stream in your Model emits a new value or a `Future` completes, you can use `watchStream` and `watchFuture`. 

The nice thing is that you don't have to worry about cancelling subscriptions. The mixin takes care of that. So instead of using a `StreamBuilder` you can just do:

```Dart
class TestStateLessWidget1 extends StatelessWidget with GetItMixin {
  @override
  Widget build(BuildContext context) {
    final currentUser = watchStream((Model x) => x.userNameUpdates, 'NoUser');
    final ready =
        watchFuture((Model x) => x.initializationReady,false).data;

    return Column(
      children: [
        if (ready != true || !currentUser.hasData) // in case of an error it could be null
         CircularProgressIndicator()
         else
        Text(currentUser.data),
      ],
    );
  }
}
```

These functions changing the Streams and Futures instances on following `build` calls. In this case the old subscription is cancelled and the new `Stream` subscribed. Check the API docs for more details.


### Event handlers
Maybe you don't need a value updated but want to show a Snackbar as soon as a `Stream` emits or a `ValueListenable` changes. If you wanted to do this without this mixin you would need a `StatefulWidget` where you subscribe to a `Stream` in `iniState` and dispose your subscription in the `dispose` function of the `State`.

With this mixin you can register handlers for `Streams` and `ValueListenables` and the mixin will dispose everything for you as soon as the widget gets destroyed.

```Dart
class TestStateLessWidget1 extends StatelessWidget with GetItMixin {
  @override
  Widget build(BuildContext context) {
    registerStreamHandler((Model x) => x.userNameUpdates, (name,_) 
        => showNameDialog(name));
    registerValueListenableHandler((Model x) => x.name, (name,_) 
        => showNameDialog(name));
    return Column(
      children: [
        //...whatever widgets needed 
      ],
    );
  }
}
```

For instance you could register a handler for `thrownExceptions` of a `flutter_command` while you use `watch()` to get the values.

In the example above you see that the handler function has a second parameter that we ignored. This is a dispose function that the handler could use to kill a registration from within itself.

## StatefulWidgets
All the functions above are available for `StatefulWidgets` too. However with this mixin the need for `StatefulWidgets` will drastically decline.
In case you need one and also want to use the comfort of this you have to use two different mixins.

```Dart
class TestStatefulWidget extends StatefulWidget with GetItStatefulWidgetMixin {
  @override
  _TestStatefulWidgetState createState() => _TestStatefulWidgetState();
}

class _TestStatefulWidgetState extends State<TestStatefulWidget> with GetItStateMixin {
  @override
  Widget build(BuildContext context) {
    final currentUser = watchX((Model x) => x.name,);
    return Column(
      children: [
        Text(currentUser),
      ],
    );
  }
}
```

Unfortunately we need two mixins in this case otherwise the automatic updating could not be realised.
