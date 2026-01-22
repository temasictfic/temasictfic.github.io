---
title: "Angular Signals: A New Era of Reactivity"
date: 2025-01-20
tags:
  - TypeScript
  - Angular
  - Frontend
description: "Exploring Angular's new signals API for fine-grained reactivity and improved performance."
---

Angular 16 introduced Signals, a new reactive primitive that changes how we think about state management in Angular applications. Let's explore what signals are and how to use them effectively.

## What Are Signals?

Signals are a new way to handle reactive state in Angular. Unlike RxJS Observables, signals are synchronous and provide fine-grained reactivity updates.

```typescript
import { signal, computed, effect } from '@angular/core';

// Create a signal with an initial value
const count = signal(0);

// Read the signal value
console.log(count()); // 0

// Update the signal
count.set(5);
count.update(value => value + 1);
```

## Computed Signals

Computed signals derive their value from other signals and automatically update when dependencies change:

```typescript
import { signal, computed } from '@angular/core';

const firstName = signal('John');
const lastName = signal('Doe');

// Computed signal - automatically updates when firstName or lastName changes
const fullName = computed(() => `${firstName()} ${lastName()}`);

console.log(fullName()); // "John Doe"

firstName.set('Jane');
console.log(fullName()); // "Jane Doe"
```

## Effects for Side Effects

Effects run whenever their signal dependencies change:

```typescript
import { signal, effect } from '@angular/core';

const theme = signal<'light' | 'dark'>('dark');

// Effect runs whenever theme changes
effect(() => {
  document.body.classList.toggle('dark-mode', theme() === 'dark');
  console.log(`Theme changed to: ${theme()}`);
});

// This triggers the effect
theme.set('light');
```

## Using Signals in Components

Here's a complete component example using signals:

```typescript
import { Component, signal, computed } from '@angular/core';

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

@Component({
  selector: 'app-todo-list',
  template: `
    <div class="todo-app">
      <h2>Todo List</h2>

      <div class="stats">
        <span>Total: {{ todos().length }}</span>
        <span>Completed: {{ completedCount() }}</span>
        <span>Remaining: {{ remainingCount() }}</span>
      </div>

      <input
        #newTodo
        (keyup.enter)="addTodo(newTodo.value); newTodo.value = ''"
        placeholder="Add a new todo..."
      />

      <ul>
        @for (todo of todos(); track todo.id) {
          <li [class.completed]="todo.completed">
            <input
              type="checkbox"
              [checked]="todo.completed"
              (change)="toggleTodo(todo.id)"
            />
            {{ todo.text }}
            <button (click)="removeTodo(todo.id)">×</button>
          </li>
        }
      </ul>
    </div>
  `
})
export class TodoListComponent {
  // State as signals
  todos = signal<Todo[]>([
    { id: 1, text: 'Learn Angular Signals', completed: false },
    { id: 2, text: 'Build something cool', completed: false }
  ]);

  // Computed values
  completedCount = computed(() =>
    this.todos().filter(t => t.completed).length
  );

  remainingCount = computed(() =>
    this.todos().filter(t => !t.completed).length
  );

  // Actions
  addTodo(text: string) {
    if (!text.trim()) return;

    this.todos.update(todos => [
      ...todos,
      { id: Date.now(), text: text.trim(), completed: false }
    ]);
  }

  toggleTodo(id: number) {
    this.todos.update(todos =>
      todos.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  }

  removeTodo(id: number) {
    this.todos.update(todos => todos.filter(t => t.id !== id));
  }
}
```

## Signal Inputs (Angular 17.1+)

Angular 17.1 introduced signal-based inputs:

```typescript
import { Component, input, computed } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: `<h1>{{ greeting() }}</h1>`
})
export class GreetingComponent {
  // Signal input with default value
  name = input('World');

  // Required signal input
  title = input.required<string>();

  // Computed from inputs
  greeting = computed(() => `${this.title()}, ${this.name()}!`);
}
```

## When to Use Signals vs RxJS

| Use Case | Signals | RxJS |
|----------|---------|------|
| Component state | ✅ | ⚠️ |
| Derived values | ✅ | ✅ |
| HTTP requests | ⚠️ | ✅ |
| Complex async flows | ❌ | ✅ |
| Event streams | ⚠️ | ✅ |

## Conclusion

Signals provide a simpler mental model for reactive state in Angular. They're synchronous, easy to debug, and integrate seamlessly with Angular's change detection. While RxJS remains valuable for complex async operations, signals are the future of local component state management in Angular.

The key benefits:
- **Simpler API** - No subscriptions to manage
- **Better performance** - Fine-grained updates
- **Easier debugging** - Synchronous and predictable
- **TypeScript-first** - Full type inference
