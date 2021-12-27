# Аспектно-ориентированная оценка тональности отзывов на рестораны

## Навигация по коду 

* [`Category_prediction`](/category/Category_prediction.ipynb) и [`Category_evaluation`](/category/Category_evaluation.ipynb) –– код для извлечение эксплицитных упоминаний аспектов.
* [`BERT_ABSA_1.ipynb`](/category_tone/BERT_ABSA_1.ipynb) и [`BERT_ABSA_2.ipynb`](/category_tone/BERT_ABSA_2.ipynb) –– код для оценки тональности упоминания аспекта.
* [`Sentiment_evaluation.ipynb`](/category_tone/Sentiment_evaluation.ipynb) –– код для оценки тональности всего отзыва по категориям.

## Baseline 1: категория упоминаний

Основной способ, которым мы пользуемся - crf (через pycrfsuite). Для этого делаем bio-разметку данных.
Признаки слов, которые подаем на вход в crf: лемма, часть речи, регистр (istitle, islower, issuper) самого слова и его соседей справа и слева.

Эксперименты:

1) Добавление tfidf и word2vec (ruscorpora_mystem_cbow_300_2_2015.bin.gz) в признаки (смотрим вектора лемм, а не самих слов). Результат практически не изменился (возможно, их надо подавать по-другому, но не нашли как, или эта модель не предназначена для обучения на них).

2) Изменение алгоритмов crf (lbfgs, l2sgd, ap, pa, arow). Дают похожие результаты, лучше всего работают lbfgs, l2sgd  (и у pa лучшая полнота, но хуже все остальное). Неплохие точность и accuracy, хуже всего показатели полноты. Поэтому пробуем добавить информацию из словарей (пункт 3). 

3) Добавление информации из словарей. Составляем правила с помощью yargy и добавляем то, что он извлек, к результатам crf. В результате для всех алгоритмов полнота действительно возрастает на 0.3-0.5, но ожидаемо падают точность и accuracy. 

Датасеты, которые для этого использовались:

1) Датасет со списом заведений общественного питания Москвы от правительства Москвы 2015 г.. Из него брались названия заведений и их тип для категории Whole. Давал совсем плохие результаты на yargy, поэтому не использовался в дальнейшем. [cafes.csv](https://github.com/ddrodionova/NLP_ABSA_project/tree/main/category/cafes.csv) [Источник](https://data.gov.ru/opendata/7710881420-obshchestvennoe)

2) Датасет с 147000 рецептами с сайта povarenok.ru. Использовалась только колонка с ингредиентами (брались названия продуктов), поскольку названия блюд были слишком длинные и их нужно было дополнительно парсить, но при попытке выделить NP из них с помощью yargy компьютер зависал и падал. (слишком большой файл для гитхаба, поэтому скачайте из источника) [Источник](https://www.kaggle.com/rogozinushka/povarenok-recipes)

3) Семантический датасет от КартыСлов (есть набор слов, где каждому соответсутвует тег какой-то семантической категории). Брались слова с тегами FOOD (для категории Food) и CONSTRUCTION (для категории Interior, поскольку там встречалась мебель) [semantic_simple.csv](https://github.com/ddrodionova/NLP_ABSA_project/tree/main/category/semantic_simple.csv) [Источник](https://raw.githubusercontent.com/dkulagin/kartaslov/master/dataset/open_semantics/simple/semantics_simple.csv)

Результаты хранятся в файлах вида pred_asp.csv (без всего), pred_asp_arg.csv (векторы+алгоритмы) и pred_asp_arg_dict.csv (векторы+алгоритмы+словари), где вместо arg -  название алгоритм.

Лучшие результаты по заданным метрикам (pred_asp_lbfgs):

```
Full match precision: 0.7906724511930586
Full match recall: 0.6126050420168068
Partial match ratio in pred: 0.8893709327548807
Full category accuracy: 0.7657266811279827
Partial category accuracy: 0.8828633405639913
```

Судя по этим метрикам + по другим метрикам для crf, самые большие проблемы возникают с аспектами, в которых больше 1 токена, плохо предсказываются I- теги.
## Baseline 2: тональность упоминания аспекта
### Метод
Использовали для определения тональности Bert. Суть метода: тональность классифицируется по CLS токену, а на упоминание аспекта вешается маска. Дополнительные корпуса мы не использовали.

### Результаты
В первой версии accuracy чуть хуже бейслайна (около 0.66). Во второй версии мы использовали stratify и выкинули часть положительных аспектов, чтобы сбалансировать выборку —— но в итоге выборка получалась очень маленькой и модель обучилась даже немного хуже.
#### Проблемы
Выборка не очень большая и очень несбалансированная: положительных отзывов в несколько раз больше всех остальных - если совершенно случайно делить обучающую и тестовую выборку —— то легко может получится, что модель будет обучаться только на положительных данных. 
#### Идеи решений
Добавить корпуса, синонимы.

## Baseline 3: тональность от категориии

### Метод
Тональность по категории определяли по агрегату тональности аспектов, в отличие от бейслайна - при наличии разных оценок учитывалась доля оценки: если она больше 0.6 - то она доминирует и побеждает (например, 9 positive и 2 negative дадут positive, а не both). Это небольшое изменение уже улучшает Overall sentiment accuracy - на данных бейслайна поднялась на 0.18. 

### Результаты
#### На бейслайне
На данных из бейслайна много ошибок были связаны с несовпадением классов - абсолютно все “absence” были неверными (absence тестовой в предсказанных имела оценку и наоборот absence в предсказанных в тестовой имела оценку).
Стоит отметить, что выборка довольно несбалансированная (positive аспектов и категорий сильно больше, чем других), из-за чего точность получается даже немного выше, если отмечать как positive все категории в которых эта оценка появлялась (на бейслайне: 0.6366197183098592 vs 0.6647887323943662). Можно ошибиться практически во всех neutral, negative и both, они в сумме все равно сильно меньше positive.

#### На наших данных
На наших данных получился Overall sentiment accuracy 0.6253521126760564 - лучше изначального безлайна, но слегка хуже, чем на предсказанных аспектах с бейслайна, виной этому то, что модель определяющая тональность аспектов обучалась на слишком маленькой выборке.

## Участники

* Дарья Измалкова
* Мария Непомнящая
* Дарья Родионова
