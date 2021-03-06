# Эллипсис и функции с произвольным числом аргументов

Наверняка все С++ (а уж просто C тем более) программисты знакомы с семейством функций
`printf`. Одной из удивительных особенностей этих функций является возможность принимать произвольное число аргументов. А также на `printf` можно писать [полноценные программы!](https://github.com/carlini/printf-tac-toe). Исследованию и описанию этого безумия даже посвящены отдельные [статьи](https://www.usenix.org/system/files/conference/usenixsecurity15/sec15-paper-carlini.pdf).

Мы же остановимся только на произвольном числе аргументов. Но для начала я расскажу одну занимательную историю.

Какая-то замечательная библиотека предоставляла красивую функцию
```C++
template <class HandlerFunc>
void ProcessBy(HandlerFunc&& fun) 
requires std::is_invocable_v<HandlerFunc, T1, T2, T3, T4, T5>;
```

И программист думал вызвать эту восхитительную функцию. В качестве `HandlerFunc` подсунуть лямбду, в которой ему было совершенно наплевать на передаваемые аргументы `T1, T2, T3, T4, T5`.
Что же он мог сделать?

Вариант первый: честно перечислить пять аргументов с их типами. Как деды делали.

```C++
ProcessBy([](T1, T2, T3, T4, T5) { do_something(); });
```
Если имена типов короткие, почему бы и нет. Но все равно как-то слишком подробно. Не удобно. Да и добавится новый аргумент — придется и тут править. Не очень современный C++-подход.

Вариант второй: воспользоваться функциями с произвольным числом аргументов.

```C++
ProcessBy([](...){ do_something(); });
```

Вау, красота! Компактно и здорово. До чего прогресс дошел!
И оно скомпилировалось. И даже работало. И так программист и оставил.

Но однажды замечательная библиотека обновилась, стала лучше и безопаснее. И начались странные, необъяснимые падения. SIGILL, SIGABRT, SIGSEGV. Все наши любимые друзья хлынули в проект.

Что произошло? Кто виноват? Что делать? Без опытного сыщика тут не обойтись...

------
Дайте разбираться.

В C можно определять собственные функции, принимающие сколь угодно много аргументов.
И сделать это можно двумя способами: 

1. Пустой список аргументов.
```C
void foo() {
    printf("foo");
}

foo(1,2,4,5,6);
```
Казалось бы, функция `foo` не должна в принципе принимать аргументы. [Но нет](https://godbolt.org/z/MPv6E4). В C функции, объявленные с пустым списком аргументов, на самом деле являются функциями с произвольным числом аргументов. Действительно ничего не принимающая функция объявляется так
```C
void foo(void);
```
В C++ это безобразие исправили.

2. Эллипсис и `va_list`

```C
#include <stdarg.h>

void sum(int count, /* чтобы получить доступ к списку аргументов, 
                     нужен хотя бы один явный  */
         ...) {
    int result = 0;
    va_list args;
    va_start(args, count);
    for (int i = 0; i < count; ++i) // причем функция не знает,
                                    // сколько аргументов передали 
    {
        result += va_arg(args, int); // запрашиваем очередной аргумент
                                     // функция не знает какой у него тип
                                     // указываем самостоятельно — int
    }
    va_end(args);
    return result;
}
```
Если явного аргумента не будет, то получить доступ к списку остальных нельзя.
Более того, мы уйдем в область implementation defined поведения.

Также на этот явный аргумент, предшествующий вариативной части, налагаются ограничения:
- Он не может быть помечен спецификатором `register`. Но это мало кому надо
- Он не может иметь «повышаемый» тип. Привет нашим любимым integer/float promotion. `float`, `short`, `char` нельзя.

Нарушаем ограничения явного аргумента — получаем неопределенное поведение.
Запрашиваем у `va_arg` повышаемый тип — снова неопределенное поведение.
Передаем не тот тип, что запрашиваем... Правильно, неопределенное поведение.

Невероятные возможности по отстрелу рук и ног себе и пользователям кода! Собственно, на этих
возможностях и идет игра, при атаках на `printf`.

И в C++, конечно же, эта прелесть осталась. И не просто осталась, но и значительно усилилась!

-------

C простой, маленький язык. В нем не так много типов. Примитивы, указатели, да пользовательские структуры.

В C++ есть ссылки. Есть объекты с интересными конструкторами и деструкторами. И вы уже наверняка догадались о том, что будет неопеределенное поведение, если засунуть ссылку или такой объект в качестве аргумента вариативной функции. Еще больше возможностей для веселой отладки!

Но C++ не был бы самим собой, если бы в нем эту проблему не «решили». И так у нас есть C++-style вариадики:

```C++
template <class... ArgT>
int avg(ArgT... arg) {
    // доступно число аргументов
    const size_t args_cnt = sizeof...(ArgT);
    // доступны их типы

    // итерироваться по аргументам нельзя
    // нужно писать рекурсивные вызовы для обработки,
    // либо использовать fold expressions
    return (arg + ... + 0) / ((args_cnt == 0) ? 1 : args_cnt);
}
```
Не очень удобно, но намного лучше и безопаснее. 

------

Ну что ж, теперь когда все карты вскрыты, вернемся к нашему детективу.

Убийца — C-вариадик!
```C++
ProcessBy([](...){ do_something(); });
```

Когда библиотека обновилась. В ней, незначительно на первый взгляд, поменялся один из типов `T`, которые передавались функцией `ProcessBy` в `HandlerFunc`. Но это изменение привело к неопределенному поведению.

А программисту же нужно было использовать C++-вариадик.

```C++
ProcessBy([](auto...){ do_something(); });
```

Все. Всего одно слово `auto` и никто бы не погиб. Удобно.

И конечно, чтобы не было лишних копирований, надо дописать два амперсанда:

```C++
ProcessBy([](auto&&...){ do_something(); });
```
Вот теперь все. Прекрасный способ принять и проигнорировать сколь угодно много аргументов.
Ну, а тем программистом был когда-то я сам.

# Полезные ссылки
1. https://habr.com/ru/post/430064/
2. https://en.cppreference.com/w/c/variadic/va_list