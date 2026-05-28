# Работа 2: «Функция деления с заданной точностью»

### 📝 Постановка задачи
Разработать функцию `calculate`, которая выполняет деление первого операнда на второй с заданной точностью (epsilon). В рамках работы также необходимо реализовать функцию `load_params` для чтения значения точности из конфигурационного файла, а также покрыть функционал тестами.

Основные требования задачи:
1.  **Функция `calculate`**:
    *   Принимает два операнда (целые или дробные) и точность `epsilon` (ключевой аргумент со значением по умолчанию `0.0001`).
    *   Точность `epsilon` должна находиться в диапазоне `10**-1 < epsilon < 10**-9`.
    *   Должна быть обработка деления на ноль.
2.  **Функция `load_params`**:
    *   Считывает значение точности из файла конфигурации `settings.ini`.
    *   Устанавливает считанное значение в качестве точности для функции `calculate`.
3.  **Тестирование**:
    *   Написать тесты для `calculate` (обычное деление, деление малых чисел, деление на ноль, проверка границ `epsilon`).
    *   Написать тесты для `load_params` (проверка открытия файла, корректность `epsilon`, обработка некорректного формата).

---

### 💻 Исходный код

Код решения разделён на несколько файлов для лучшей организации.

#### **1. Файл исключений (`exceptions.py`)**
Определяет собственные исключения для проекта, что повышает читаемость и упрощает отладку.

```python
class DivisionWithEpsException(Exception):
    def __str__(self):
        return "General Exception for Division with Epsilon Project"

class SettingsFileNotFoundError(DivisionWithEpsException):
    def __str__(self):
        return "Отсутствует конфигурационный файл"

class EpsilonKeyNotFound(DivisionWithEpsException):
    def __str__(self):
        return "Отсутствует ключ epsilon"
```
*(Источник: `exceptions.py`)*

#### **2. Основная логика (`calc.py`)**
Содержит функции вычисления и загрузки параметров, а также несколько строк для демонстрации работы.

```python
import os
import exceptions

def calculate(devidend, divider, epsilon=0.0001):
    """Выполняет деление с заданной точностью."""
    if not (10**-9 < epsilon < 10**-1):
        raise ValueError("Точность должна быть в диапазоне от 10^-1 до 10^-9")
    if divider == 0:
        raise ZeroDivisionError("Деление на ноль невозможно")

    # ... (логика округления с учетом научной нотации)
    return round(devidend / divider, len(str(epsilon)) - 2 if '.' in str(epsilon) else int(str(epsilon).split('e-')[1]))

def load_params(config_file='C:/Codes/p/3 sem/lab2/settings.ini'):
    """Считывает значение точности из конфигурационного файла."""
    if not os.path.exists(config_file):
        raise exceptions.SettingsFileNotFoundError()

    try:
        with open(config_file, 'r', encoding='utf-8') as file:
            content = file.read()
            for line in content.split('\n'):
                line = line.strip()
                if not line or line.startswith('#') or line.startswith(';'):
                    continue
                if '=' in line:
                    key, value = line.split('=', 1)
                    key, value = key.strip(), value.strip()
                    if key == 'epsilon':
                        epsilon = float(value)
                        if not (10**-9 < epsilon < 10**-1):
                            raise ValueError("Точность должна быть в диапазоне от 10^-1 до 10^-9")
                        return epsilon
        raise exceptions.EpsilonKeyNotFound()
    except ValueError:
        raise ValueError("Некорректный формат числа в конфигурационном файле")

# Демонстрация работы
print(calculate(10, 3, load_params()))  # Вывод: 3.333
print(10**-11)
```
*(Источник: `calc.py`)*

#### **3. Конфигурационный файл (`settings.ini`)**
Хранит настройки точности в простом текстовом формате.

```ini
; Конфигурационный файл для настройки точности вычислений
; Формат: ключ=значение
epsilon=0.001
```
*(Источник: `settings.ini`)*

#### **4. Тесты (`test_calc_funcs.py`)**
Используют фреймворк `unittest` для проверки всех сценаев использования.

```python
import unittest
import os
import calc
from exceptions import SettingsFileNotFoundError, EpsilonKeyNotFound

class TestCalculateFunction(unittest.TestCase):
    def test_division_normal_case(self):
        self.assertEqual(calc.calculate(1, 2, epsilon=0.01), 0.5)

    def test_division_small_numbers(self):
        self.assertEqual(calc.calculate(1, 1000, epsilon=0.001), 0.001)

    def test_division_by_zero(self):
        with self.assertRaises(ZeroDivisionError):
            calc.calculate(1, 0)

    # ... другие тесты

class TestLoadParamsFunction(unittest.TestCase):
    def test_valid_config_file(self):
        # Создание временного файла и проверка загрузки
        with open(self.test_config, 'w', encoding='utf-8') as f:
            f.write("epsilon=0.01\n")
        epsilon = calc.load_params(self.test_config)
        self.assertEqual(epsilon, 0.01)

    def test_file_not_found(self):
        with self.assertRaises(SettingsFileNotFoundError):
            calc.load_params('non_existent_file.ini')

    # ... другие тесты
```
*(Источник: `test_calc_funcs.py`)*

---

### 🧪 Тестирование и результаты

Разработанный тестовый набор покрывает как основные функции, так и граничные случаи, включая работу с файлами.

| **Тестовый класс** | **Тестовая функция** | **Проверяемый сценарий** | **Результат** |
| :--- | :--- | :--- | :--- |
| `TestCalculateFunction` | `test_division_normal_case` | Обычное деление (1/2, eps=0.01) | ✅ Успешно |
| | `test_division_small_numbers` | Деление малых чисел (1/1000, eps=0.001) | ✅ Успешно |
| | `test_division_by_zero` | Деление на ноль | ✅ Вызвано исключение |
| | `test_epsilon_out_of_range` | `epsilon` вне диапазона | ✅ Вызвано исключение |
| `TestLoadParamsFunction` | `test_valid_config_file` | Чтение корректного конфиг-файла | ✅ Успешно |
| | `test_file_not_found` | Отсутствие конфиг-файла | ✅ Вызвано исключение |
| | `test_epsilon_out_of_range_in_file` | Значение `epsilon` в файле вне диапазона | ✅ Вызвано исключение |
| | `test_invalid_number_format` | Некорректный формат числа в файле | ✅ Вызвано исключение |
| | `test_missing_epsilon` | Отсутствие ключа `epsilon` в файле | ✅ Вызвано исключение |

---

### 📊 Анализ и выводы

**Что получилось реализовать:**

1.  **Полностью работоспособное ядро приложения:** Функции `calculate` и `load_params` выполняют все поставленные задачи.
2.  **Надёжную обработку ошибок:** Реализована проверка всех возможных исключительных ситуаций (деление на ноль, выход `epsilon` за пределы диапазона, проблемы с чтением или форматированием конфигурационного файла). Благодаря кастомным исключениям в `exceptions.py`, код легко читать и отлаживать.
3.  **Тестовое покрытие:** Написанные тесты покрывают ключевые аспекты работы и могут служить документацией к использованию функций.

**Какие были трудности:**

1.  **Округление чисел в научной нотации:** Основная сложность возникла с определением количества знаков для округления, когда `epsilon` очень маленький (например, `1e-10`). Стандартный способ через `len(str(epsilon)) - 2` не работает из-за преобразования в строку с научной нотацией (`1e-10`). Автор решил эту проблему, добавив проверку на наличие точки в строке и, при её отсутствии, обрабатывал научную нотацию, выделяя показатель степени.
2.  **Абсолютный путь к файлу:** Путь `C:/Codes/p/3 sem/lab2/settings.ini` жёстко задан в коде, что делает его немобильным. Лучше было бы использовать относительные пути или передавать путь как аргумент командной строки.

**Что нового узнали:**

*   Углублено понимание работы с плавающей точкой в Python и сложностей, связанных с их строковым представлением.
*   Получен опыт в проектировании собственной иерархии исключений для предметной области.
*   Освоены методы тестирования кода, работающего с файловой системой (создание/удаление временных файлов в тестах).

**Заключение:**

Поставленная задача выполнена в полном объёме. Несмотря на некоторые архитектурные недочёты (например, жёстко прописанный путь), основная функциональность работает стабильно и корректно обрабатывает ошибки.
