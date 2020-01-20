---
published: true
title: 'Qt Remote Objects'
---

В составе `Qt` есть прекрасная библиотека [Qt Remote Objects (QtRO)](https://doc.qt.io/qt-5/qtremoteobjects-gettingstarted.html). Если вы уже используете этот фреймворк и вам понадобилась распределенная система или `RPC`, то стоит обратить внимание на нее. Документация по ней на мой взгляд несколько скуднее, чем на остальные части `Qt`, но наличие большого количества примеров позволяет быстро въехать что к чему.

Преимущества:
* кодогенерация - бойлерплейт генерируется за вас, за пару минут можно получить рабочую систему;
* работа с сетью - большой объем работы для подобных приложений обычно занимает обработка ошибок сетевого уровня, с `QtRO` мы об этом вообще не думаем;
* код замаскирован под обычное для `Qt` взаимодействие через сигналы-слоты;
* интеграция с экосистемой `Qt`. Это одновременно и очевидно и поразительно удобно.

В примерах к этой библиотеке есть код для клиента, который отображает древовидную модель (qt/examples/remoteobjects/modelviewclient) и сервера (qt/examples/remoteobjects/modelviewserver).

Код клиента настолько короткий, что вызвал у меня искреннее восхищение и восторг!

```cpp
    QRemoteObjectNode node(QUrl(QStringLiteral("local:registry")));
    node.setHeartbeatInterval(1000);

    QTreeView view;
    view.setWindowTitle(QStringLiteral("RemoteView"));
    view.resize(640,480);

    QScopedPointer<QAbstractItemModelReplica> model(node.acquireModel(QStringLiteral("RemoteModel")));
    view.setModel(model.data());

    view.show();
```

Конечно, за кадром остаются вопросы производительности и гибкости. Но для быстрого старта функционала более чем достаточно.
