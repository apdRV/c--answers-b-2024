## Билет 81. Parameter pack (детали)

### Работа с parameter pack классах и функциях: рекурсия

Нельзя объявить пак из переменных или полей, если это не аргументы функции. 
В то же время при написании обобщенного кода хранить в себе значение каждого типа из какого-то
пака часто необходимо. В таком случае можно просто хранить одно поле типа `std::tuple` 
и обращаться к значениям в нём с помощью
`std::get`. Но иногда `tuple` использовать не хочется или не получается.
В этом случае используется паттерн с рекурсивными вложенными структурами и паттерн-матчингом. 

Разберем этот паттерн на примере написания собственного наброска для `std::tuple` и связанных
с ним структур и функций.

Суть приёма в том, что мы храним структуру, в которой лежит переменная первого типа из пака и такая же структура, только для пака без первого элемента. При этом нужно специализировать структуру от пустого пака, т.к. её невозможно
определить описанным методом (у пустого пака нет первого элемента).

На практике вместо храненеия рекурсивной структуры используют наследование от неё,
т.к. в C++ нет ZST, а хранить в себе лишний байт на пустую структуру не хочется.

```c++
template <typename... T>
struct tuple {};  // Используется только для пустого пака, для остальных
                  // следующая перегрузка более подходящая

template <typename Head, typename... Tail>
struct tuple<Head, Tail...> {
    Head head{};
    tuple<Tail...> tail;
};
```

Попробуем получить доступ к полям нашего `tuple`. Для начала научимся извлекать их тип.
Для этого также применим рекурсивную процедуру - для получения нулевого элемента нужно вернуть
тот, что хранится в текущей структуре, а для получения остальных - запуститься от "хвоста", 
при этом индекс элемента в "хвосте" ровно на 1 меньше, чем его индекс в исходном массиве.

```c++
template <std::size_t I, typename T>
struct tuple_element {
};  // Вызовет ошибку, если мы считываем type, а T - не tuple

template <typename Head, typename... Tail>
struct tuple_element<0, tuple<Head, Tail...>> {
    using type = Head;
};

template <std::size_t I, typename Head, typename... Tail>
struct tuple_element<I, tuple<Head, Tail...>> {
    using type = typename tuple_element<I - 1, tuple<Tail...>>::type;
};

```

Последнюю специализацию можно записать короче через наследлование:

```c++
template <std::size_t I, typename Head, typename... Tail>
struct tuple_element<I, tuple<Head, Tail...>>
    : tuple_element<I - 1, tuple<Tail...>> {};
```

При повторении того же приёма с функциями врзникает тонкий момент -
частичных специализаций у функций не бывает. Из-за этого до C++17 приходилось оборачивать функцию 
в структуру, превращая в статический метод.

```c++
template <std::size_t I>
struct getter {
    template <typename... T>
    static auto &get(tuple<T...> &t) {
        return getter<I - 1>::get(t.tail);
    }
};

template <>
struct getter<0> {
    template <typename... T>
    static auto &get(tuple<T...> &t) {
        return t.head;
    }
};

template <std::size_t I, typename... T>
auto &get(tuple<T...> &t) {
    return getter<I>::get(t);
}
```

В C++17 появилась конструкция `if constexpr`, которая позволяет сделать разбор случаев прямо в теле `get`


```c++
template <std::size_t I, typename... T>
auto &get(tuple<T...> &t) {
    if constexpr (I == 0) {
        return t.head;
    } else {
        return get<I - 1>(t.tail);
    }
}
```

Для полноты напишем также `tuple_size`

```c++
template <typename T>
struct tuple_size {
};  // Вызовет ошибку, если мы считываем размер, а T - не tuple

template <typename... T>
struct tuple_size<tuple<T...>>
    : std::integral_constant<std::size_t, sizeof...(T)> {};
```

Здесь `std::integral_constant` - тип из стандартной библиотеки, обеспечивающий удобное хранение констант
на этапе компиляции.
Для того, чтобы начать хранить в себе константу достаточно отнаследоваться от него с подходящим шаблонным параметром. 
Подробнее о его устройстве - в билете о метапрограммировании.

### Циклы по элементам пака аргументов

В C++17 появились fold expression с запятой и стало возсожно применить какую-либо опреацию к каждому опреанду:

```c++
template <typename... T>
void print(int n, const T &...vals) {
    auto f = [&](const auto &val) -> void {
        if (n > 0) {
            std::cout << typeid(val).name() << " : " << val << std::endl;
            n--;
        }
    };

    (f(vals), ...); // (1)
}
```

(1) раскроется в вызов `f` на элементах пака через оператор запятая, гарантирующий порядок выполнения.


До C++17 подобных fold expression не было и их приходилось имитировать, пользуясь тем, что можно было распаковать 
пак через запятую в инициализации массивов. При этом было важно, чтобы функтор, который вызывался в цикле возвращал 
значение такого же типа, как и у элементов массива.

```c++
template <typename... T>
void print(int n, const T &...vals) {
    auto f = [&](const auto &val) -> int {
        if (n > 0) {
            std::cout << typeid(val).name() << " : " << val << std::endl;
            n--;
        }
        return 0;
    };

    [[maybe_unused]] int dummy[] = {f(vals)...};
}
```
Здесь импользуется pack expansion - шаблон повторяется через запятую для всех элементов пака (Точно такой же используется
при передаче всего пака в качестве агрументов к другой функции)

### Цикл по элементам tuple

Есть идея использовать для цикла по элементам tuple не рекурсию, а что-то похожее на выражения из предыдущего пункта.
Выражение, которое хочется превратить в шаблон - `get<I>(t)`, где T - наш `tuple`. 

Очевидно, что итерироваться надо по `I`, а значит, нам нужен value pack из `std::size_t` от 0 до размера массива.
Его можно получить с помощью стандартной библиотеки.


```c++
template <typename... T, std::size_t... I>
void print_impl(const tuple<T...> &t, std::index_sequence<I...>) {
    auto f = [&](const auto &val) -> void {
        std::cout << typeid(val).name() << " : " << val << std::endl;
    };

    (f(get<I>(t)), ...);  // (1)
}

template <typename... T>
void print(const tuple<T...> &t) {
    print_impl(t, std::make_index_sequence<sizeof...(T)>{});
}

```

`std::index_sequence` - фиктивная структура, необходимая только для того, чтобы компилятор мог вывести I.
(Без такого автовывода нет никакой возможности передать пак из констант в функцию. Именно из-за
этого и требуется дополнителная функция `print_impl`) 
`std::make_index_sequence<N>` - псевдоним для структуры вида `std::index_sequence`, заполненной std::size_t от 0 до N.
Экземпляр подоной структуры мы создаем при вызове функции.






