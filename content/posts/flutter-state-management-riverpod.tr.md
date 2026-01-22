---
title: "Flutter'da State Management: Riverpod Rehberi"
date: 2025-01-15
tags:
  - Dart
  - Flutter
  - Mobile
description: "Flutter uygulamalarında Riverpod ile modern state management yaklaşımları."
---

Flutter uygulamalarında state management her zaman tartışmalı bir konu olmuştur. Provider, Bloc, GetX... Seçenekler çok fazla. Bu yazıda Riverpod'u inceleyeceğiz - Provider'ın yaratıcısı tarafından geliştirilen, daha güvenli ve esnek bir alternatif.

## Neden Riverpod?

Riverpod, Provider'ın bilinen sorunlarını çözmek için tasarlandı:

- **Compile-time güvenlik** - Runtime hatalarını minimuma indirir
- **BuildContext bağımsızlığı** - Provider'lara her yerden erişebilirsiniz
- **Test edilebilirlik** - Mock ve override işlemleri çok kolay
- **Kod üretimi** - riverpod_generator ile boilerplate azalır

## Kurulum

`pubspec.yaml` dosyasına ekleyin:

```yaml
dependencies:
  flutter_riverpod: ^2.4.9
  riverpod_annotation: ^2.3.3

dev_dependencies:
  riverpod_generator: ^2.3.9
  build_runner: ^2.4.8
```

## Temel Kavramlar

### Provider Türleri

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'providers.g.dart';

// Basit bir değer provider'ı
@riverpod
String appName(AppNameRef ref) {
  return 'My Flutter App';
}

// Async provider - API çağrıları için
@riverpod
Future<List<User>> users(UsersRef ref) async {
  final response = await http.get(Uri.parse('https://api.example.com/users'));
  final data = jsonDecode(response.body) as List;
  return data.map((json) => User.fromJson(json)).toList();
}

// Stream provider - gerçek zamanlı veriler için
@riverpod
Stream<int> counter(CounterRef ref) {
  return Stream.periodic(
    const Duration(seconds: 1),
    (count) => count,
  );
}
```

### Notifier - State Yönetimi

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'todo_notifier.g.dart';

@immutable
class Todo {
  final String id;
  final String title;
  final bool completed;

  const Todo({
    required this.id,
    required this.title,
    this.completed = false,
  });

  Todo copyWith({String? title, bool? completed}) {
    return Todo(
      id: id,
      title: title ?? this.title,
      completed: completed ?? this.completed,
    );
  }
}

@riverpod
class TodoList extends _$TodoList {
  @override
  List<Todo> build() {
    // Başlangıç state'i
    return [];
  }

  void addTodo(String title) {
    state = [
      ...state,
      Todo(
        id: DateTime.now().millisecondsSinceEpoch.toString(),
        title: title,
      ),
    ];
  }

  void toggleTodo(String id) {
    state = state.map((todo) {
      if (todo.id == id) {
        return todo.copyWith(completed: !todo.completed);
      }
      return todo;
    }).toList();
  }

  void removeTodo(String id) {
    state = state.where((todo) => todo.id != id).toList();
  }
}
```

## Widget'larda Kullanım

### ConsumerWidget

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class TodoScreen extends ConsumerWidget {
  const TodoScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todos = ref.watch(todoListProvider);
    final completedCount = todos.where((t) => t.completed).length;

    return Scaffold(
      appBar: AppBar(
        title: Text('Yapılacaklar ($completedCount/${todos.length})'),
      ),
      body: ListView.builder(
        itemCount: todos.length,
        itemBuilder: (context, index) {
          final todo = todos[index];
          return TodoTile(todo: todo);
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddDialog(context, ref),
        child: const Icon(Icons.add),
      ),
    );
  }

  void _showAddDialog(BuildContext context, WidgetRef ref) {
    final controller = TextEditingController();

    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Yeni Görev'),
        content: TextField(
          controller: controller,
          decoration: const InputDecoration(
            hintText: 'Görev adı...',
          ),
          autofocus: true,
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('İptal'),
          ),
          FilledButton(
            onPressed: () {
              if (controller.text.isNotEmpty) {
                ref.read(todoListProvider.notifier).addTodo(controller.text);
                Navigator.pop(context);
              }
            },
            child: const Text('Ekle'),
          ),
        ],
      ),
    );
  }
}

class TodoTile extends ConsumerWidget {
  final Todo todo;

  const TodoTile({super.key, required this.todo});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Dismissible(
      key: Key(todo.id),
      onDismissed: (_) {
        ref.read(todoListProvider.notifier).removeTodo(todo.id);
      },
      background: Container(
        color: Colors.red,
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 16),
        child: const Icon(Icons.delete, color: Colors.white),
      ),
      child: ListTile(
        leading: Checkbox(
          value: todo.completed,
          onChanged: (_) {
            ref.read(todoListProvider.notifier).toggleTodo(todo.id);
          },
        ),
        title: Text(
          todo.title,
          style: TextStyle(
            decoration: todo.completed
                ? TextDecoration.lineThrough
                : TextDecoration.none,
          ),
        ),
      ),
    );
  }
}
```

### AsyncValue ile Yükleme Durumları

```dart
class UserListScreen extends ConsumerWidget {
  const UserListScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final usersAsync = ref.watch(usersProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Kullanıcılar')),
      body: usersAsync.when(
        loading: () => const Center(
          child: CircularProgressIndicator(),
        ),
        error: (error, stack) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.error, size: 48, color: Colors.red),
              const SizedBox(height: 16),
              Text('Hata: $error'),
              const SizedBox(height: 16),
              FilledButton(
                onPressed: () => ref.invalidate(usersProvider),
                child: const Text('Tekrar Dene'),
              ),
            ],
          ),
        ),
        data: (users) => RefreshIndicator(
          onRefresh: () => ref.refresh(usersProvider.future),
          child: ListView.builder(
            itemCount: users.length,
            itemBuilder: (context, index) {
              final user = users[index];
              return ListTile(
                leading: CircleAvatar(child: Text(user.name[0])),
                title: Text(user.name),
                subtitle: Text(user.email),
              );
            },
          ),
        ),
      ),
    );
  }
}
```

## Provider'lar Arası İletişim

```dart
@riverpod
class FilterType extends _$FilterType {
  @override
  TodoFilter build() => TodoFilter.all;

  void setFilter(TodoFilter filter) {
    state = filter;
  }
}

enum TodoFilter { all, active, completed }

@riverpod
List<Todo> filteredTodos(FilteredTodosRef ref) {
  final todos = ref.watch(todoListProvider);
  final filter = ref.watch(filterTypeProvider);

  switch (filter) {
    case TodoFilter.all:
      return todos;
    case TodoFilter.active:
      return todos.where((t) => !t.completed).toList();
    case TodoFilter.completed:
      return todos.where((t) => t.completed).toList();
  }
}
```

## Test Yazma

Riverpod ile test yazmak çok kolay:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:riverpod/riverpod.dart';

void main() {
  group('TodoList', () {
    test('başlangıçta boş olmalı', () {
      final container = ProviderContainer();
      addTearDown(container.dispose);

      expect(container.read(todoListProvider), isEmpty);
    });

    test('todo eklenebilmeli', () {
      final container = ProviderContainer();
      addTearDown(container.dispose);

      container.read(todoListProvider.notifier).addTodo('Test görevi');

      final todos = container.read(todoListProvider);
      expect(todos.length, 1);
      expect(todos.first.title, 'Test görevi');
      expect(todos.first.completed, false);
    });

    test('todo tamamlanabilmeli', () {
      final container = ProviderContainer();
      addTearDown(container.dispose);

      container.read(todoListProvider.notifier).addTodo('Test');
      final todoId = container.read(todoListProvider).first.id;

      container.read(todoListProvider.notifier).toggleTodo(todoId);

      expect(container.read(todoListProvider).first.completed, true);
    });
  });
}
```

## Sonuç

Riverpod, Flutter'da state management için güçlü ve modern bir çözüm sunuyor. Özellikle:

- **Tip güvenliği** sayesinde hataları erken yakalarsınız
- **Kod üretici** ile boilerplate kodu minimuma inersiniz
- **Test edilebilirlik** ile kaliteli kod yazarsınız

Provider'dan geçiş yapmayı düşünüyorsanız, Riverpod kesinlikle değerlendirilmesi gereken bir seçenek.
