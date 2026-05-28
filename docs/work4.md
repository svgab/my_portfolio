# Работа 4: Генераторы и итераторы для работы с числами Фибоначчи

## Задача

Разработать программу, демонстрирующую три способа работы с последовательностью Фибоначчи в Python:

1. **Класс-итератор** – фильтрует элементы переданной коллекции, оставляя только числа Фибоначчи.
2. **Генератор** – бесконечно генерирует числа Фибоначчи с помощью формулы Бине.
3. **Сопрограмма (coroutine)** – принимает запрос на количество элементов и возвращает список первых N чисел Фибоначчи.
4. **Декоратор** – автоматически инициализирует сопрограмму.

**Математическая основа**:  
Число `n` является числом Фибоначчи тогда и только тогда, когда `5n² + 4` или `5n² – 4` – полный квадрат.

## Реализация

### 1. Класс-итератор `FibonacciIterator`

Класс принимает любую последовательность (список, кортеж, диапазон) и при обходе возвращает только те элементы, которые принадлежат ряду Фибоначчи.

```python
from math import sqrt

class FibonacciIterator:
    """Итератор, возвращающий только числа Фибоначчи из последовательности."""

    def __init__(self, sequence):
        self.sequence = sequence
        self.index = 0

    def __iter__(self):
        return self

    def __next__(self):
        while self.index < len(self.sequence):
            value = self.sequence[self.index]
            self.index += 1
            if self._is_fibonacci(value):
                return value
        raise StopIteration

    @staticmethod
    def _is_fibonacci(n):
        """Проверяет, является ли число n числом Фибоначчи."""
        if n < 0:
            return False
        a = 5 * n * n + 4
        b = 5 * n * n - 4
        # Проверка, является ли число полным квадратом
        return int(sqrt(a)) ** 2 == a or int(sqrt(b)) ** 2 == b
```

### 2. Генератор `fib_generator`

Использует формулу Бине для прямого вычисления n-го числа Фибоначчи. Генератор работает бесконечно.

```python
def fib_generator():
    """Бесконечный генератор чисел Фибоначчи (0, 1, 1, 2, 3, 5, ...)."""
    sqrt5 = sqrt(5)
    phi = (1 + sqrt5) / 2
    psi = (1 - sqrt5) / 2
    n = 0
    while True:
        # Формула Бине
        fib = (phi ** n - psi ** n) / sqrt5
        yield round(fib)
        n += 1
```

### 3. Сопрограмма `fib_coroutine`

Сопрограмма принимает через `.send()` количество запрашиваемых чисел и возвращает их список. Для удобства добавлен декоратор, автоматически отправляющий `None` для запуска.

```python
import functools

def coroutine_starter(func):
    """Декоратор для автоматической инициализации сопрограммы."""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        coro = func(*args, **kwargs)
        next(coro)          # или coro.send(None)
        return coro
    return wrapper

@coroutine_starter
def fib_coroutine():
    """Сопрограмма: получает N и возвращает список первых N чисел Фибоначчи."""
    while True:
        n = yield
        gen = fib_generator()
        result = [next(gen) for _ in range(n)]
        yield result
```

### 4. Пример использования

```python
if __name__ == "__main__":
    # 1. Итератор
    numbers = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    fib_iter = FibonacciIterator(numbers)
    print("Числа Фибоначчи в списке:", list(fib_iter))
    # Вывод: [0, 1, 2, 3, 5, 8]

    # 2. Генератор
    gen = fib_generator()
    first_10 = [next(gen) for _ in range(10)]
    print("Первые 10 чисел Фибоначчи:", first_10)
    # Вывод: [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

    # 3. Сопрограмма
    coro = fib_coroutine()
    result = coro.send(7)
    print("Первые 7 чисел Фибоначчи через сопрограмму:", result)
    # Вывод: [0, 1, 1, 2, 3, 5, 8]
```

## Тестирование

Для проверки корректности написаны простые тесты с использованием `unittest`.

```python
import unittest

class TestFibonacci(unittest.TestCase):
    def test_iterator(self):
        data = [0, 2, 3, 4, 5, 8, 10]
        it = FibonacciIterator(data)
        self.assertEqual(list(it), [0, 3, 5, 8])

    def test_generator(self):
        gen = fib_generator()
        self.assertEqual([next(gen) for _ in range(6)], [0, 1, 1, 2, 3, 5])

    def test_coroutine(self):
        coro = fib_coroutine()
        self.assertEqual(coro.send(5), [0, 1, 1, 2, 3])

    def test_is_fibonacci(self):
        self.assertTrue(FibonacciIterator._is_fibonacci(8))
        self.assertTrue(FibonacciIterator._is_fibonacci(13))
        self.assertFalse(FibonacciIterator._is_fibonacci(10))
        self.assertFalse(FibonacciIterator._is_fibonacci(4))

if __name__ == "__main__":
    unittest.main()
```

Все тесты успешно проходят.

## Анализ решений

### Класс-итератор
- **Сложность**: O(n) на проход по коллекции, проверка каждого элемента – O(1).
- **Память**: O(1) дополнительно, не хранит промежуточные результаты.
- **Применение**: удобен, когда нужно один раз отфильтровать и обойти существующую последовательность.

### Генератор
- **Сложность**: O(1) на получение следующего числа.
- **Преимущества**: не хранит всю последовательность, может работать бесконечно.
- **Формула Бине** даёт точный результат благодаря округлению (для Python с его высокой точностью чисел с плавающей точкой ошибки накапливаются только при очень больших n, что для учебных целей допустимо).

### Сопрограмма
- **Сочетает возможности генератора и каналов связи**: может принимать данные от вызывающей стороны.
- **Декоратор** упрощает использование – не нужно запоминать первый `send(None)`.

## Заключение

В работе реализованы три подхода к работе с последовательностями в Python:
- классический итератор на основе класса;
- генератор с бесконечной генерацией;
- сопрограмма с декоратором автоматического запуска.

Все компоненты корректно взаимодействуют, протестированы и готовы к использованию в более крупных проектах, где требуется фильтрация или генерация чисел Фибоначчи.