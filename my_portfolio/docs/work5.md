### 🧩 Работа 5: Паттерн «Декоратор» для экспорта курсов валют

В этой работе реализован классический структурный паттерн **Декоратор** на примере получения и экспорта курсов валют с сайта Центрального Банка России.  
Базовый компонент загружает данные в формате JSON, а декораторы добавляют возможность сохранять их в виде YAML и CSV, не изменяя исходный код основной логики.

---

## 📌 Задача

Необходимо разработать систему, которая:

1. Получает актуальные курсы валют с публичного API ЦБ РФ.
2. Предоставляет возможность экспортировать полученные данные в трёх форматах: **JSON**, **YAML**, **CSV**.
3. Реализует расширение функциональности без изменения существующего кода — за счёт паттерна **Декоратор**.
4. Использует единый интерфейс (`Component`) для всех компонентов и декораторов.

---

## 🧠 Реализация

### 🔷 Базовый интерфейс `Component`

Абстрактный класс, задающий единый контракт для всех компонентов и декораторов:

```python
class Component(ABC):
    @abstractmethod
    def values_return(self) -> str:
        """Возвращает данные в виде строки (формат JSON по умолчанию)."""
        pass

    @abstractmethod
    def export_to_file(self, filename: Optional[str] = None) -> None:
        """Экспортирует данные в файл."""
        pass
```

### 🔷 Конкретный компонент `ConcreteComponent`

Этот класс выполняет основную работу: загружает курсы валют и возвращает их в виде отформатированной JSON-строки.

- **`_fetch_data()`** отправляет GET-запрос к API ЦБ РФ (`https://www.cbr-xml-daily.ru/daily_json.js`) и возвращает ответ в виде словаря Python.
- **`values_return()`** преобразует полученный словарь в красивую JSON-строку с отступами (`indent=4`).
- **`export_to_file()`** сохраняет эту JSON-строку в файл (по умолчанию `output.json`).

```python
class ConcreteComponent(Component):
    def __init__(self, filename: str = "output.json") -> None:
        self._default_filename = filename

    def _fetch_data(self) -> Dict[str, Any]:
        response = requests.get(
            "https://www.cbr-xml-daily.ru/daily_json.js",
            timeout=10
        )
        response.raise_for_status()
        return response.json()

    def values_return(self) -> str:
        data = self._fetch_data()
        return json.dumps(data, indent=4, ensure_ascii=False)

    def export_to_file(self, filename: Optional[str] = None) -> None:
        if filename is None:
            filename = self._default_filename
        content = self.values_return()
        with open(filename, "w", encoding="utf-8") as f:
            f.write(content)
```

### 🔷 Базовый декоратор `Decorator`

Базовый класс для всех декораторов. Он хранит ссылку на обёрнутый компонент и делегирует ему вызовы методов. Благодаря этому каждый конкретный декоратор может изменять только ту часть поведения, которая ему нужна.

```python
class Decorator(Component):
    def __init__(self, component: Component) -> None:
        self._component = component

    def values_return(self) -> str:
        return self._component.values_return()

    def export_to_file(self, filename: Optional[str] = None) -> None:
        self._component.export_to_file(filename)
```

### 🔷 Декоратор `YAMLDecorator`

Этот декоратор преобразует JSON-строку, полученную от обёрнутого компонента, в формат YAML.

- **`values_return()`** загружает JSON из компонента и с помощью библиотеки `yaml.dump()` преобразует его в YAML-строку.
- **`export_to_file()`** сохраняет полученную YAML-строку в файл (по умолчанию `output.yaml`).

```python
class YAMLDecorator(Decorator):
    def values_return(self) -> str:
        json_str = self._component.values_return()
        data = json.loads(json_str)
        return yaml.dump(data, allow_unicode=True, sort_keys=False)

    def export_to_file(self, filename: str = "output.yaml") -> None:
        content = self.values_return()
        with open(filename, "w", encoding="utf-8") as f:
            f.write(content)
```

### 🔷 Декоратор `CSVDecorator`

Этот декоратор преобразует исходные данные в плоский CSV-файл формата «ключ → значение».

- **`values_return()`** загружает JSON, обходит все пары ключ-значение исходного словаря и формирует строки CSV. Если значение является словарём или списком, оно сериализуется в JSON-строку.
- Полученные строки записываются в объект `StringIO`, который затем возвращается в виде строки.
- **`export_to_file()`** сохраняет результат в файл (по умолчанию `output.csv`).

```python
class CSVDecorator(Decorator):
    def values_return(self) -> str:
        json_str = self._component.values_return()
        data = json.loads(json_str)

        rows = []
        for key, value in data.items():
            if isinstance(value, (dict, list)):
                value = json.dumps(value, ensure_ascii=False)
            rows.append([key, value])

        output = io.StringIO()
        writer = csv.writer(output)
        writer.writerows(rows)
        return output.getvalue()

    def export_to_file(self, filename: str = "output.csv") -> None:
        content = self.values_return()
        with open(filename, "w", newline="", encoding="utf-8") as f:
            f.write(content)
```

---

## 🧪 Использование

Клиентский код работает с любым компонентом через единый интерфейс `Component`:

```python
def client_code(component: Component, description: str) -> None:
    print(description)
    print(component.values_return())
    component.export_to_file()
    print("Файл сохранён.\n")
```

Пример создания и использования:

```python
if __name__ == "__main__":
    base = ConcreteComponent()
    client_code(base, "JSON")

    yaml_decorated = YAMLDecorator(base)
    client_code(yaml_decorated, "YAML")

    csv_decorated = CSVDecorator(base)
    client_code(csv_decorated, "CSV")
```

При выполнении программа:

- Загрузит актуальные курсы валют с сайта ЦБ РФ.
- Сохранит их в трёх файлах: `output.json`, `output.yaml`, `output.csv`.
- Выведет в консоль содержимое каждого файла и сообщение об успешном сохранении.

---

## 📊 Анализ решения

### Достоинства

- **Гибкость** – новый формат экспорта можно добавить, создав ещё один декоратор, без изменения существующих классов.
- **Соблюдение принципа Open/Closed** – классы открыты для расширения, но закрыты для модификации.
- **Единый интерфейс** – клиентский код работает с любым декорированным компонентом одинаково.
- **Переиспользование** – базовый компонент не зависит от форматов вывода, декораторы можно комбинировать.

### Особенности реализации

- **JSON → YAML** преобразование происходит через промежуточную загрузку в словарь Python (`json.loads`).
- **CSV** получается плоским: вложенные структуры превращаются в JSON-строки, что гарантирует корректную запись.
- **Обработка ошибок** оставлена на усмотрение вызывающего кода: сетевые ошибки и ошибки парсинга пробрасываются наверх.

### Возможные улучшения

- Добавить обработку ошибок сети и повторные попытки запроса.
- Реализовать комбинирование декораторов (например, сначала сжатие, потом шифрование).
- Добавить поддержку других форматов (XML, Excel).

---

## 📌 Заключение

В работе успешно реализован паттерн «Декоратор» для расширения функциональности выгрузки курсов валют. Базовый компонент выполняет загрузку данных, а декораторы добавляют возможность сохранения в YAML и CSV без изменения исходного кода. Такое архитектурное решение обеспечивает гибкость, упрощает добавление новых форматов и соответствует принципам SOLID.