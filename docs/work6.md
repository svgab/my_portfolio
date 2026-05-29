# Работа 6. Паттерн «Одиночка» (Singleton) для получения курсов валют

## Задача

Реализовать класс для получения актуальных курсов валют с сайта Центрального Банка России с использованием Singleton
## Реализация

### 1. Метакласс `SingletonMeta`

Гарантирует, что класс имеет единственный экземпляр даже в многопоточной среде.

```python
import threading
from typing import Dict, Any

class SingletonMeta(type):
    _instances: Dict[type, Any] = {}
    _lock: threading.Lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            with cls._lock:
                if cls not in cls._instances:
                    instance = super().__call__(*args, **kwargs)
                    cls._instances[cls] = instance
        return cls._instances[cls]
```

- Двойная проверка с блокировкой обеспечивает потокобезопасность.
- Экземпляры хранятся в словаре по классу.

### 2. Класс `CurrencyRate`

Основной класс, использующий метакласс `SingletonMeta`.

```python
class CurrencyRate(metaclass=SingletonMeta):
    def __init__(self, min_interval: float = 1.0) -> None:
        self.url = "https://www.cbr.ru/scripts/XML_daily.asp"
        self.min_interval = min_interval
        self._last_request_time: float = 0.0
        self._cache: Optional[Dict[str, Tuple[str, Tuple[str, str]]]] = None
        self._lock = threading.Lock()
```

- `min_interval` – минимальное время между реальными запросами к API.
- `_cache` хранит словарь: `{код_валюты: (название, (целая_часть, дробная_часть))}`.

#### 2.1 Получение и парсинг данных

```python
def _fetch_and_parse(self) -> Dict[str, Tuple[str, Tuple[str, str]]]:
    response = requests.get(self.url, timeout=10)
    response.raise_for_status()
    root = ET.fromstring(response.content)

    result = {}
    for valute in root.findall("Valute"):
        char_code = valute.find("CharCode").text
        name = valute.find("Name").text
        value_str = valute.find("Value").text      # например "36,1234"

        if "," in value_str:
            int_part, frac_part = value_str.split(",")
        else:
            int_part, frac_part = value_str, "0"

        result[char_code] = (name, (int_part, frac_part))

    return result
```

- Используется `xml.etree.ElementTree` для парсинга.
- Значение разделяется на целую и дробную части **как строки** – требование задачи.

#### 2.2 Логика кэширования

```python
def _should_refresh(self) -> bool:
    if self._cache is None:
        return True
    elapsed = time.time() - self._last_request_time
    return elapsed >= self.min_interval
```

Данные обновляются, если кэш пуст или прошло достаточно времени.

#### 2.3 Публичный метод `get_currencies`

```python
def get_currencies(self, currencies_ids_lst: List[str]) -> List[Optional[Tuple[str, Tuple[str, str]]]]:
    # Проверка типа аргумента
    if not isinstance(currencies_ids_lst, list):
        raise TypeError(...)

    with self._lock:
        if self._should_refresh():
            self._cache = self._fetch_and_parse()
            self._last_request_time = time.time()

        if self._cache is None:
            return [None] * len(currencies_ids_lst)

        return [self._cache.get(curr_id) for curr_id in currencies_ids_lst]
```

- Возвращает список результатов в том же порядке, что и запрошенные коды. Для отсутствующей валюты – `None`.
- Блокировка `self._lock` предотвращает одновременное обновление кэша из разных потоков.

#### 2.4 Визуализация

```python
def visualize_currencies(self, currencies_ids_lst: List[str]) -> None:
    data = self.get_currencies(currencies_ids_lst)
    currencies, values = [], []
    for curr_id, info in zip(currencies_ids_lst, data):
        if info is None:
            continue
        name, (int_part, frac_part) = info
        numeric_value = float(int_part + "." + frac_part)
        currencies.append(f"{curr_id}\n({name})")
        values.append(numeric_value)

    # Построение столбчатой диаграммы через matplotlib
    plt.figure(figsize=(10, 6))
    bars = plt.bar(currencies, values, color="skyblue")
    plt.title("Currency Exchange Rates (RUB per unit)")
    plt.ylabel("Rate (RUB)")
    plt.xticks(rotation=45, ha="right")
    plt.tight_layout()
    for bar, val in zip(bars, values):
        plt.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.5,
                 f"{val:.4f}", ha="center", va="bottom", fontsize=8)
    plt.savefig("currencies.jpg", dpi=150)
    plt.close()
    print("График сохранён как currencies.jpg")
```

- Для каждой валюты преобразует строки целой и дробной части в число с плавающей точкой.
- Строит столбчатую диаграмму с подписями значений.
- Сохраняет в файл `currencies.jpg`.

### 3. Тестирование и демонстрация

В функции `main()` реализованы простые проверки:

```python
def main():
    # Singleton
    rate1 = CurrencyRate()
    rate2 = CurrencyRate()
    assert rate1 is rate2

    # Неверный код
    assert rate1.get_currencies(["XYZ"])[0] is None

    # Корректность USD
    result_usd = rate1.get_currencies(["USD"])[0]
    assert result_usd is not None
    name, (int_part, frac_part) = result_usd
    assert int_part.isdigit() and frac_part.isdigit()
    value_float = float(int_part + "." + frac_part)
    assert 50 <= value_float <= 150   # разумный диапазон

    # Rate limiting
    start = time.time()
    _ = rate1.get_currencies(["EUR"])
    elapsed = time.time() - start
    assert elapsed < 0.1   # данные должны взяться из кэша

    # Визуализация
    rate1.visualize_currencies(["USD", "EUR", "GBP", "CNY", "JPY"])
```

При успешном прохождении выводится `Все тесты пройдены успешно!` и создаётся файл `currencies.jpg`.

## Анализ решения

### Использованные возможности Python

- **Метаклассы** – для реализации Singleton на низком уровне, что гарантирует единственность экземпляра даже при наследовании.
- **Типизация** (`typing`) – явно указаны типы аргументов и возвращаемых значений, что улучшает читаемость и помогает IDE.
- **Модуль `threading`** – для блокировок при работе с кэшем.
- **Библиотеки**: `requests` (HTTP-запросы), `xml.etree.ElementTree` (парсинг XML), `matplotlib` (визуализация).

### Соответствие требованиям

| Требование | Реализовано |
|------------|--------------|
| Singleton через метакласс | ✅ `SingletonMeta` |
| Потокобезопасность | ✅ блокировка при создании экземпляра и при обновлении кэша |
| Контроль частоты запросов | ✅ `min_interval` и `_should_refresh()` |
| Парсинг XML | ✅ `ET.fromstring` и поиск по тегам |
| Хранение курса как кортеж (целая, дробная) | ✅ разделение по запятой |
| Визуализация | ✅ столбчатая диаграмма, сохранение в JPG |
| Тесты | ✅ встроены в `main()` |

### Возможные улучшения

- **Обработка ошибок сети** – сейчас при сбое запроса программа упадёт с исключением. Можно добавить повторные попытки или возврат старого кэша.
- **Логирование** – вместо `print` использовать модуль `logging`.
- **Расширяемость** – поддержка других источников данных (например, JSON API) через стратегию.
- **Параметризация имени файла** – передавать имя выходного файла для графика.

### Плюсы Singleton в данном контексте

- Экономия ресурсов – данные загружаются один раз и используются повторно.
- Глобальная точка доступа к курсам – удобно для приложений, которым нужны актуальные курсы в разных частях кода.
- Кэширование с ограничением частоты запросов предотвращает перегрузку сервера ЦБ РФ.

### Минусы

- Singleton иногда считается анти-паттерном из-за скрытых зависимостей и затруднения тестирования. Однако для учебной задачи его применение оправдано.

Код написан в соответствии с PEP-8, снабжён докстрингами и аннотациями типов. Он готов к использованию как часть более крупного проекта или для демонстрации принципов ООП и паттернов проектирования.Раб
