---
layout: post
title: Когда данных больше, чем памяти
comments: true
tags: python, big data
---

Все мы любим `numpy` и `pandas` за удобство и богатый функционал. Однако иногда приходится работать с массивами в сотни миллионов или даже миллиарды строк, так что суммарный объем данных измеряется десятками и сотнями гигабайт, и поэтому они уже не вмещаются в оперативную память.

Если есть такая возможность, то лучше всего увеличить объем доступной памяти, дооснастив ваш комьютер или перейдя на более мощную виртуальную машину в облаке. Но иногда такой возможности нет, а анализ все же провести надо, причем без развертывания [Hadoop](http://hadoop.apache.org/), [Spark](http://spark.apache.org/) или монструозных СУБД.

К счастью, не все потеряно, и есть варианты выхода из этой затруднительной ситуации.


## bcolz и bquery
Пакет [bcolz](http://bcolz.blosc.org/) основан на библиотеке очень быстрого сжатия данных [blosc](http://www.blosc.org/) и содержит две структуры данных:

- **carray** - массив однотипных элементов (аналог [numpy.ndarray](https://docs.scipy.org/doc/numpy/reference/generated/numpy.ndarray.html))
- **ctable** - таблица с разнотипными колонками (аналог [numpy.recarray](https://docs.scipy.org/doc/numpy/reference/generated/numpy.recarray.html) или [pandas.DataFrame](http://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.html))

Но главное отличие в том, что данные хранятся в сжатом виде - `blosc` поддерживает алгоритмы сжатия `lz4`, `zstd`, `snappy`, `zlib`. Поэтому большой массив будет занимать в разы или даже в десятки раз меньше места в памяти. 

Кроме того, обе структуры поддерживают как in-memory, так и disk-persistent хранение, что позволяет работает с данными астрономических размеров.

В репозитории [movielens-bench](https://github.com/Blosc/movielens-bench) собраны [отличные примеры использования bcolz в сравнении с pandas](https://github.com/Blosc/movielens-bench/blob/master/querying-ep14.ipynb).

При этом `bcolz`, конечно же, не обладает всем богатством функционала `numpy`. В частности, сложные свертки с броадкастингом здесь выполнить не удастся. Однако все базовые векторные операции и агрегаты есть - для нормальной работы вполне хватает.
{: .message}

Когда требуются не только арифметические вычисления, но и группировка данных, вам может пригодиться пакет [bquery](https://github.com/visualfabriq/bquery), который основан на `bcolz` и расширяет структуру `ctable`.

Опять же, `bquery.ctable` сильно уступает по функционалу датафреймам из `pandas`. Но для простых операций со столбцами или `groupby` по гигантскому датафрейму смело используйте `bquery`.


## SFrame
Пакет [SFrame](https://github.com/turi-code/SFrame) содержит три структуры данных:

- **SArray** - массив (аналог [numpy.ndarray](https://docs.scipy.org/doc/numpy/reference/generated/numpy.ndarray.html) или [pandas.Series](http://pandas.pydata.org/pandas-docs/stable/generated/pandas.Series.html))
- **SFrame** - таблица, состоящая из столбцов SArray (аналог [pandas.DataFrame](http://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.html))
- **SGraph** - граф, заданный набором вершин и ребер.

Все три структуры поддерживают disk-persistent хранение, поэтому вы можете спокойно работать с огромными массивами, размер которых многократно превышает объем доступной оперативной памяти. Естественно, это негативно скажется на производительности. Зато ваш код будет прост и понятен, а вам не нужно будет волноваться о количестве строк и столбцов - ваше приложение не упадет с ошибкой _Out of Memory_.

Еще одно маленькое, но важное удобство - функция [SFrame.read_csv](https://turi.com/products/create/docs/generated/graphlab.SFrame.read_csv.html) принимает не только имя файла, но и объект [glob](https://docs.python.org/2/library/glob.html). А это значит, что вы можете загружать данные сразу из многих файлов (например, всех файлов из каталога `"./data"` или файлов, заданных шаблоном `"data_2016_*.csv"`)

После загрузки данных работаете почти как с обычным `pandas`-датафреймом:

```python
sf_visits['duration'] = sf_visits['START_TIME'] - sf_visits['END_TIME']

sf_visits.groupby(['patient_id'], 
       {'total_visits': sframe.aggregate.COUNT('visit_id'),
        'total_duration': sframe.aggregate.SUM('duration')})
```

Единственный существенный недостаток - в настоящий момент пакет SFramе нельзя установить в python 3.5 под Windows. Но в 2.7 и 3.4 все работает. Под Linux тоже все работает даже в python 3.5
{: .message}


## dask
[Dask](http://dask.pydata.org/) - это очень мощная библиотека для параллельных вычислений. И мы еще не раз будем про нее рассказывать. Сейчас же упомянем, что в ней есть две структуры данных:

- [Array](http://dask.pydata.org/en/latest/array.html) (параллельный вариант [numpy.ndarray](https://docs.scipy.org/doc/numpy/reference/generated/numpy.ndarray.html))
- [DataFrame](http://dask.pydata.org/en/latest/dataframe.html) (параллельный вариант [pandas.DataFrame](http://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.html)))

Понятно, что `Array` и `DataFrame` не содержат полный набор методов `numpy` и `pandas`. Зато они позволяют удобно и легко работать с массивами гигантских размеров, которые не вмещаются в оперативную память.

```python
import dask.dataframe as dd

# создаем общий датафрейм из 100 csv-файлов
df_visits = dd.read_csv('visits_2015_*.csv')

df_visit_count = df_visits.groupby(['patient_id'])['visit_id']
                       .count().compute())
```

Кроме того, `dask` умеет делать распределенные вычисления с помощью библиотеки [distributed](https://distributed.readthedocs.io).

---
В общем, огромные массивы данных это совсем не страшно, и вы легко можете с ними справиться даже на самом простом компьютере. Причем вам не понадобится настраивать сложные системы типа [Hadoop](http://hadoop.apache.org/) или [Spark](http://spark.apache.org/). Достаточно установить дополнительный пакет в python и затем вы сможете работать с большими данными в вашем обычном jupyter-ноутбуке.
