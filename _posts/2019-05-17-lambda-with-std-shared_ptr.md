---
published: true
title: 'Используем std::shared_ptr - это же модно!'
---

Недавно на проекте встретили интересный баг связанный с применением умных указателей совместно с лямбдами.

Рассмотрим простой пример, иллюстрирующий проблему:

```cpp
// main.cpp

#include <QCoreApplication>

#include <memory>

#include <QDebug>
#include <QTimer>

int main(int argc, char * argv[]) {
    QCoreApplication a(argc, argv);

    QObject obj;
    {
        auto timer_ptr = std::make_shared<QTimer>();
        obj.connect(timer_ptr.get(), &QTimer::timeout, [&]() {
            qDebug() << "timeout!";
            timer_ptr.reset();
        });
        timer_ptr->start(1000);
    }

    return a.exec();
}

```

Как вы думаете, что делает этот код? Ждет секунду и печатает на экран "timeout!" - это же очевидно.

А вот и нет, на экран не будет напечатано НИЧЕГО. Давайте разберемся почему так происходит и в чем здесь ошибка.

Итак, у нас есть некий `QObject obj`, который "владеет" связью сигнала и лямбды. Эта связь существует до тех пор, пока существует `obj`. Очевидно, что он существует до конца функции `main`.
Далее открывается блок, в котором создается `std::shared_ptr<QTimer>`. Время жизни этого объекта штука сложная и зависит от многих факторов. В данном случае, мы хотим, чтобы объект существовал столь же долго, сколько живет `obj`. Для этого мы выполняем в лямбде захват всех видимых в блоке объектов по ссылке. Это значит, что мы захватили ссылку на локальную переменную `timer_ptr`, которая умрет после выхода из блока. Проверим нашу гипотезу:
```c++
// main.cpp

#include <QCoreApplication>

#include <memory>

#include <QDebug>
#include <QTimer>

int main(int argc, char * argv[]) {
    QCoreApplication a(argc, argv);

    QObject obj;
    {
        auto timer_ptr = std::make_shared<QTimer>();
        obj.connect(timer_ptr.get(), &QTimer::timeout, [&timer_ptr]() {
            qDebug() << "timeout!";
            timer_ptr.reset();
        });
        obj.connect(timer_ptr.get(), &QTimer::destroyed,
                    []() { qDebug() << "destroyed!"; });
        timer_ptr->start(1000);
    }

    return a.exec();
}

```

В консоли видим радостное сообщение: `destroyed!`, а это значит, что при захвате умного указателя по ссылке, в лямбда выражении сохраняется именно ССЫЛКА на этот объект. Поэтому счетчик ссылок `std::shared_ptr` не увеличивается и объект разрушается сразу после выхода из блока.

Как это исправить?
Очевидным решением является смена режима захвата с `&` на `=`. Но более правильно, будет захватить лишь то, что потребуется в дальнейшей работе:
```c++
// main.cpp

#include <QCoreApplication>

#include <memory>

#include <QDebug>
#include <QTimer>

int main(int argc, char * argv[]) {
    QCoreApplication a(argc, argv);

    QObject obj;
    {
        auto timer_ptr = std::make_shared<QTimer>();
        obj.connect(timer_ptr.get(), &QTimer::timeout, [timer_ptr]() mutable {
            qDebug() << "timeout!";
            timer_ptr.reset();
        });
        obj.connect(timer_ptr.get(), &QTimer::destroyed,
                    []() { qDebug() << "destroyed!"; });
        timer_ptr->start(1000);
    }

    return a.exec();
}

```

Еще одним изменением в коде стало добавление ключевого слова `mutable`. Оно нам потребуется для того, чтобы вызывать не константные методы для объектов из списка захвата.

Исходники для своих экспериментов можно получить по ссылке: [github](https://github.com/KirillShirkunov/blog_lamda-with-smart-pointers/)