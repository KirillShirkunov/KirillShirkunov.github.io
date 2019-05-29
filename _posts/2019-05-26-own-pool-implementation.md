---
published: false
title: 'Пишем свой пул объектов и наступаем на грабли'
---
## Пишем свой пул объектов и наступаем на грабли

Что из себя представляет пул объектов? Обычно это некий список заранее созданных и переиспользуемых объектов. Нужен он для того, чтобы не тратить время на создание этих самых объектов во время выполнения программы. Т.е. схема работы такая: взяли объект из пула, поиспользовали, выкинули обратно в пул ~~тут должна быть картинка~~

### Наивная реализация

Пойдём (хватит ущемлять букву Ё!) простым путём. Возьмем `std::stack` и создадим вокруг него нашу функциональную обертку:

```cpp
// pool.hpp

#include <memory>
#include <stack>

/**
 * @brief The Pool class simple pool of elements
 */
template <typename T>
class Pool {
  public:
    using item_ptr = std::shared_ptr<T>;

  protected:
    std::stack<item_ptr> pool;  // active pool of items

  protected:
    item_ptr create() {
        return std::make_shared<T>();
    }

  public:
    item_ptr pop() {
        if (this->pool.empty()) {
            return this->create();
        } else {
            auto t = this->pool.top();
            this->pool.pop();
            return t;
        }
    }

    void push(item_ptr && item) {
        this->pool.push(item);
    }

    /**
     * @brief reserve create `count` elements into pool
     */
    void reserve(size_t count) {
        size_t existing = this->pool.size();

        Q_ASSERT_X(count > 0, "Pool::reserve",
                   "argument `count` should be > 0");

        while (existing <= count) {
            this->pool.push(this->create());
            ++existing;
        }
    }
};

```

Отлично, ~~стильно-модно-молодежно~~ теперь у нас есть шаблонный пул для объектов типа `T`. Заметьте, что в этой реализации пул не владеет объектами, которые он создает. Возможно это не совсем правильно, но этот вариант работы очень удобен, главное не забывать возвращать объекты в пул, иначе от него нет никакого прока.

Начинаем использовать `Pool` в проекте и замечаем, что иногда в релизной сборке мы крашимся. Благо у нас на проекте собираются дампы и можно понять, что там произошло.

Перейдем ближе к делу и рассмотрим пример использования, который приводит к проблеме:

```cpp
class Widget: public QDialog {
 ...
};

Pool<Widget> pool;

void foo(int count) {
    for (int i = 0; i < count; ++i) {
        auto w = pool.pop();

        w->setParent(qApp);
        w->exec();

        w->push(w);
    }
}

```

У нас имеется глобальный доступ к пулу (в проекте это был синглтон, но для примера это не важно).
Когда мы закрываем приложение - получаем двойное освобождение памяти. Тот, кто имеет опыт с `Qt` сразу заметит очень важную строку `w->setParent(qApp);`. Причина, естественно, именно в ней.

Недолго думая добавляем специализацию пула для наследников `QObject`:

```cpp
/**
 * @brief The QPool class specialized pool for QObjects
 */
template <typename T>
class QPool : public Pool<T> {
  protected:
    // enabled only for QObject classes
    template <typename = typename std::enable_if<
                      std::is_base_of<QObject, T>::value>::type>
    typename Pool<T>::item_ptr create() {
        return std::shared_ptr<T>(new T(), [](T * t) {  // set custom deleter
            auto tmp = qobject_cast<QObject *>(t);

            Q_ASSERT_X(tmp, "QPool",
                       "QObject should be inherited from QObject!");

            if (tmp) {
                tmp->setParent(nullptr);  // disable Qt parent system
                tmp->deleteLater();
            } else {
                delete tmp;
            }
        });
    }

  public:
    // enabled only for QObject classes
    template <typename = typename std::enable_if<
                      std::is_base_of<QObject, T>::value>::type>
    typename Pool<T>::item_ptr pop() {
        if (this->pool.empty()) {
            return this->create();
        }

        return Pool<T>::pop();
    }

    // enabled only for QObject classes
    template <typename = typename std::enable_if<
                      std::is_base_of<QObject, T>::value>::type>
    void push(typename Pool<T>::item_ptr && ptr) {
        ptr->setParent(nullptr);  // disable Qt parent system

        Pool<T>::push(std::move(ptr));
    }

    /**
     * @brief reserve create `count` elements into pool
     * @note reimplemented, because it is non virtual
     */
    void reserve(size_t count) {
        Q_ASSERT_X(count > 0, "QPool::reserve",
                   "argument `count` should be > 0");

        size_t existing = this->pool.size();

        while (existing <= count) {
            this->pool.push(this->create());
            ++existing;
        }
    }
};
```

Видно два основных дополнения:

* свой кхм...удалятор (как это будет по-русски?);
* сброс родителя при возвращении объекта в пул (если не вернули - разбирайтесь сами).
* стало еще молодежней, даешь `SFINAE` в массы!

Что же тут вообще происходит? Методы `QPool::pop`, `QPool::push`, `QPool::create` затеняют методы из `Pool`, причем произойдет это только, если `T` наследник `QObject` (да здравствует `SFINAE` ~~эх, сюда бы концепты!~~). Если параметр шаблона не удовлетворяет этим условиям - получаем обычный `Pool` ~~а зачем? уж лучше ошибку компиляции~~

Выпускаем новую версию пула - не падает. Радуемся и тихо пилим проект дальше.

## Грабли

В самый разгар релиза прилетает дамп. После некоторой медитации над ним, стало понятно, что с пулом что-то не так. Из сотен запусков только один привел к проблеме, а это уже серьезный повод задуматься.

Между прочим, эта реализация без проблем отработала почти год - аргумент, как-никак. Но исправлять все равно надо. Еще пара часов медитаций и гуглений и вот он ответ: `QWidget` значительно переопределяет метод `QObject::setParent`, а метод-то этот не виртуальный! Выходит что пул вызывает совсем не то, что нужно для `QWidget`, а это уже похоже на UB...
  
## А как же многопоточность

Пока никак, падаем.