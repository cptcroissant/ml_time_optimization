# Оптимизация машинного обучения  
Статья для портала NewNechAudit. Опубликована на [хабре](https://habr.com/ru/post/563494/).

### Datascience – это не только fit-predict

Представим, что вы начали работать в компании, которая производит однообразные операции с бесконечными таблицами. Например, в крупном ретейлере или у ведущего оператора связи. Ежедневно перед вами ставят задачу выяснить, останется ли клиент с вами или хватит ли товара на полках до конца недели. Алгоритм выглядит просто. Вы берете выборку, изучаете бесконечные ряды признаков, удаляете мусор, генерируете новые признаки, собираете сводную таблицу. Подаете готовые данные в модель, настраиваете параметры и с нетерпением ждете заветных цифр итоговой метрики. Это повторяется день за днем. Затрачивая каждый день всего 60 минут на генерацию фич или подбор параметров, за месяц вы израсходуете минимум 20 часов. Это, без малого, целые сутки, за которые можно выполнить новую задачу, обучить нейросеть или прочесть несколько статей на arxiv’e.

Удобно, когда структура данных не меняется. Стабильный набор лейблов и признаков каждый день. Вы понимаете алгоритм обработки и выстраиваете пайплайн. Однообразные таблички со знакомыми признаками начинают обрабатываться без вашего участия. Сложности начинаются в момент, когда признаки в данных становятся разными от задачи к задаче. Или, что еще страшнее, фич становится мало и модель начинает выдавать низкие метрики. Надо снова тратить время на предобработку. Рутина поглощает, блеск в глазах пропадает, продуктивность падает. Вы не первый, кто сталкивался с такими проблемами. Разработчики выкладывают в открытый доступ библиотеки, которые помогают автоматизировать однообразные операции.

### TL:DR

В статье мы рассмотрим применение трех решений оптимизации рабочего процесса: генератор признаков featuretools, подборщик гиперпараметров optuna и коробочное решение автоматического машинного обучения от H2O AutoML.

Сравним работу новых инструментов против классических pandas и sklearn. Не будем глубоко закапываться в код. Приведем основные моменты и оставим анализ полного функционала в качестве домашнего задания. Статья будет полезна DS-специалистам и аналитикам. Требуется понимание основных шагов процесса машинного обучения.

В качестве примера возьмем датасет с платформы [kaggle](https://kaggle.com/kingoffitpredict/time-optimization). Простые синтетические данные содержат только числовые значения без пропусков. Kaggle позволит применить изучаемые библиотеки сразу, без дополнительной установки. Для наглядной валидации результатов создадим три сабмита в январском соревновании. Сравним на тестовых данных предсказания простенькой настроенной вручную модели, модели с оптимизированными optuna гиперпараметрами и результат работы автоматизированного машинного обучения.

### Нужно больше данных

Начнем с генерации признаков. Зачем вообще нужны новые фичи в машинном обучении? Они агрегируют данные, дают новую информацию для анализа и улучшают качество работы моделей. Например, методы groupby() и sum() – это рядовые представители сгенерированных признаков. Классический подход инжиниринга данных – изучить датасет вручную и написать код для каждого нового признака. Такой метод практичен, но отнимает много времени и велика вероятность ошибки. Особенно, когда однообразного кода становится много.

На помощь приходит [featuretools](https://featuretools.alteryx.com/en/stable) (англ. дословно «инструмент для признаков»). Фреймворк позволяет производить параллельные вычисления, создавать сложные признаки и работать сразу с несколькими таблицами.

Инструмент предлагает два направления генерации фич: feature primitives (fp, англ. «примитивные признаки») и deep feature synthesis (dfs, англ. «глубокий синтез признаков»). Первый метод производит простые математические операции между определенными фичами. Второй, более сложный, позволяет соединять несколько примитивных операций над признаками. Каждая новая операция увеличивает глубину (“depth”) сложности алгоритма. Например, мы можем посчитать сначала все средние значения по определенному пользователю, а затем сразу суммировать их.

Переходим к практическому применению. Установка и импорт.

```
pip install featuretools
import featuretools as ft
```

Featuretools обладают очень удобным API и встроенной документацией. Для вызова возможных функций, просто запустите следующий код:

ft.primitives.list_primitives()

На выходе вы получите список всех возможных «примитивных» функций с их описанием и возможностью применения данным.

В нашем примере мы рассмотрим применение библиотеки с одним датасетом. В несколько строк кода мы создадим попарно умноженные и попарно сложенные исходные признаки. Работа featuretools начинается с создания класса EntitySet (англ. «множество сущностей»), которому обязательно присвоить id. Id используется системой для идентификации определенной сущности, когда вы работаете сразу с несколькими источниками. Затем последовательно добавляем в него имеющиеся датафреймы, в нашем случае – единственный. И запускаем автоматическую генерацию признаков.

```
es = ft.EntitySet(id=’data’)                                                       # Новый пустой EntitySet
es.entity_from_dataframe(entity_id = ‘january’,                                    # Добавляем в него информацию
			      dataframe=df_train.drop(‘target’, axis=1),
			      index=1)
feature_matrix, feature_defs = ft.dfs(entityset=es,                                # Запускаем генерацию признаков
				          target_entity=’january’,
				          trans_primitives=[‘add_number’, ‘multiply_numeric’],
				          verbose=1)
```                  

Вспомните, сколько строк кода вам пришлось бы писать для подобной обработки на pandas. А если бы мы захотели получить более «тяжелые» признаки? После генерации фич мы получаем pandas датафрейм feature_matrix, который можно сразу отправлять в модель. Ничего конвертировать дополнительно не нужно.

Всю мощь featuretools раскрывает при обработке нескольких таблиц. Подробный туториал с примерами кода есть на [официальном сайте](https://featuretools.alteryx.com/en/getting_started/afe.html). Рекомендую к ежедневному использованию.

### Искусство легких настроек

После предобработки данных переходим к обучению и настройке параметров модели. Классические и современные методы оптимизации с виду действуют одинаково. Мы передаем им данные, модель и таблицу со значениями гиперпараметров. На выходе получаем комбинацию атрибутов, с которыми модель показывает наилучшую метрику.

Самые известные инструменты поиска гиперпараметров из sklearn — GridSearchCV и RandomizedSearchCV. GridSearchCV перебирает все возможные комбинации параметров на кросс-валидации. RandomizedSearchCV сначала создает словарь с несколькими случайно выбранными параметрами из всех переданных значений. Затем алгоритм тестирует отобранные значения на моделях и выбирает лучшие. Первый метод является крайне ресурсозатратным. Второй быстрее, но мало применим к сложным ансамблям градиентного бустинга. Для обучения серьезных моделей классические алгоритмы не подходят.

Бытует мнение, что истинные профи не запускают подбор, а с первого раза вбивают нужные параметры, просто смотря на входные данные. Для рядовых специалистов есть несколько «коробочных» решений: optuna, hyperopt, scikit-optimization.

Разберем применения самого быстрого и самого молодого из них – optuna.

Кроме настройки классических моделей sklearn и ансамблей бустинга (xgboost, lightgbm, catboost), фреймворк позволяет настраивать нейросети, написанные на pytorch, tensorflow, chainer и mxnet. В отличие от классических sklearn методов, optuna, вместе с другими новыми фреймворками, может обрабатывать непрерывные значения гиперпараметров. Например, альфа- или лямбда-регуляризации будут принимать любые значения с плавающей точкой в заданном диапазоне. Все это делает его одним из самых гибких инструментов настройки моделей глубокого обучения.

В своей работе optuna использует байесовские алгоритмы подбора с возможностью удаления заведомо проигрышного пространства заданных гиперпараметров из анализа. Рассмотрим практическое применение фреймворка. Как библиотека устроена «под капотом» на английском языке можно прочесть на [arxiv.org](https://arxiv.org/pdf/1907.10902.pdf). На русском языке есть подробная статья на [Хабре](https://habr.com/ru/company/antiplagiat/blog/528384). В ней вы найдете математическое описание методов оптимизации практически на пальцах.

Перейдем к написанию кода. В качестве примера рассмотрим настройку градиентного бустинга LightGBM. Данные будем использовать в первозданном виде. Напомню, что код к статье вы можете найти [здесь](https://kaggle.com/kingoffitpredict/time-optimization).

```
# установка (при необходимости)
pip install optuna
# импорт библиотеки и средств визуализации
import optuna
from optuna.visualization import plot_optimization_history, plot_param_importances
```

Посмотрим на работу модели без подбора гиперпараметров. Базовая метрика RMSE на приватном списке победителей = 0.73562. Запустим optuna и посмотрим на улучшение метрики за 5 итераций. Имеющиеся данные достаточно простые и мы не ждем глобального скачка качества после подбора гиперпараметров. Нам интересен сам процесс и скорость его работы.

Сетка гиперпараметров задается несколькими типами значений – непрерывными, категориальными и количественными. Разница в их оформлении выглядит так:

```
‘reg_alpha’: trial.suggest_loguniform(‘reg_alpha’, 1e-3, 10.0) 
 # передаем нижнюю и верхнюю границы непрерывного параметра
‘num_leaves’: trial.suggest_int(‘num_leaves’, 1, 300)
# передаем нижнее и верхнее значение числового параметра
‘max_depth’: trial.suggest_categorical(‘max_depth’, [-1,10,20]) 
# передаем список определенных значений категориального признака, которые надо проверить
```

Сам процесс обучения прописывается в две строки:

```
study = optuna.create_study(direction=’minimize’)  # минимизируем ошибку
study.optimize(objective, n_trials=5) 		      # objective – задание для поиска, 5 – количество
      # итераций оптимизации  
```

В приложенном к статье kaggle ноутбуке рассматривается базовый вариант запуска optuna. За 5 итераций удалось улучшить метрику на отложенной выборке на 0.1 пункта метрики RMSE. Процесс длился 5 минут. Интересно, сколько по времени работал бы традиционный GridSearchCV с данным количеством параметров. Метрика на финальном сабмите с оптимизацией = 0.7221, что лучше ручной настройки модели на сырых данных.

### Machine Learning за 7 строк кода

Такой заголовок подошел бы для кликбейтовой рекламы в интернете. Но мы действительно создадим полный пайплайн обучения за минимальное количество блоков кода. Поможет нам в этом [h2о](https://docs.h2o.ai/h2o/latest-stable/h2o-docs/welcome.html) от Amazon. Это библиотека содержит в себе все популярные модули обработки данных и машинного обучения. Разберем модуль автоматического машинного обучения — [automl](https://docs.h2o.ai/h2o/latest_stable/h2o-docs/automl.html). Фреймворк позволяет подбирать модель и параметры без участия специалиста.

Сегодня мы отправим в обработку сырой датафрейм в силу его простоты. В ежедневных задачах данным все же требуется небольшая предподготовка. Для нее удобно использовать [встроенные алгоритмы](https://docs.h2o.ai/h2o/latest_stable/h2o-docs/data-munging.html). Начнем с установки и импорта модулей.

```
pip install –f https://h2o-release.s3.amazonaws.com/h2o/latest_stable_Py.html h2o
 from h2o.automl import H2OAutoML
# установим максимальный размер используемой оперативной памяти
h2o.init(max_mem_size=’16G’)
```

Pandas больше не нужен! Можно загружать данные в рабочий ноутбук с помощью AutoML.

```
train = h20.import_file(‘../input/tabular-playground-series-jan-21/train.csv’)
test = h20.import_file(‘../input/tabular-playground-series-jan-21/test.csv’)
```

Выберем признаки и таргет.

```
x = test.columns[1:]
y = ‘target’
```

Запускаем машинное обучение. Библиотека будет последовательно обучать встроенные модели с различными гиперпараметрами. В нашем примере настроим количество моделей, random_state=47 и максимальное время обучения одной модели = 3100 секунд (3600 секунд по умолчанию)

```
aml = H20AutoML(max_models=2,  		# Количество различных моделей для обучения
                                 seed=SEED, 
                                 max_runtime_secs = 3100)   # Максимальное время обучения одной модели
aml.train(x=x, y=y, training_frame=train)
# Ждем, когда заполнится строка обучения AutoML
```

Готово! Параметры подобраны, модели выбраны. Все сделано в автоматическом режиме. Можно посмотреть на метрики лучших моделей или вывести информацию о наилучшей. Подробный код смотрите в учебном ноутбуке на [kaggle](https://kaggle.com/kingoffitpredict/time-optimization). Остается создать и отправить на валидацию предсказания модели. Это единственный случай, когда потребуется загруженный pandas.

```
preds = aml.predict(test)
 df_sub[‘target’] = preds.as_data_frame().values_flatten()
 df_sub.to_csv(‘h2o_submission_5.csv, index=False)
```

Финальная метрика AutoMl на приватном списке победителей = 0.71487. Напомню, что метрика среднеквадратичной ошибки (root mean squared error, RMSE) отражает среднее отклонение предсказаний от реального значения. Простыми словами – она должна быть минимальной и стремиться к нулю. RMSE у победителя соревнования = 0.69381. В пределах рутинных задач с которыми можно столкнуться в ежедневной работе – разница так же невелика. Но время AutoML экономит значительно.

### Заключение

Мы рассмотрели три направления оптимизации рабочих процессов: машинное обучение без участия специалиста, автоматическую генерацию новых признаков и подбор гиперпараметров модели. Базовые функции инструментов обладаю простым синтаксисом, находятся в открытом доступе и позволяют экономить драгоценное время.

Оптимизация параметров и автоматизированное обучение повысили качество финальной метрики RMSE в сравнении с базовой моделью, на 0.2 пункта. Для простых синтетических данных это адекватный результат, который демонстрирует применимость методов к серьезным задачам.

Конечно, изученные фреймворки не могут полностью заменить грамотного специалиста, но помогают в ежедневной рутинной деятельности. Настоятельно рекомендую изучить полную документацию каждой библиотеки. Используйте время с умом, работайте с удовольствием.
