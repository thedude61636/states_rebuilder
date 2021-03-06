# ex_009_1_3_ca_todo_mvc_with_state_persistence

States_rebuilder is a simple and efficient state management solution for Flutter.
By this example, I will demonstrate the above statement.

The example consists of the [Todo MVC app](https://github.com/brianegan/flutter_architecture_samples/blob/master/app_spec.md) extended to handle dynamic dark/light theme and app internationalization.
The app state will be stored using SharedPreferences, Hive, and sqflite for demonstration purposes.

# Setting persistence provider

Since we want to persist in the chosen theme and language as well as the todos list, we start by defining the persistence provider.

with states_rebuilder, you have the freedom of choosing your storage provider. All you need to do is to implement the `IPersistStore` interface. 

## SharedPreferences:
```dart
class SharedPreferencesImp implements IPersistStore {
  SharedPreferences _sharedPreferences;

  @override
  Future<void> init() async {
    //Initialize the plugging
    _sharedPreferences = await SharedPreferences.getInstance();
  }

  @override
  Object read(String key) {
    try {
      return _sharedPreferences.getString(key);
    } catch (e) {
      //throw a costume exceptions
      throw PersistanceException('There is a problem in reading $key: $e');
    }
  }

  @override
  Future<void> write<T>(String key, T value) async {
    try {
      return _sharedPreferences.setString(key, value as String);
    } catch (e) {
    //throw a costume exceptions
      throw PersistanceException('There is a problem in writing $key: $e');
    }
  }

  @override
  Future<void> delete(String key) async {
    return _sharedPreferences.remove(key);
  }

  @override
  Future<void> deleteAll() {
    return _sharedPreferences.clear();
  }
}
```

## Hive:
```dart
class HiveImp implements IPersistStore {
  Box box;

  @override
  Future<void> init() async {
    await Hive.initFlutter();
    box = await Hive.openBox('myBox');
  }

  @override
  Object read(String key) {
    try {
      return box.get(key);
    } catch (e) {
      throw PersistanceException('There is a problem in reading $key: $e');
    }
  }

  @override
  Future<void> write<T>(String key, T value) async {
    try {
      return box.put(key, value);
    } catch (e) {
      throw PersistanceException('There is a problem in writing $key: $e');
    }
  }

  @override
  Future<void> delete(String key) async {
    return box.delete(key);
  }

  @override
  Future<void> deleteAll() async {
    return box.clear();
  }
}
```

## Sqflite:
It's not the best choice here, but I give it for demonstration purpose.
```dart
class SqfliteImp implements IPersistStore {
  Database _db;
  final _tableName = 'AppStorage';

  @override
  Future<void> init() async {
    final databasesPath =
        await path_provider.getApplicationDocumentsDirectory();
    _db = await openDatabase(
      join(databasesPath.path, 'todo_db.db'),
      version: 1,
      onCreate: (db, _) async {
        await db.execute(
          'CREATE TABLE $_tableName (key TEXT PRIMARY KEY, value TEXT)',
        );
      },
    );
  }

  @override
  Object read(String key) async {
    try {
      final result = await _db.query(
        _tableName,
        where: 'key = ?',
        whereArgs: [key],
      );
      if (result.isNotEmpty) {
        return result.first['value'];
      }
      return null;
    } catch (e) {
      throw PersistanceException('There is a problem in reading $key: $e');
    }
  }

  @override
  Future<void> write<T>(String key, T value) async {
    try {
      return await _db.insert(
        _tableName,
        {
          'key': key,
          'value': value,
        },
        conflictAlgorithm: ConflictAlgorithm.replace,
      );
    } catch (e) {
      throw PersistanceException('There is a problem in writing $key: $e');
    }
  }

  @override
  Future<void> delete(String key) async {
    return _db.delete(_tableName, where: 'key = $key');
  }

  @override
  Future<void> deleteAll() async {
    return _db.delete(_tableName);
  }
}
```

# Dynamic dark/light theme
Since we want to toggle between dark and light mode, we inject and persist a Boolean value to know if we've chosen dark or light.

```dart
final isDarkMode = RM.inject<bool>(
  () => true,
  //Show our intention to persist the state by defining the persist parameter
  persist: () => PersistState(
    //Give it a unique key. [key / value]
    key: '__themeData__',
    //Tell how to transition from json to state and the opposite.
    //Our case is simple:
    //'1' is true, and '0' is false
    fromJson: (json) => json == '1',
    toJson: (themeData) => themeData ? '1' : '0',
  ),
);
```
That's all for the business logic part. In the UI, we can register to the injected `isDarkMode` and change its state.

>  You can handle a rainbow of themes using enumeration rather than boolean primitive

[Refer to main.dart](lib/main.dart)
```dart
class App extends StatelessWidget {
  const App({Key key}) : super(key: key);
  @override
  Widget build(BuildContext context) {
    //Register to isDarkMode
    return isDarkMode.whenRebuilderOr(
      onWaiting: () => const Center(
        child: const CircularProgressIndicator(),
      ),
      builder: () {
        return MaterialApp(
          //On app start, the state of isDarkMode is read from the storage
          theme: isDarkMode.state ? ThemeData.dark() : ThemeData.light(),
          home: .... 
        );
      },
    );
  }
}
```
To change the theme, we simply switch the state as follows:

[Refer to Extra Actions Button](lib/ui/pages/home_screen/extra_actions_button.dart#L24)
```dart
 isDarkMode.state = !isDarkMode.state;
```

<details>
  <summary>Click here to see how dynamic theming is tested</summary>

[Refer to main_test.dart file](test/main_test.dart#L9)
```dart
  testWidgets('Toggle theme should work', (tester) async {
    await tester.pumpWidget(App());
    //App start in dark mode
    expect(Theme.of(RM.context).brightness == Brightness.dark, isTrue);

    //tap on the ExtraActionsButton
    await tester.tap(find.byType(ExtraActionsButton));
    await tester.pumpAndSettle();
    //And tap to toggle to light mode
    await tester.tap(find.byKey(Key('__toggleDarkMode__')));
    await tester.pumpAndSettle();
    //
    //Expect the themeData is persisted
    expect(storage.store['__themeData__'], '0');
    //And theme is light
    expect(Theme.of(RM.context).brightness == Brightness.light, isTrue);
    //
    //Tap to toggle theme to dark mode
    await tester.tap(find.byType(ExtraActionsButton));
    await tester.pumpAndSettle();
    await tester.tap(find.byKey(Key('__toggleDarkMode__')));
    await tester.pumpAndSettle();
    //
    //The storage.stored themeData is updated
    expect(storage.store['__themeData__'], '1');
    //And theme is dark
    expect(Theme.of(RM.context).brightness == Brightness.dark, isTrue);
  });
```
</details>

<br>

# Localization configuration

There are many ways to configure the localization of the app. In this example, we first start by defining an abstract class `I18N` which have tree static methods and the default strings of our app.


[Refer to language_base file](lib/ui/common/localization/languages/language_base.dart)
```dart
abstract class I18N {
  ///A map of Locale to its I18N implementation
  static Map<Locale, I18N> _supportedLanguage = {
    Locale.fromSubtags(languageCode: 'en'): EN(), //EN and AR implements I18N
    Locale.fromSubtags(languageCode: 'ar'): AR(),
    //
    //Add new locales here
  };

  //Get the supportedLocale. To be used in MaterialApp widget
  static List<Locale>  getSupportedLocale => _supportedLanguage.keys.toList();
  
  //Get the language implementation from the chosen locale
  static I18N getLanguages(Locale locale) => _supportedLanguage[locale] ?? EN();

  //Default Strings of the app (in English)
  String appTitle = 'States_rebuilder Example';
  String todos = 'Todos';
  String stats = 'Stats';

  //You can use methods

}
```
Now we have to define the EN and AR or any other language implementations of I18N interface:
 
[Refer to en_us.dart file](lib/ui/common/localization/languages/en_us.dart)
```dart
//This is the default language
class EN extends I18N {}
```

[Refer to ar.dart file](lib/ui/common/localization/languages/ar.dart)
```dart
//This is the default language
class AR extends I18N {
  String appTitle = 'States_rebuilder مثال';
  String todos = 'واجبات';

  String stats = 'إحصاء';



}
```

Now, we are ready to inject and persist our locale.


[Refer to ar.dart file](lib/ui/common/localization/localization.dart)
```dart
//Inject and persist the locale
final locale = RM.inject<Locale>(
  () => Locale.fromSubtags(languageCode: 'en'),
  onData: (_) {
    //Each time the locale is changed, we refresh the i18n to get the right language implementation
    return i18n.refresh();
  },
  //Persist the locale
  persist: () => PersistState(
    key: '__localization__',
    //
    //take the stored String and return a Locale object
    fromJson: (String json) => Locale.fromSubtags(languageCode: json),
    //
    //any non supported locale will be stored as 'und'.
    //'und' also will be used for system local
    toJson: (locale) =>
        I18N.supportedLocale.contains(locale) ? locale.languageCode : 'und',
  ),
);
```

[Refer to ar.dart file](lib/ui/common/localization/localization.dart)
```dart
//Inject i18n
//We must register with I18N interface
final Injected<I18N> i18n = RM.inject(
  () {
    //Whenever i18n is refreshed, (from onData of locale) it gets the corresponding language implementation
    return I18N.getLanguages(locale.state);
  },
);
```

In the UI:

[Refer to main.dart file](lib/main.dart)
```dart
class App extends StatelessWidget {
  const App({Key key}) : super(key: key);
  @override
  Widget build(BuildContext context) {
    //Listen to the injected isDarkModel and locale
    return [isDarkMode, locale].whenRebuilderOr(
      //If any of isDarkMode or locale injected models are waiting 
      //we will display a CircularProgressIndicator
      onWaiting: () => const Center(
        child: const CircularProgressIndicator(),
      ),
      builder: () {
        // Resister to i18n with an InheritedWidget of its type.
        //the state of i18n can be obtained using of(context) method and 
        //when the i18n changes, all widgets that have used the of(context) will rebuild.
        return i18n.inherited(
          builder: (context) => MaterialApp(
            //Get the i18n translation using the of(context) method
            title: i18n.of(context).appTitle,
            theme: isDarkMode.state ? ThemeData.dark() : ThemeData.light(),
            locale: locale.state.languageCode == 'und' ? null : locale.state,
            supportedLocales: I18N.supportedLocale,
            localizationsDelegates: [
              GlobalMaterialLocalizations.delegate,
              GlobalWidgetsLocalizations.delegate,
            ],
            routes: {
              //Notice const here and everywhere in this app.
              // This is a huge benefit in performance terms.
              AddEditPage.routeName: (context) => const AddEditPage(),
              HomeScreen.routeName: (context) => const HomeScreen(),
            },
            //Set navigation key so states_rebuilder can navigate without BuildContext
            navigatorKey: RM.navigate.navigatorKey,
          ),
        );
      },
    );
  }
}
```
To change the locale, we can do it manually :

[Refer to main.dart file](lib/ui/pages/home_screen/languages.dart#L9)
```dart
locale.state = Locale.fromSubtags(languageCode: 'ar');
```

Or listen to the system's locale change, and mutate the locale state

[Refer to main.dart file](lib/main.dart#L21)
```dart
//
StateWithMixinBuilder.widgetsBindingObserver(
    //Called when the system locale is changed
    didChangeLocales: (context, locales) {
    if (locale.state.languageCode == 'und') {
        locale.state = locales.first;
    }
    },
    builder: (_, __) => App(),
),
```

<details>
  <summary>Click here to see how app localization is tested</summary>

[Refer to main_test.dart file](test/main_test.dart#L39)
```dart
  testWidgets('Change language should work', (tester) async {
    await tester.pumpWidget(App());
    //App start with english
    expect(MaterialLocalizations.of(RM.context).alertDialogLabel, 'Alert');

    //Tap on the language action button
    await tester.tap(find.byType(Languages));
    await tester.pumpAndSettle();
    //choose 'AR' language
    await tester.tap(find.text('AR'));
    await tester.pump();
    await tester.pumpAndSettle();
    //ar is persisted
    expect(storage.store['__localization__'], 'ar');
    //App is in arabic
    expect(MaterialLocalizations.of(RM.context).alertDialogLabel, 'تنبيه');
    //
    await tester.tap(find.byType(Languages));
    await tester.pumpAndSettle();
    //tap to use system language
    await tester.tap(find.byKey(Key('__System_language__')));
    await tester.pump();
    await tester.pumpAndSettle();
    //and for systemLanguage is persisted
    expect(storage.store['__localization__'], 'und');
    //App is back to system language (english).
    expect(MaterialLocalizations.of(RM.context).alertDialogLabel, 'Alert');
  });
```
</details>
<br>

# Todos logic

For the todos we first start defining a Todo data class:


<details>
  <summary>Click here to see the Todo class</summary>

[Refer to todo.dart file](lib/domain/entities/todo.dart)
```dart
@immutable
class Todo {
  final String id;
  final bool complete;
  final String note;
  final String task;

  Todo(this.task, {String id, this.note, this.complete = false})
      : id = id ?? Uuid().v4();

  factory Todo.fromJson(Map<String, Object> map) {
    if (map == null) {
      return null;
    }
    return Todo(
      map['task'] as String,
      id: map['id'] as String,
      note: map['note'] as String,
      complete: map['complete'] as bool,
    );
  }

  // toJson is called just before persistence.
  Map<String, Object> toJson() {
    _validation();
    return {
      'complete': complete,
      'task': task,
      'note': note,
      'id': id,
    };
  }

  void _validation() {
    if (id == null) {
      // Custom defined error classes
      throw ValidationException('This todo has no ID!');
    }
    if (task == null || task.isEmpty) {
      throw ValidationException('Empty task are not allowed');
    }
  }

  Todo copyWith({
    String task,
    String note,
    bool complete,
    String id,
  }) {
    return Todo(
      task ?? this.task,
      id: id ?? this.id,
      note: note ?? this.note,
      complete: complete ?? this.complete,
    );
  }

  @override
  bool operator ==(Object o) {
    if (identical(this, o)) return true;

    return o is Todo &&
        o.id == id &&
        o.complete == complete &&
        o.note == note &&
        o.task == task;
  }

  @override
  int get hashCode {
    return id.hashCode ^ complete.hashCode ^ note.hashCode ^ task.hashCode;
  }

  @override
  String toString() {
    return 'Todo(task:$task, complete: $complete)';
  }
}
```
</details>
<br>

The logic for adding, updating, deleting, and toggling todos is encapsulated in `ListTodoX` extension.

[Refer to todo.dart file](lib/service/todos_state.dart)
```dart
extension ListTodoX on List<Todo> {
  //Add a todo
  List<Todo> addTodo(Todo todo) {
    return List<Todo>.from(this)..add(todo);
  }

  //Update todo
  List<Todo> updateTodo(Todo todo) {
    return map((t) => t.id == todo.id ? todo : t).toList();
  }

  List<Todo> deleteTodo(Todo todo) {
    return List<Todo>.from(this)..remove(todo);
  }

  //toggle all todos
  List<Todo> toggleAll() {
    final allComplete = this.every((e) => e.complete);
    return map(
      (t) => t.copyWith(complete: !allComplete),
    ).toList();
  }

  List<Todo> clearCompleted() {
    return List<Todo>.from(this)
      ..removeWhere(
        (t) => t.complete,
      );
  }

  ///Parsing the state, use is state persistance
  String toJson() => convert.json.encode(this);
  static List<Todo> fromJson(json) {
    final result = convert.json.decode(json) as List<dynamic>;
    return result.map((m) => Todo.fromJson(m)).toList();
  }
}
```

The next step is to inject and persist the todos list:

[Refer to injected.dart file](lib/injected.dart#L10)
```dart
final Injected<List<Todo>> todos = RM.inject(
  () => [],//Start with empty list
  persist: () => PersistState(
    key: '__Todos__',
    //parse the state
    toJson: (todos) => todos.toJson(),
    fromJson: (json) => ListTodoX.fromJson(json),
  ),
  //If the persistence is failed, a snackbar is displayed and the state is undo to the last valid state
  onError: (e, s) => ErrorHandler.showErrorSnackBar(e),
  //As we want to manually undo the state after deleting a todo, we set the undo stack length to 1
  undoStackLength: 1,
);
```
As the todos can be filtered (All, completed, active), we inject a `VisibilityFilter` enumeration and a computed filtered todos:

[Refer to injected.dart file](lib/injected.dart#L25)
```dart
final activeFilter = RM.inject(() => VisibilityFilter.all);

//this todosFiltered will be recomputed whenever the activeFilter or todos state is changed.
final Injected<List<Todo>> todosFiltered = RM.injectComputed(
  compute: (_) {
    //Return the active todos
    if (activeFilter.state == VisibilityFilter.active) {
      return todos.state.where((t) => !t.complete).toList();
    }
    //Return the completed todos
    if (activeFilter.state == VisibilityFilter.completed) {
      return todos.state.where((t) => t.complete).toList();
    }
    //Return all todos
    return todos.state;
  },
);
```
> `todosFiltered` results in a performance gain. The filtered todos are cached in memory so that they are not recalculated unless the value of the `VisibilityFilter` is modified (by the user) or the original list of todos is changed (add, delete, update the todos ).


## The UI

### AppTab
> The home contains two Tabs: [the List of Todos and Stats about the Todos](https://github.com/brianegan/flutter_architecture_samples/blob/master/app_spec.md#Home-Screen).

To Navigate between the list of todos and stats we define the AppTab enumeration and inject it.


[Refer to injected.dart file](lib/injected.dart#L40)
```dart
enum AppTab { todos, stats }

final activeTab = RM.inject(() => AppTab.todos);
```
In the `home_screen` we consume the activeTab injected model:

[Refer to home_screen.dart file](lib\ui\pages\home_screen\home_screen.dart#L33)
```dart

  Widget build(BuildContext context) {
    return Scaffold(
      appBar: ... ,
      body: todos.whenRebuilderOr(

        //Shows a loading screen until the Todos have been loaded from file storage or the web.
        onWaiting: () => const Center(
          child: const CircularProgressIndicator(),
        ),

        //subscribe to activeTab and depending on its state render the wanted widget
        builder: () => activeTab.rebuilder(
          () => activeTab.state == AppTab.todos
              ? const TodoList()
              : const StatsCounter(),
        ),
      ),
      floatingActionButton: ...,
      bottomNavigationBar: activeTab.rebuilder(
        () => BottomNavigationBar(
          currentIndex: AppTab.values.indexOf(activeTab.state),
          onTap: (index) {
            //Mutate the state of the activeTab
            activeTab.state = AppTab.values[index];
          },
          items: [
            BottomNavigationBarItem(
              icon: Icon(Icons.list),
              title: Text(i18n.of(context).stats),
            ),
            BottomNavigationBarItem(
              icon: Icon(Icons.show_chart),
              title: Text(i18n.of(context).todos),
            ),
          ],
        ),
      ),
    );
  }
```

### Load and display list of todos
> [Displays the list of Todos entered by the User](https://github.com/brianegan/flutter_architecture_samples/blob/master/app_spec.md#List-of-Todos)

When the application starts, the todos list is retrieved from the provided storage. Once the list of todos is obtained, we use a `ListView` builder to display the todos items (`TodoItem` widget).

For performance and build optimization, we won't pass any dynamic parameters to the `TodoItem` so we can use the `const` modifier.

To get the state of any todo from its `TodoItem` child widgets, we will rely on` InheritedWidget`.

With states_rebuilder, we can inject a widget-aware state; in other words, the state is obtained according to its position in the widget tree. Here we are referring to the concept of `InheritedWidget`.


Note: I assume you are familiar with how InheritedWidget works. [Read here for more information](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html)

To do this, we first define a global injection to represent the state of an Item.

[Refer to injected.dart file](lib\injected.dart#L52)
```dart
//This is called the global state reference of Injected<Todo> items
//This can be seen as a template for a todo state
final Injected<Todo> todoItem = RM.inject(() => null);//Will be overridden
```
To injected an `Injected<Todo>` in the widget tree we use :

```dart
  //Widget-aware injection
  return todoItem.inherited(
    //The global state is null as defined above. Here we
    //override it. and define how one todo state is obtained 
    //from list of todos
    stateOverride: () =>  todos[index],// the Todo state
    builder: (_) => const TodoItem(), // const here
  );
```

Form a child widget we can get the injected Todo:

```dart
  final todoItem = todoItem(context); //Internally call InheritedWidget
```

todoItem has onWaiting, onError and onData callbacks.
```dart
final Injected<Todo> todoItem = RM.inject(
  () => null,
  onWaiting: (){
  //Called if at least one todo item state is waiting
    print('onWaiting');
  }
  onError: (e, s) {
  //Called if at least one todo item state throws an error
    ErrorHandler.showErrorSnackBar(e);
  },
  //Called if all todo items has data and exposed the todo state of the item emitting data
  onData: (todo) => todos.state.updateTodo(todo), 
);
```

Wrap up:
* `Injected.inherited` is useful when displaying a list of items of the same type (Products, todos, ..).
* Relying on the concept of Inherited widget, the right item state is obtained using the BuildContext.
* Global state reference is something like a template for one item
* The global state reference, exposes there callbacks:
  * `onWaiting` : called if at least one item is waiting for a pending async task.
  * `onError`: called if no item is waiting, and at least one item has an error.
  * `onData`: called if all items have data.
* When the refresh method is called on a global state reference, all item states will be refreshed.


[Refer to todo list file](lib/ui/pages/home_screen/todo_list.dart)
```dart
class TodoList extends StatelessWidget {
  const TodoList();
  @override
  Widget build(BuildContext context) {

    //Subscribe to todosFiltered. 
    //todosFiltered.rebuilder is invoked each time the todos and/or activeFilter injected models are changed
    return todosFiltered.rebuilder(
      () {
        final todos = todosFiltered.state;
        return ListView.builder(
          itemCount: todos.length,
          itemBuilder: (BuildContext context, int index) {
            
            //
            return todoItem.inherited(
              //As this is a list of dismissible items a key must be given
              key: Key('${todos[index].id}'),
              stateOverride: () =>  todos[index],
              //Build is optimized by using const
              builder: (context) => const TodoItem(),
            );
          },
        );
      },
      //By default 'rebuilder' will rebuild on data only.
      //In addition, in our case we want it to rebuild on error so to return back to the last state if 
      //the persistance failed. (We can use whenRebuilderOr instead)
      shouldRebuild: () => todosFiltered.hasData || todosFiltered.hasError,
    );
  }
}
```

The TodoItem looks like this :

[Refer to todo item file](lib/ui/pages/home_screen/todo_item.dart)
```dart
class TodoItem extends StatelessWidget {
  //const constructor
  const TodoItem({ Key key }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    //Get the todo state using the context.
    final todo = todoItem(context);
    return todo.rebuilder(
      () {
        return Dismissible(
          key: Key('__${todo.state.id}__'),
          onDismissed: (direction) {
            //remove a todo on dismiss
            removeTodo(todo.state);
          },
          child: ListTile(
            onTap: () async {
              //Navigate to DetailScreen
              final shouldDelete = await RM.navigate.to(
                //As this is a new route, getting a todo state from context no longer possible.
                //Using the 'reInherited' method will make the todo state available on the new route.
                todoItem.reInherited(
                  context: context,
                  builder: (context) => const DetailScreen(),
                ),
              );
              if (shouldDelete == true) {
                //Removing todo will show a Snackbar
                //We explicitly set the context to get the right scaffold
                RM.scaffoldShow.context = context;
                removeTodo(todo.state);
              }
            },
            leading: Checkbox(
              //key used for test
              key: Key('__Checkbox${todo.state.id}__'),
              value: todo.state.complete,
              onChanged: (value) {
                final newTodo = todo.state.copyWith(
                  complete: value,
                );
                //set the new todo state
                //This will toggle the value of th checkBox
                //and
                //the onData of todoItem is invoked to update the todos list
                todo.state = newTodo;
              },
            ),
            title: Text(
              todo.state.task,
              style: Theme.of(context).textTheme.headline6,
            ),
            subtitle: Text(
              todo.state.note,
              maxLines: 1,
              overflow: TextOverflow.ellipsis,
              style: Theme.of(context).textTheme.subtitle1,
            ),
          ),
        );
      },
    );
  }
}
```
**Tests**
<details>
  <summary>Click here to see add a todo test</summary>

[Refer to stats_test.dart file](test/home_screen_test.dart#L33)
```dart
  testWidgets('add todo', (tester) async {
    await tester.pumpWidget(App());
    //No todos
    expect(find.byType(TodoItem), findsNothing);
    //Tap on FloatingActionButton to add a todo
    await tester.tap(find.byType(FloatingActionButton));
    await tester.pumpAndSettle();
    //We are in the AddEditPage
    expect(find.byType(AddEditPage), findsOneWidget);
    //
    //Enter some text
    await tester.enterText(find.byKey(Key('__TaskField')), 'Task 1');
    await tester.enterText(find.byKey(Key('__NoteField')), 'Note 1');
    //
    //Tap on FloatingActionButton to add  the todo
    await tester.tap(find.byType(FloatingActionButton));
    await tester.pumpAndSettle();
    //We are back in the HomeScreen
    expect(find.byType(HomeScreen), findsOneWidget);
    //And a todo is displayed
    expect(find.byType(TodoItem), findsOneWidget);
    //
    //The add todo is persisted
    expect(storage.store['__Todos__'].contains('"task":"Task 1"'), isTrue);
    expect(storage.store['__Todos__'].contains('"note":"Note 1"'), isTrue);
  });
```
</details>


<details>
  <summary>Click here to see remove todo using a dismissible and undo test</summary>

[Refer to stats_test.dart file](test/home_screen_test.dart#L59)
```dart
  testWidgets('Remove todo using a dismissible and manual undo', (tester) async {
    //pre populate the store with tree todos
    storage.store.addAll({'__Todos__': todos3});
    await tester.pumpWidget(App());
    //Start with three todos
    expect(find.byType(TodoItem), findsNWidgets(3));
    expect(storage.store['__Todos__'].contains('"note":"Note2"'), isTrue);
    //Dismiss the second todo
    await tester.drag(find.text('Note2'), Offset(-1000, 0));
    await tester.pumpAndSettle();
    //
    //the second todo is removed
    expect(find.text('Note2'), findsNothing);
    expect(find.byType(TodoItem), findsNWidgets(2));
    //The new state is persisted
    expect(storage.store['__Todos__'].contains('"note":"Note2"'), isFalse);
    //A SnackBar is displayed with undo button
    expect(find.byType(SnackBar), findsOneWidget);
    expect(find.text('Undo'), findsOneWidget);
    //
    //tap undo to restore the removed todo
    await tester.tap(find.text('Undo'));
    await tester.pumpAndSettle();
    expect(find.byType(TodoItem), findsNWidgets(3));
    expect(storage.store['__Todos__'].contains('"note":"Note2"'), isTrue);
  });
```
</details>
<br>
This is important. Here we fake a persistence provider failure and see that states_rebuilder mutate the state to the new desired state and try to persist it, as it fails it goes back to the last valid state and displays an error message.

<details>
  <summary>Click here to see remove todo using a dismissible and auto undo if persistance fails test</summary>

[Refer to stats_test.dart file](test/home_screen_test.dart#L86)
```dart
   testWidgets(
    'Remove todo using a dismissible and undo if persistance fails',
    (tester) async {
      // storage.should
      //pre populate the store with tree one
      storage.store.addAll({'__Todos__': todos3});

      await tester.pumpWidget(App());
      //Start with three todos
      expect(find.byType(TodoItem), findsNWidgets(3));
      expect(storage.store['__Todos__'].contains('"note":"Note2"'), isTrue);
      //
      //Set the mocked store to throw PersistanceException after one seconds,
      //when writing to the store
      storage.exception = PersistanceException('mock message');
      storage.timeToThrow = 1000;
      //Dismiss the second todo
      await tester.drag(find.text('Note2'), Offset(-1000, 0));
      await tester.pumpAndSettle();
      //
      //the second todo is removed
      expect(find.text('Note2'), findsNothing);
      expect(find.byType(TodoItem), findsNWidgets(2));
      //The new state is persisted
      expect(storage.store['__Todos__'].contains('"note":"Note2"'), isTrue);
      //A SnackBar is displayed with undo button
      expect(find.byType(SnackBar), findsOneWidget);
      expect(find.text('Undo'), findsOneWidget);
      //
      //After one seconds
      await tester.pumpAndSettle(Duration(seconds: 1));
      //The second todo is displayed back
      expect(find.text('Note2'), findsOneWidget);
      expect(find.byType(TodoItem), findsNWidgets(3));
      //A SnackBar with error is displayed
      expect(find.byType(SnackBar), findsOneWidget);
      expect(find.byIcon(Icons.error_outline), findsOneWidget);
    },
  );
```
</details>
<br>
<details>
  <summary>Click here to see toggling a todo form home and from DetailScreen test</summary>

[Refer to stats_test.dart file](test/home_screen_test.dart#L126)
```dart
  testWidgets(
    'should toggle a todo form home and from DetailScreen',
    (tester) async {
      storage.store.addAll({'__Todos__': todos3});
      await tester.pumpWidget(App());
      //
      final checkedCheckBox = find.byWidgetPredicate(
        (widget) => widget is Checkbox && widget.value == true,
      );
      final unCheckedCheckBox = find.byWidgetPredicate(
        (widget) => widget is Checkbox && widget.value == false,
      );

      expect(checkedCheckBox, findsNWidgets(1));
      expect(unCheckedCheckBox, findsNWidgets(2));

      //Check the first todo
      await tester.tap(find.byType(Checkbox).first);
      await tester.pumpAndSettle();
      expect(checkedCheckBox, findsNWidgets(2));
      expect(unCheckedCheckBox, findsNWidgets(1));
      //
      //to on the first todo to go to detailed page
      await tester.tap(find.byType(TodoItem).first);
      await tester.pumpAndSettle();
      //We are in the DetailScreen
      expect(find.byType(DetailScreen), findsOneWidget);
      expect(checkedCheckBox, findsNWidgets(1));
      expect(unCheckedCheckBox, findsNWidgets(0));
      //toggle the todo in the detailed screen
      await tester.tap(find.byType(Checkbox).first);
      await tester.pumpAndSettle();
      //It is unchecked
      expect(checkedCheckBox, findsNWidgets(0));
      expect(unCheckedCheckBox, findsNWidgets(1));
      //
      //Back to the home screen
      RM.navigate.back();
      await tester.pumpAndSettle();
      expect(find.byType(HomeScreen), findsOneWidget);
      //it is updated
      expect(checkedCheckBox, findsNWidgets(1));
      expect(unCheckedCheckBox, findsNWidgets(2));
    },
  );
```
</details>

<details>
  <summary>Click here to see removing a todo form  DetailScreen test</summary>

[Refer to stats_test.dart file](test/home_screen_test.dart#L172)
```dart
  testWidgets(
    'should Remove a todo form  DetailScreen',
    (tester) async {
      storage.store.addAll({'__Todos__': todos3});
      await tester.pumpWidget(App());
      expect(find.byType(TodoItem), findsNWidgets(3));

      //to on the first todo to go to detailed page
      await tester.tap(find.byType(TodoItem).first);
      await tester.pumpAndSettle();
      //We are in the DetailScreen
      expect(find.byType(DetailScreen), findsOneWidget);
      //tap on the delete icon
      await tester.tap(find.byIcon(Icons.delete));
      await tester.pumpAndSettle();
      expect(find.byType(HomeScreen), findsOneWidget);

      expect(find.byType(TodoItem), findsNWidgets(2));
      //A SnackBar is displayed with undo button
      expect(find.byType(SnackBar), findsOneWidget);
      expect(find.text('Undo'), findsOneWidget);
    },
  );
```
</details>

<details>
  <summary>Click here to see editing a todo test</summary>

[Refer to stats_test.dart file](test/home_screen_test.dart#L196)
```dart
  testWidgets(
    'should edit a todo',
    (tester) async {
      storage.store.addAll({'__Todos__': todos3});
      await tester.pumpWidget(App());
      expect(find.byType(TodoItem), findsNWidgets(3));

      //to on the first todo to go to detailed page
      await tester.tap(find.text('Note1').first);
      await tester.pumpAndSettle();
      //We are in the DetailScreen
      expect(find.byType(DetailScreen), findsOneWidget);
      //top on FloatingActionButton to edit todo
      await tester.tap(find.byType(FloatingActionButton).first);
      await tester.pumpAndSettle();
      expect(find.byType(AddEditPage), findsOneWidget);
      //
      //Enter some text
      await tester.enterText(find.byKey(Key('__TaskField')), 'New Task 1');
      await tester.enterText(find.byKey(Key('__NoteField')), 'New Note 1');
      //
      //Tap on FloatingActionButton to submit  the todo
      await tester.tap(find.byType(FloatingActionButton));
      await tester.pumpAndSettle();
      //It is updated
      expect(find.byType(DetailScreen), findsOneWidget);
      expect(find.text('New Task 1'), findsOneWidget);
      //Navigate to home screen
      RM.navigate.back();
      await tester.pumpAndSettle();
      expect(find.byType(HomeScreen), findsOneWidget);
      //it is updated
      expect(find.text('New Task 1'), findsOneWidget);
    },
  );
```
</details>
<br>

### Filter Todos
> [User can filter to show All Todos (Active and Complete), ONLY active todos, or ONLY completed todos](https://github.com/brianegan/flutter_architecture_samples/blob/master/app_spec.md#filter-todos)

[Refer to filter button file](lib/ui/pages/home_screen/filter_button.dart)
```dart
 @override
  Widget build(BuildContext context) {
    //Register to activeFilter model
    return activeFilter.rebuilder(
      () {
        return PopupMenuButton<VisibilityFilter>(
          tooltip: i18n.of(context).filterTodos,
          onSelected: (filter) {
            //mutate  activeFilter and notify listener
            activeFilter.state = filter;
            //As filteredTodos injected model depends on activeFilter, it will be revaluated
            //and display the wanted todos
          },
          itemBuilder: (BuildContext context) =>
              <PopupMenuItem<VisibilityFilter>>[
            PopupMenuItem<VisibilityFilter>(
              key: Key('__Filter_All__'),
              value: VisibilityFilter.all,
              child: Text(
                i18n.of(context).showAll,
                style: activeFilter.state == VisibilityFilter.all
                    ? activeStyle
                    : defaultStyle,
              ),
            ),
            PopupMenuItem<VisibilityFilter>(
              key: Key('__Filter_Active__'),
              value: VisibilityFilter.active,
              child: Text(
                i18n.of(context).showActive,
                style: activeFilter.state == VisibilityFilter.active
                    ? activeStyle
                    : defaultStyle,
              ),
            ),
            PopupMenuItem<VisibilityFilter>(
              key: Key('__Filter_Completed__'),
              value: VisibilityFilter.completed,
              child: Text(
                i18n.of(context).showCompleted,
                style: activeFilter.state == VisibilityFilter.completed
                    ? activeStyle
                    : defaultStyle,
              ),
            ),
          ],
          icon: const Icon(Icons.filter_list),
        );
      },
    );
  }
```

<details>
  <summary>Click here to see filter todos test</summary>

[Refer to stats_test.dart file](test/home_screen_test.dart#L231)
```dart
testWidgets(
    'Show filter todos: all, active and completed todos',
    (tester) async {
      storage.store.addAll({'__Todos__': todos3});
      await tester.pumpWidget(App());
      //
      final checkedCheckBox = find.byWidgetPredicate(
        (widget) => widget is Checkbox && widget.value == true,
      );
      final unCheckedCheckBox = find.byWidgetPredicate(
        (widget) => widget is Checkbox && widget.value == false,
      );

      expect(checkedCheckBox, findsNWidgets(1));
      expect(unCheckedCheckBox, findsNWidgets(2));

      //Top to filter active todos
      await tester.tap(find.byType(FilterButton));
      await tester.pumpAndSettle();
      await tester.tap(find.byKey(Key('__Filter_Active__')));
      await tester.pumpAndSettle();

      //Only active todos are displayed
      expect(checkedCheckBox, findsNWidgets(0));
      expect(unCheckedCheckBox, findsNWidgets(2));
      //
      //Top to filter complete todos
      await tester.tap(find.byType(FilterButton));
      await tester.pumpAndSettle();
      await tester.tap(find.byKey(Key('__Filter_Completed__')));
      await tester.pumpAndSettle();

      //Only completed todos are displayed
      expect(checkedCheckBox, findsNWidgets(1));
      expect(unCheckedCheckBox, findsNWidgets(0));
      //
      //Top to show all todos
      await tester.tap(find.byType(FilterButton));
      await tester.pumpAndSettle();
      await tester.tap(find.byKey(Key('__Filter_All__')));
      await tester.pumpAndSettle();

      //Only completed todos are displayed
      expect(checkedCheckBox, findsNWidgets(1));
      expect(unCheckedCheckBox, findsNWidgets(2));
      //
    },
  );
```
</details>

<br>

### Toggle all todos and clear completed todos
> [If all or some todos are incomplete, all todos in the list are marked as complete. Or if all the todos are marked as complete, all todos in the list are marked as incomplete.](https://github.com/brianegan/flutter_architecture_samples/blob/master/app_spec.md#overflow-menu)

First define and ExtraAction enum and inject it:
[Refer to extra action file](lib/ui/pages/ome_screen/extra_actions_button.dart)
```dart
enum ExtraAction {
  toggleAllComplete,
  clearCompleted,
  toggleDarkMode,
}

final _extraAction = RM.inject(
  () => ExtraAction.clearCompleted,
);
```

[Refer to extra action file](lib/ui/pages/ome_screen/extra_actions_button.dart)
```dart
 @override
  Widget build(BuildContext context) {
    //Register to _extraAction
    return _extraAction.rebuilder(() {
      return PopupMenuButton<ExtraAction>(
        onSelected: (action) {
          //mutate the state of _extraAction
          _extraAction.state = action;

          if (action == ExtraAction.toggleDarkMode) {
            //toggle the darkMode theme
            isDarkMode.state = !isDarkMode.state;
            return;
          }

          if (action == ExtraAction.toggleAllComplete) {
            //set the todos state to toggle all,
            todos.setState((s) => s.toggleAll());
            //Refresh the global todoItem state so that all todo items will be refreshed.
            //Only todo items that are changed will be rebuilt.
            todoItem.refresh();
          } else {
            //Clear all todos
            todos.setState((s) => s.clearCompleted());
          }
        },
        itemBuilder: (BuildContext context) {
          return <PopupMenuItem<ExtraAction>>[
            PopupMenuItem<ExtraAction>(
              key: Key('__toggleAll__'),
              value: ExtraAction.toggleAllComplete,
              child: Text(todosStats.state.allComplete
                  ? i18n.of(context).markAllIncomplete
                  : i18n.of(context).markAllComplete),
            ),
            PopupMenuItem<ExtraAction>(
              key: Key('__toggleClearCompleted__'),
              value: ExtraAction.clearCompleted,
              child: Text(i18n.of(context).clearCompleted),
            ),
            PopupMenuItem<ExtraAction>(
              key: Key('__toggleDarkMode__'),
              value: ExtraAction.toggleDarkMode,
              child: Text(
                isDarkMode.state
                    ? i18n.of(context).switchToLightMode
                    : i18n.of(context).switchToDarkMode,
              ),
            ),
          ];
        },
      );
    });
  }
```


<details>
  <summary>Click here to see toggle all test</summary>

[Refer to stats_test.dart file](test/home_screen_test.dart#L280)
```dart
  testWidgets(
    'toggle all completed / uncompleted',
    (tester) async {
      storage.store.addAll({'__Todos__': todos3});
      await tester.pumpWidget(App());
      //
      final checkedCheckBox = find.byWidgetPredicate(
        (widget) => widget is Checkbox && widget.value == true,
      );
      final unCheckedCheckBox = find.byWidgetPredicate(
        (widget) => widget is Checkbox && widget.value == false,
      );

      expect(checkedCheckBox, findsNWidgets(1));
      expect(unCheckedCheckBox, findsNWidgets(2));

      //Top to toggle all to completed
      await tester.tap(find.byType(ExtraActionsButton));
      await tester.pumpAndSettle();
      await tester.tap(find.byKey(Key('__toggleAll__')));
      await tester.pumpAndSettle();

      //
      expect(checkedCheckBox, findsNWidgets(3));
      expect(unCheckedCheckBox, findsNWidgets(0));
      //
      //Toggle all to uncompleted
      await tester.tap(find.byType(ExtraActionsButton));
      await tester.pumpAndSettle();
      await tester.tap(find.byKey(Key('__toggleAll__')));
      await tester.pumpAndSettle();

      //Only active todos are displayed
      expect(checkedCheckBox, findsNWidgets(0));
      expect(unCheckedCheckBox, findsNWidgets(3));
    },
  );

```
</details>


<details>
  <summary>Click here to see clear competed test</summary>

[Refer to home_screen_test.dart file](test/home_screen_test.dart#L318)
```dart

  testWidgets(
    ' clear completed',
    (tester) async {
      storage.store.addAll({'__Todos__': todos3});
      await tester.pumpWidget(App());
      //
      final checkedCheckBox = find.byWidgetPredicate(
        (widget) => widget is Checkbox && widget.value == true,
      );
      final unCheckedCheckBox = find.byWidgetPredicate(
        (widget) => widget is Checkbox && widget.value == false,
      );

      expect(checkedCheckBox, findsNWidgets(1));
      expect(unCheckedCheckBox, findsNWidgets(2));

      //Top to clear completed
      await tester.tap(find.byType(ExtraActionsButton));
      await tester.pumpAndSettle();
      await tester.tap(find.byKey(Key('__toggleClearCompleted__')));
      await tester.pumpAndSettle();

      //one completed todo is removed
      expect(checkedCheckBox, findsNWidgets(0));
      expect(unCheckedCheckBox, findsNWidgets(2));
      //
      //Toggle all to completed
      await tester.tap(find.byType(ExtraActionsButton));
      await tester.pumpAndSettle();
      await tester.tap(find.byKey(Key('__toggleAll__')));
      await tester.pumpAndSettle();

      //
      expect(checkedCheckBox, findsNWidgets(2));
      expect(unCheckedCheckBox, findsNWidgets(0));

      await tester.tap(find.byType(ExtraActionsButton));
      await tester.pumpAndSettle();
      await tester.tap(find.byKey(Key('__toggleClearCompleted__')));
      await tester.pumpAndSettle();

      //all todos are removed
      expect(checkedCheckBox, findsNWidgets(0));
      expect(unCheckedCheckBox, findsNWidgets(0));
    },
  );

```
</details>

<br>

### Stats Screen
> [Shows a stats of number of completed and active todos](https://github.com/brianegan/flutter_architecture_samples/blob/master/app_spec.md#stats-screen)

First, let's define a data class that contains the stats we want to display:

[Refer to extra action file](lib/domain/value_object/todos_stats.dart)
```dart
class TodosStats {
  final int numCompleted;
  final int numActive;
  final bool allComplete;

  TodosStats({
    @required this.numCompleted,
    @required this.numActive,
  }) : allComplete = numActive == 0;
}
```


The next step is to inject the `TodosStats` as computed:

[Refer to extra action file](lib/ui/pages/ome_screen/extra_actions_button.dart)
```dart
final Injected<TodosStats> todosStats = RM.injectComputed(
  compute: (_) {
    return TodosStats(
      numCompleted: todos.state.where((t) => t.complete).length,
      numActive: todos.state.where((t) => !t.complete).length,
    );
  },
);
```

[Refer to stats file](lib/ui/pages/home_screen/stats_counter.dart)
```dart
  @override
  Widget build(BuildContext context) {
    return todosStats.rebuilder(
      () => Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Padding(
              padding: const EdgeInsets.only(bottom: 8.0),
              child: Text(
                i18n.of(context).completedTodos,
                style: Theme.of(context).textTheme.headline6,
              ),
            ),
            Padding(
              padding: const EdgeInsets.only(bottom: 24.0),
              child: Text(
                '${todosStats.state.numCompleted}',
                style: Theme.of(context).textTheme.subtitle1,
              ),
            ),
            Padding(
              padding: const EdgeInsets.only(bottom: 8.0),
              child: Text(
                i18n.of(context).activeTodos,
                style: Theme.of(context).textTheme.headline6,
              ),
            ),
            Padding(
              padding: const EdgeInsets.only(bottom: 24.0),
              child: Text(
                '${todosStats.state.numActive}',
                style: Theme.of(context).textTheme.subtitle1,
              ),
            )
          ],
        ),
      ),
    );
  }
```

<details>
  <summary>Click here to see stats test</summary>

[Refer to stats_test.dart file](test/stats_test.dart#L9)
```dart
void main() async {
  final storage = await RM.localStorageInitializerMock();
  setUp(() {
    storage.clear();
  });
  testWidgets('Show stats, and toggle completed', (tester) async {
    storage.store.addAll({'__Todos__': todos3});
    await tester.pumpWidget(App());
    //
    await tester.tap(find.byIcon(Icons.show_chart));
    await tester.pumpAndSettle();

    expect(find.byType(StatsCounter), findsOneWidget);
    expect(find.text('2'), findsOneWidget);
    expect(find.text('1'), findsOneWidget);

    //Top to toggle all to completed
    await tester.tap(find.byType(ExtraActionsButton));
    await tester.pumpAndSettle();
    await tester.tap(find.byKey(Key('__toggleAll__')));
    await tester.pumpAndSettle();

    //
    expect(find.text('3'), findsOneWidget);
    expect(find.text('0'), findsOneWidget);
    //
    //Toggle all to uncompleted
    await tester.tap(find.byType(ExtraActionsButton));
    await tester.pumpAndSettle();
    await tester.tap(find.byKey(Key('__toggleAll__')));
    await tester.pumpAndSettle();

    //Only active todos are displayed
    expect(find.text('0'), findsOneWidget);
    expect(find.text('3'), findsOneWidget);
  });

  testWidgets('Show stats, and clear completed', (tester) async {
    storage.store.addAll({'__Todos__': todos3});
    await tester.pumpWidget(App());
    //
    await tester.tap(find.byIcon(Icons.show_chart));
    await tester.pumpAndSettle();

    expect(find.byType(StatsCounter), findsOneWidget);
    expect(find.text('2'), findsOneWidget);
    expect(find.text('1'), findsOneWidget);

    //Top to clear completed
    await tester.tap(find.byType(ExtraActionsButton));
    await tester.pumpAndSettle();
    await tester.tap(find.byKey(Key('__toggleClearCompleted__')));
    await tester.pumpAndSettle();

    //one completed todo is removed. Remains to active todos
    expect(find.text('2'), findsOneWidget);
    expect(find.text('0'), findsOneWidget);
    //
    //Toggle all to completed
    await tester.tap(find.byType(ExtraActionsButton));
    await tester.pumpAndSettle();
    await tester.tap(find.byKey(Key('__toggleAll__')));
    await tester.pumpAndSettle();

    //Two completed todos
    expect(find.text('0'), findsOneWidget);
    expect(find.text('2'), findsOneWidget);

    await tester.tap(find.byType(ExtraActionsButton));
    await tester.pumpAndSettle();
    await tester.tap(find.byKey(Key('__toggleClearCompleted__')));
    await tester.pumpAndSettle();

    //all todos are removed
    expect(find.text('0'), findsNWidgets(2));
  });
}
```
</details>