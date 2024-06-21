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

Внутрь `std::tuple` можно складывать ссылки, lvalue, rvalue, константые и всякие другие, и эти ссылки начинают себя вести "может быть немножко странно".
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
```c++
template<typename T>
struct vector {
    T *data = nullptr;
    std::size_t len = 0;

    vector() = default;
    vector(const vector &) = default;
    vector(vector &&) = default;

    vector &operator=(vector &&) = default;
    vector &operator=(const vector &other) {
        if (this == &other) {
            return *this;
        }
        // NOTE: two separate algorithms for providing a strong exception safety
        if constexpr (std::is_nothrow_copy_assignable_v<T>) {
            // Never throws (like in lab10)
            std::cout << "naive copy assignment\n";
            for (std::size_t i = 0; i < len && i < other.len; i++) {
                data[i] = other.data[i];
            }
            // ...
        } else {
            std::cout << "creating a new buffer\n";
            // May actually throw, cannot override inidividual elements, should allocate a new buffer.
            *this = vector(other);
        }
        return *this;
    }
};
```

Благодаря проверке, выкидывает ли тип `T` исключения при копировании смогли добиться строгой гарантии исключений в операторе копирования вектора, реализовав в зависимости от случая разные алгоритмы. Тем не менее всё ещё есть проблема с `std::optional`. Иногда оператор копирования должен быть помечен `noexcept`, а иногда нет. Решается так:
```c++
template<typename T>
struct vector {
    T *data;
    std::size_t len = 0;

    vector() = default;
    vector(const vector &) = default;
    vector(vector &&) = default;

    vector &operator=(vector &&) = default;
    vector &operator=(const vector &other) {
        if (this == &other) {
            return *this;
        }
        // NOTE: two separate algorithms for providing a strong exception safety
        if constexpr (std::is_nothrow_copy_assignable_v<T>) {
            // Never throws (like in lab10)
            std::cout << "naive copy assignment\n";
            for (std::size_t i = 0; i < len && i < other.len; i++) {
                data[i] = other.data[i];
            }
            // ...
        } else {
            std::cout << "creating a new buffer\n";
            // May actually throw, cannot override inidividual elements, should allocate a new buffer.
            *this = vector(other);
        }
        return *this;
    }
};

template<typename T>
struct optional {
    alignas(T) char bytes[sizeof(T)];
    bool exists = false;

    optional() = default;
    ~optional() { reset(); }

    T &data() { return reinterpret_cast<T&>(bytes); }
    const T &data() const { return reinterpret_cast<const T&>(bytes); }

    void reset() {
        if (exists) {
            data().~T();
            exists = false;
        }
    }
    optional &operator=(const optional &other)
        // default is bad, optional<int> is not nothrow-copyable
        // noexcept  // optional<std::string> is not nothrow-copyable
        noexcept(std::is_nothrow_copy_constructible_v<T>)  // ok, conditional noexcept-qualifier
    {
        if (this == &other) {
            return *this;
        }
        reset();
        if (other.exists) {
            new (bytes) T(other.data());
            exists = true;
        }
        return *this;
    }
};
```
Обратим внимание на копирующий опреатор присваивания `optional<T>`, там есть то, что нам надо. Метод помечается `noexcept` в зависимости от значения выражения в скобках.

#### Условный оператор `noexcept`, конструкция `noexcept(noexcept(expr))`
```c++
int foo() noexcept { return 1; }
int bar()          { return 2; }
std::vector<int> get_vec() noexcept { return {}; }

int main() {
    int a = 10;
    std::vector<int> b;
    // Simple cases
    static_assert(noexcept(a == 10));
    static_assert(!noexcept(new int{}));   // no leak: not computed
    static_assert(noexcept(a == foo()));
    static_assert(!noexcept(b == b));      // vector::operator== is not noexcept for some reason, but I don't know how it can fail
    bool x = noexcept(a == 20);
    assert(x);

    // Complex expressions
    static_assert(!noexcept(a == bar()));  // bar() is not noexcept
    static_assert(noexcept(get_vec()));  // noexcept even though copying vector may throw: return value creation is considered "inside" function
    static_assert(noexcept(b = get_vec()));  // operator=(vector&&) does not throw
    static_assert(!noexcept(b = b));  // operator=(const vector&) may throw
}
```
Оператор `noexcept` позволяет проверять, верно ли, что выражение никогда не кидает исключений (помечено `noexcept`). Он не вычисляет выражение, поэтому никаких утечек быть не может. Все проверки происходят на этапе компиляции.

#### `std::declval`, где можно вызывать, зачем













