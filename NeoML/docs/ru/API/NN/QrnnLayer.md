# Класс CQrnnLayer

<!-- TOC -->

- [Класс CQrnnLayer](#класс-cqrnnlayer)
    - [Настройки](#настройки)
        - [Размер скрытого слоя](#размер-скрытого-слоя)
        - [Размер окна](#размер-окна)
        - [Шаг окна](#шаг-окна)
        - [Дополнительные элементы последовательности (padding)](#дополнительные-элементы-последовательности-(padding))
        - [Функция активации](#функция-активации)
        - [Dropout](#dropout)
        - [Разворот последовательностей](#разворот-последовательностей)
    - [Обучаемые параметры](#обучаемые-параметры)
        - [Фильтры свертки](#фильтры-свертки)
        - [Свободные члены](#свободные-члены)
    - [Входы](#входы)
        - [Размер первого входа](#размер-первого-входа)
        - [Размеры второго входов](#размеры-второго-входов)
    - [Выходы](#выходы)

<!-- /TOC -->

Класс реализует квази-рекуррентный слой, применяющийся к набору последовательностей векторов.

Результатом операции является последовательность векторов, каждый вектор в которой имеет размер `GetHiddenSize()`.

В отличие от LSTM или GRU, в этом слое большая часть вычислений производится до рекуррентной части, что значительно повышает производительность на GPU.
Это достигается за счет использования одномерной свертки "по времени" ([CTimeConvLayer](ConvolutionLayers/TimeConvLayer.md)).

Реализация основана на [следующей статье](https://arxiv.org/abs/1611.01576).

## Настройки

### Размер скрытого слоя

```c++
void SetHiddenSize(int hiddenSize);
```

Установить размер скрытого слоя. Это значение влияет на размер выхода.

### Размер окна

```c++
void SetWindowSize(int windowSize);
```

Устанавливает размер окна, которым свертка "по времени" итерируется по входным последовательностям.

### Шаг окна

```c++
void SetStride(int stride);
```

Устанавливает шаг, с которым свертка "по времени" итерируется по входным последовательностям.

### Дополнительные элементы последовательности (padding)

```c++
void SetPadding(int padding);
```

Устанавливает количество дополнительных нулевых элементов к последовательности. Если `IsReverseSequence()` равен `false`, то нулевые элементы будут добавлены к началу последовательности. Иначе, нулевые элементы будут добавлены к концу последовательности. По умолчанию равен `0`.

### Функция активации

```c++
void SetActivation( TActivationFunction newActivation );
```

Установиливает функцию активации, используемую в `update` гейте. По умолчанию используется `AF_Tanh`.

### Dropout

```c++
void SetDropout(float rate);
```

Установиливает вероятность зануления `forget` гейта.

### Разворот последовательностей

```c++
void SetReverseSequence( bool isReverseSequense )
```

Включает обработку последовательностей в обратном порядке.

## Обучаемые параметры

### Фильтры свертки

```c++
CPtr<CDnnBlob> GetFilterData() cons;
```

Фильтры, содержащие веса сразу для всех гейтов, представляют собой [блоб](DnnBlob.md) размеров:

- `BatchLength` равен `1`;
- `BatchWidth` равен `3 * GetHiddenSize()`;
- `Height` равен `GetWindowSize()`;
- `Width` равен `1`;
- `Depth` равен `1`;
- `Channels` равен `Height * Width * Depth * Channels` у входов.

Вдоль оси `BatchWidth` матрица содержит веса гейтов в следующем порядке:

```c++
G_Update, // update gate (Z из статьи)
G_Forget, // forget gate (F из статьи)
G_Output, // output gate (O из статьи)
```

### Свободные члены

```c++
CPtr<CDnnBlob> GetFreeTermData() const
```

Свободные члены представляют собой блоб, имеющий суммарный размер, равный `3 * GetHiddenSize()`. Порядок относительно гейтов см. [выше](#фильтры-свертки).

## Входы

Слой имеет от 1 до 2 входов:

1. Набор входных последовательностей векторов.
2. *[Опционально]* Начальное состояние рекуррентного слоя. Будет использовано в качестве состояния слоя перед первым шагом. Если вход не задан, то будет заполнено нулями.

### Размеры первого входа

- `BatchLength` - длина последовательности;
- `BatchWidth` - количество последовательностей в наборе;
- `Height * Width * Depth * Channels` - размер векторов в последовательностях.

### Размеры второго входа

- `BatchLength`, `Height`, `Width` и `Depth` должны быть равны `1`;
- `BatchWidth` должен быть равен `BatchWidth` у первого входа;
- `Channels` должен быть равен `GetHiddenSize()`.

## Выходы

Единственный выход содержит блоб с результатами следующиего размера:

- `BatchLength` вычисляемый относительно размеров входа по следующей формуле `(BatchLength + GetPaddingFront() - (GetWindowSize() - 1)) / GetStride() + 1)`;
- `BatchWidth` равный `BatchWidth` у первого входа;
- `ListSize`, `Height`, `Width` и `Depth` равные `1`;
- `Channels` равный `GetHiddenSize()`.
