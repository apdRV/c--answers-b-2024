## Билет 82. Вспомогательные инструменты метапрограммирования

#### `std::apply` для вызова функции с параметрами из кортежа

`std::apply` позволяет вызывать функцию, аргументами функции станут элементы кортежа, переданного в `std::apply`.
Никакой динамической типизации нет, типы известны на этапе компиляции.
```c++
void foo(int a, string b) {
    assert(a == 10);
    assert(b == "hello");
}

int main() {
    auto t = std::make_tuple(10, "hello");
    std::apply(foo, t);  // You can call a function
}
```

#### Ссылки внутри `std::tuple`, функция `std::tie`

Внутрь `std::tuple` монжо складывать ссылки, lvalue, rvalue, константые и всяки другие и эти ссылки начинают себя вести "может быть немножко странно".
```c++
std::tuple<int, string> foo() {
    return {30, "baz"};
}

int main() {
    {
        int a = 10; string b = "foo";
        std::tuple<int&, string&&> t(a, std::move(b));  // хранит lvalue на a и rvalue на b
        t = std::make_tuple(20, "bar");  // теперь a = 20, b = "bar"
        assert(a == 20);
        assert(b == "bar");

        std::tie(a, b) = foo();  // creates a temporary tuple<int&, string&>
        assert(a == 30);
        assert(b == "baz");

        auto [c, d] = foo();  // structured binding (C++17), always new variables.
    }
}
```

`std::tie` возвращает tuple из ссылое на свои переменные. Таким образом можно в несколько переменных одновременно что-то записать.
'Structured binding для бедных'. `std::tie` умеет модифицировать старые, а structured binding всегда создает новые.

#### Необходимость правильной расстановки `noexcept` для эффективной работы. Пример: копирование `std::vector<T>` может использовать разные алгоритмы.

#### Условный оператор `noexcept`, конструкция `noexcept(noexcept(expr))`

#### `std::declval`, где можно вызывать, зачем













