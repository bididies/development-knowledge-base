## Паттерны обработки данных
> Ниже описаны шаблоны обработки сырых данных, преобразуемых в вид, удобный для их дальнейшего анализа (аналитиками или машинами).

### Alpha
> - это архитектурный шаблон обработки большого кол-ва данных.
> 
> При котором входные (сырые) данные обрабатываются одновременно по двум каналам `Speed layer` и `Batch layer`. 
> Далее обработанные данные попадают в слой `Serving layer`. Тут хранятся данные в оптимизированном виде для запросов и анализа (например для обучения каких либо ИИ моделей).

![[Pasted image 20240805011400.png]]

#### Speed layer (Stream)
> Потоковая обработка данных позволяет анализировать данные в реальном времени по мере их поступления, мгновенно реагировать на события и принимать решения.

Данный канал выдает **быстрый**, но **менее точный результат**.

#### Batch layer
> Пакетная обработка данных - это процесс обработки данных в больших объемах, сгруппированных в небольшие партии (батчи). Включая исторические данные.

Данный канал обрабатывает пачку данных и выдает **медленный**, но **более точный результат**, т.к. основан на анализе большого кол-ва данных.

#### Слой сервисных данных (Serving Layer)
Слой сервисных данных - это место, где хранятся пакетные представления данных, доступные для запросов и анализа. Этот слой обеспечивает низкую задержку при доступе к данным.

_Описание:_ В слое сервисных данных пакетные представления данных хранятся в оптимизированной форме, что обеспечивает быстрый доступ к ним. Это позволяет строить интерактивные приложения для анализа данных.

### Kappa
> шаблон потоковой обработки данных.

Т.е. только `Speed layer` из Alpha-архитектуры, но достаточно быстрый, чтобы отдавать точный результат почти мгновенно.


### Alpha vs. Kappa
Для прогнозирования будущих событий на основе исторических данных, следует выбрать Лямбда-подход.
С другой стороны, если необходимо недорого развернуть Big Data систему для эффективной обработки уникальных событий в реальном времени без исторического анализа, Каппа-архитектура отлично справится с этой задачей. Каппа подходит для тех алгоритмов Machine Learning, которые обучаются в режиме онлайн и не нуждаются в пакетном уровне.