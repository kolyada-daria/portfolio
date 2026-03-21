# Лабораторная работа №10: Методы оптимизации вычисления кода с помощью потоков, процессов, Cython, отпускания GIL

### Цель работы
Исследовать методы оптимизации вычисления кода, используя потоки, процессы, Cython и отключение GIL на основе сравнения времени вычисления функции численного интегрирования методом прямоугольников, реализованной на чистом Python.

### Реализация

Дана функция:

```python
import math

# итерация 1
def integrate(f, a, b, *, n_iter=100000):
  acc = 0
  step = (b - a) / n_iter
  for i in range(n_iter):
    acc += f(a + i*step) * step
  return acc

integrate(math.cos, 0, math.pi, n_iter=1000)
```

#### Итерация 1

```python
import math import timeit

def integrate(f, a: float, b: float, *, n_iter=100000) -> float:
    """
    Реализует вычисление суммы площадей прямоугольников под графиком функции f на отрезке от a до b.
    Метод разбивает отрезок на n_iter частей и суммирует площади левых прямоугольников.
    :param f: интегрируемая функция
    :param a: начало отрезка
    :param b: конец отрезка
    :param n_iter: количество шагов
    :return: приблизительное значение площади под графиком """
    acc = 0
    step = (b - a) / n_iter for i in range(n_iter):
        acc += f(a + i*step) * step return acc
        iters_to_test = [1000, 10000, 100000, 1000000] 
        for n in iters_to_test:
            # timeit.timeit создает тест и замеряет, сколько в среднем занимает 10 запусков функции
            t = timeit.timeit(lambda: integrate(math.sin, 0, math.pi, n_iter=n), number=10)
            average_time = t / 10 
            print(n, average_time)
```

#### Итерация 2. Оптимизация с помощью потоков

```python
def integrate_async(f, a: float, b: float, *, n_jobs: int=2, n_iter: int=1000) -> float:
    """
    Проводит вычисления определенного интеграла функции f параллельно, используя процессы.

    Метод разделяет интервал от a до b на n_jobs равных отрезков.
    Каждый отрезок обрабатывается в отдельном потоке через ThreadPoolExecutor.
    Общее количество итераций распределяется между потоками.

    :param f: интегрируемая функция
    :param a: начало отрезка
    :param b: конец отрезка
    :param n_jobs: количество параллельных процессов
    :param n_iter: количество итераций (потоков)
    :return: приблизительное значение интеграла, сумма результатов всех процессов
    """

    # Создаваемый пул тредов размера n_jobs
    executor = ftres.ThreadPoolExecutor(max_workers=n_jobs)

    spawn = partial(executor.submit, integrate, f, n_iter = n_iter // n_jobs)

    # Определяем ширину зоны ответственности для каждого процесса
    step = (b - a) / n_jobs
    for i in range(n_jobs):
      print(f"Работник {i}, границы: {a + i * step}, {a + (i + 1) * step}")

    # Создание потоков с помощью генератора списков
    fs = [spawn(a + i * step, a + (i + 1) * step) for i in range(n_jobs)]

    # as_completed() возвращает фьючерсы по мере завершения, затем суммируем результаты f.result()
    return sum(list(f.result() for f in ftres.as_completed(fs)))


print(integrate_async(math.sin, 0, math.pi))
```

#### Итерация 3. Оптимизация с помощью процессов

```python
def integrate_processes(f, a: float, b: float, *, n_jobs: int = 2, n_iter: int = 1000) -> float:
    """
    Вычисляет интеграл функции параллельно с использованием процессов.
    Функция разделяет отрезок от a до b на n_jobs равных отрезков и проводит вычисления для каждого из них.

    :param f: интегрируемая функция
    :param a: начало отрезка
    :param b: конец отрезка
    :param n_jobs: количество процессов
    :param n_iter: количество итераций, распределяемое между процессами
    :return:
    """
    # with, чтобы пул корректно закрывался сам
    with ftres.ProcessPoolExecutor(max_workers=n_jobs) as executor:
        step = (b - a) / n_jobs
        # Распределение нагрузки: каждый процесс делает часть общего n_iter
        spawn = partial(executor.submit, integrate, f, n_iter=n_iter // n_jobs)

        fs = [spawn(a + i * step, a + (i + 1) * step) for i in range(n_jobs)]
        # Собираем и суммируем результаты
        return sum(f_res.result() for f_res in ftres.as_completed(fs))
```

* Запуск итерации 2 и 3 для замера времени выполнения:

```python
if __name__ == "__main__":
    N = 10  
    print(f"Запуск на {N} итераций...\n")

    for jobs in [2, 3, 4]:
        start = time.perf_counter()
        res1 = integrate_processes(math.sin, 0, math.pi, n_jobs=jobs, n_iter=N)
        res2 = integrate_async(math.sin, 0, math.pi, n_jobs=jobs, n_iter=N)
        end = time.perf_counter()
        print(f"Ядер: {jobs} | Время: {end - start:.4f} сек | Результат: {res1}")
        print(f"Ядер: {jobs} | Время: {end - start:.4f} сек | Результат: {res2}")
```

#### Итерация 4
??? info "Описание"

    1. Профилирование и оптимизация функции integrate
    2. Оптимизировать функцию integrate с помощью Cython
    3. Замерить время вычисления функции без потоков и процессов (сравнить с итерацией 1)
    4. Замерить время вычисления с потоками и процессами (сравнить с итерациями 2 и 3 соответственно)
    5. Использовать annotate = True получить html-файл для модуля integrate и максимально
    6. Оптимизировать код для уменьшения взаимодействия с C-API

```python
# cython: language_level=3, boundscheck=False, wraparound=False, cdivision=True
from libc.math cimport sin
def integrate_cy(double a, double b, int n_iter): 
    cdef double acc = 0.0
    cdef double step = (b - a) / n_iter cdef int i
    
    for i in range(n_iter):
        acc += sin(a + i * step) * step return acc
```

#### Итерация 5.
??? info "Описание"

    1. Переписать код функции integrate с использованием noGIL.
    2. Сделать замеры позволяющие оценить время выполнения кода с 2, 4, 6 потоками и сравнить
    3. Время вычисления с помощью потоков integrate без GIL (noGIL ) и сайтонизированной с
    4. Временем вычисления с помощью процессов сайтонизированной версии


```python
# cython: language_level=3, boundscheck=False, wraparound=False, cdivision=True
from libc.math cimport sin
def integrate_nogil(double a, double b, int n_iter): 
    cdef double acc = 0.0
    cdef double step = (b - a) / n_iter cdef int i
    
    with nogil:
        for i in range(n_iter):
            acc += sin(a + i * step) * step return acc
```

### Общий вывод
Анализ производительности различных методов вычисления интеграла показал, что стандартные потоки в Python ограничены механизмом GIL и не дают прироста скорости в вычислительных задачах. 
Использование многопроцессорности позволяет обойти это ограничение, ускоряя расчеты пропорционально количеству ядер. 
Так, наиболее значительный эффект дает Cython. 
За счет компиляции кода в C он работает эффективнее обычного интерпретатора. 
В финальной пятой итерации режим noGIL позволил потокам работать параллельно, что сделало их быстрее процессов из-за отсутствия задержек на запуск. 
В итоге комбинация Cython и многопоточности показала максимальную эффективность.


### Ссылка на репозиторий
[Репозиторий проекта](https://github.com/kolyada-daria/py_itmo/tree/master/lab10)