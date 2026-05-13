# Поток (англ. flux)

20й стандарт порадовал нас ranges. 
Наконец множество задач решаются приятнее, чуть более по-людски. 

Когда только начинаешь вникать в возможности, сразу как-то силу в руках чувствуешь. 
Какие-то новые способности в себе открываешь. 
И там ещё как-то всё оптимально, лениво с этими view (когда безопасно, конечно же, да `std::ranges::views::filter`?).

Но идеален ли код с ренджами? 
Чисто визуально. 

Я как настоящий эстет не всегда могу с этим согласиться. 

Мне ещё во времена универа понравились стримы в Java. 
Вот например рандомный кусок из интернета:
```java
Map<String, Double> revenueByCategory =
            orders.stream()
                .filter(Order::isPaid)
                .filter(o -> o.getDate().isAfter(cutoff))
                .flatMap(o -> o.getItems().stream())
                .collect(groupingBy(
                    Item::getCategory,
                    summingDouble(Item::getRevenue)
                ));
```
Многословно, но это ж Java. 
Там вообще `Borshch borshch = new Borshch();`.

Чисто для души хочется чего-то похожего на плюсах. 
Чтобы читалось красиво. 

Есть решение! 
[flux](https://github.com/tcbrindle/flux) -- библиотека для решения тех же задач, что и стандартные ranges, только в стиле Java streams. 
Может даже ближе по духу к Python itertools или, прости господи, Rust iterators.

Давайте попробуем в следующем ключе: описываем задачу, пытаемся решить её с помощью ranges, пытаемся решить с помощью flux. 
Возможно где-то не получится. 
Возможно где-то не получится _у меня_, а у Вас может получится. Напишите в комментарии. 

## Example 1

Для начала давайте возьмём почти пример из README flux: в `vector` с числами отфильтруем чётные, удвоим остальные, и получим их сумму:
```cpp
// ranges
auto ranges_res = v 
        | views::filter([](const int x) { return x % 2 == 0; })
        | views::transform([](const int x) { return x * 2; });
int x = std::ranges::fold_left(ranges_res, 0, std::plus<>{});
```
```cpp
// flux
int y = flux::ref(v)
            .filter(flux::pred::even)
            .map([](const int x) { return x * 2; })
            .sum();
```
Ну как-то второе поприятнее! 
Да, вы можете сказать, что дело как минимум наличии именной лямбды для `even`. 
Типа читерство. 
Но тут либа даёт готовые инструмент. Я им пользуюсь, и мне радостно.
Менее многослово в целом. 
А ещё `sum()` есть, а не целая отдельная строчка для подсчёта суммы с явным указанием операции...

Link: [flux.godbolt.org/z/sPvKT3szT](https://flux.godbolt.org/z/sPvKT3szT).

Что важно, для тестов пришлось сообразить какой-то вариант `toVector`, 
потому что наличием терминальных операцией стандартные ренджи не блещут, а хочеца. 
Останавливаться на нём не будем. Думаю, вы разберётесь прекрасно сами. 
Конечно, рано или поздно `std::ranges::to` доедет везде, но пока его нет, будем страдать. 
Вот вам моментик сразу отметить. 

Хотя конечно `flux` такое имеет...
```cpp
auto y = flux::ref(v)
            .filter(flux::pred::even)
            .map([](const int x) { return x * 2; })
            .to<std::vector>();
```
Link: [flux.godbolt.org/z/zEnEEdvr8](https://flux.godbolt.org/z/zEnEEdvr8).

## Example 2

Давайте такую:
- имеем `std::vector<int>`
- хотим посчитать префиксную сумму
- оставить только элементы >10 (то есть отбросить начало вектора)
- считаем суммы со sliding window размера 3

Стандартная библиотека нам тут сильно не поможет:
```cpp
int acc = 0;
for (int i = 0; i < input.size(); ++i) {
    acc += input[i];
    input[i] = acc;
}

auto filtered = input
    | std::views::filter([](int x) { return x > 10; })
    | toVector;

std::vector<int> result;
for (size_t i = 0; i + 2 < filtered.size(); ++i) {
    int s = filtered[i] + filtered[i + 1] + filtered[i + 2];
    result.push_back(s);
}
```
Проблем несколько:
- у нас нет ничего для sliding window
- состояние какое-то промежуточное держать с ренджами тяжковато (не говорю невозможно, это же всё-таки плюсы)

`flux` чуть более богат на реализованные функции:
```cpp
auto tmp = flux::ref(input)
    .scan([](int acc, int x) { return acc + x; }, 0)
    .filter([](int x) { return x > 10; })
    .to<std::vector>();

auto result = flux::from(std::move(tmp))
    .slide(3)
    .map([](auto win) {
        return flux::from(win).sum();
    })
    .to<std::vector>();
```
Link: [flux.godbolt.org/z/rT5sEcxa4](https://flux.godbolt.org/z/rT5sEcxa4).

Читается ли этот код проще -- вопрос субъективный. 
Я вам просто показываю. 
Идём дальше. 

## Example 3

Задача:
- берём `std::vector<std::vector<std::string>>`
- уплощаем (или как сказать "делаем плоским"? Наверное, "делаем плоским" и сказать. Flatten'ируем)
- приводим к upper case
- join'им через запятую

Стандартная библиотека:
```cpp
auto view = input
    | std::views::join
    | std::views::filter([](auto const& s) { return !s.empty(); })
    | std::views::transform([](std::string s) {
        std::ranges::transform(s, s.begin(), ::toupper);
        return s;
    });

std::string result;
bool first = true;
for (auto const& s : view) {
    if (!first) result += ", ";
    result += s;
    first = false;
}
```
`flux`:
```cpp
return flux::ref(input)
    .flatten()
    .filter([](auto const& s) { return !s.empty(); })
    .map([](std::string s) {
        for (auto& c : s) c = std::toupper(static_cast<unsigned char>(c));
        return s;
    })
    .fold([](std::string acc, std::string const& s) {
            if (!acc.empty()) acc += ", ";
            acc += s;
            return acc;
        },
        std::string{}
    );
```
Link: [flux.godbolt.org/z/sjGvT61ds](https://flux.godbolt.org/z/sjGvT61ds).

И там, и сям нет терминирующего `join`, но в `flux` хотя бы цепочку рвать для этого не нужно. 

## Example 4

Вводные:
- имеем 2 `vector`
- делаем zip
- оставляем элементы на чётных позициях
- перемножаем

STL:
```cpp
return std::views::zip(std::views::iota(size_t{0}), a, b)
    | std::views::filter([](auto t) { return std::get<0>(t) % 2 == 0; })
    | std::views::transform([](auto t) { return std::get<1>(t) * std::get<2>(t); })
    | toVector;
```

`flux`:
```cpp
return flux::zip(flux::ints(0), std::move(a), std::move(b))
    .filter([](auto t) { return std::get<0>(t) % 2 == 0; })
    .map([](auto t) { return std::get<1>(t) * std::get<2>(t); })
    .to<std::vector>();
```
Link: [flux.godbolt.org/z/9jq83a3b6](https://flux.godbolt.org/z/9jq83a3b6).

Строчка в строчку. 

## Example 5

Чтобы вы не думали, что я тут ranges закидываю известными отходами, вот вам немного фактов. 

ranges такие сложные, потому что пытаются дать вам как можно гарантий. 
Если у вас что-то не так, вы ломаетесь на компиляции. 
И это вообще-то прекрасно! 

`flux` такой чувачок с района. 
Он слово по-пацански держит, но особо не разбрасывается. 
Требования чуть попроще, чаще молчит при проблемах. 

Одна из фичей, которые on the fence, это возможность несколько раз получать результаты вычислений:
```cpp
auto seq = flux::from(input)
    .filter(pred)
    .take(10);
auto vec = seq.to<std::vector>();
auto sum = seq.sum();
```
В среднем у вас всё будет оки-доки, но если `input` -- input sequence (поток, генератор), на втором вызове вы можете тихонько получить маслину (в райнтаме, конечно же). 

Так что следить приходится чуть аккуратнее. 

Если ranges -- бюрократ, который первенца готов отдать за строгое исполнение контрактов и гарантий, `flux` просто пытается быть удобнее. 

`flux` ещё кстати не на итераторах живёт (ranges обязаны были быть построенными вокруг этой модели, так как нужна совместимость в рамках уже существующих в STL решений), 
а на своей модели последовательностей и курсоров.
Тут есть некоторый кек, что `flux` иногда может сделать чуть больше, чем вы просите. 
Прочитать чуть больше данных, забуферизировать что-нибудь. 
Это может быть вредно для перфа (делать бенчмарки себе целью я не ставил, тут вы разберётесь, думаю).

Я не то чтобы хотел продать вам эту либу. 
Скорее показать чуть другой вариант развития событий. 
У меня нет никакой глубокой экспертизы. 
Я буквально только на godbolt с ней потыкался да попробовал посравнивать с ranges. 
Для души покайфовал. 

Может вам зашло. 

P.S. niki4smirn принёс первый пример на Rust:
```rust
let x: = v
    .iter()
    .copied()
    .filter(|x| x % 2 == 0)
    .map(|x| x * 2)
    .sum();
```
И что есть «[flux](https://github.com/gattaca-com/flux) дома, но он про другое».
