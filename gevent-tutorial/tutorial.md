[TOC]

# Введение

[Оригинал](http://sdiehl.github.io/gevent-tutorial/) на англисйком языке находится здесь http://sdiehl.github.io/gevent-tutorial/

Структура данного руководства предполагает средний уровень владения Python, но не больше. Никаких знаний в области параллельной обработки и конкурентных задач не требуется. Цель -- дать вам необходимые инструменты для работы с gevent, помочь приручить нынешние проблемы одновременных вычислений и начать писать асинхронные программы.

### Соучастники

В хронологическом порядке:
[Stephen Diehl](http://www.stephendiehl.com)
[J&eacute;r&eacute;my Bethmont](https://github.com/jerem)
[sww](https://github.com/sww)
[Bruno Bigras](https://github.com/brunoqc)
[David Ripton](https://github.com/dripton)
[Travis Cline](https://github.com/traviscline)
[Boris Feld](https://github.com/Lothiraldan)
[youngsterxyf](https://github.com/youngsterxyf)
[Eddie Hebert](https://github.com/ehebert)
[Alexis Metaireau](http://notmyidea.org)
[Daniel Velkov](https://github.com/djv)
[Sean Wang](https://github.com/sww)
[Inada Naoki](https://github.com/methane)
[Balthazar Rouberol](https://github.com/brouberol)
[Glen Baker](https://github.com/iepathos)
[Jan-Philip Gehrcke](https://gehrcke.de)
[Matthijs van der Vleuten](https://github.com/zr40)
[Simon Hayward](http://simonsblog.co.uk)
[Alexander James Phillips](https://github.com/AJamesPhillips)
[Ramiro Morales](https://github.com/ramiro)
[Philip Damra](https://github.com/djheru)
[Francisco Jos&eacute; Marques Vieira](https://github.com/fvieira)
[David Xia](https://www.davidxia.com)
[satoru](https://github.com/satoru)
[James Summerfield](https://github.com/jsummerfield)
[Adam Szkoda](https://github.com/adaszko)
[Roy Smith](https://github.com/roysmith)
[Jianbin Wei](https://github.com/jianbin-netskope)
[Anton Larkin](https://github.com/ToxicWar)
[Matias Herranz](https://github.com/matiasherranz-santex)
[Pietro Bertera](http://www.bertera.it)
[Vladimir Donets](http://vovkd.github.io/gevent-tutorial)

Также спасибо Денису Биленко за создание gevent и помощь в написании этого руководства.

# Основы

## Параллелизм

Примечание — В русскоязычной литературе нередко путаются термины «параллелизм» и «конкурентность». Оба термина означают одновременность процессов, но первый — на физическом уровне (параллельное исполнение нескольких процессов, нацеленное только на повышение скорости исполнения за счёт использования соответствующей аппаратной поддержки), а второй — на логическом (парадигма проектирования систем, идентифицирующая процессы как независимые, что в том числе позволяет их исполнять физически параллельно, но в первую очередь нацелено на упрощение написания многопоточных программ и повышение их устойчивости).

В англоязычной литературе существует два понятия параллельности: Concurrency и Parallelism. Термин Concurrency будет переводиться «одновременность», а термин Parallelism будет переводиться «параллелизм». 

[ВИКИ](https://ru.wikipedia.org/wiki/%D0%9F%D0%B0%D1%80%D0%B0%D0%BB%D0%BB%D0%B5%D0%BB%D0%B8%D0%B7%D0%BC_(%D0%B8%D0%BD%D1%84%D0%BE%D1%80%D0%BC%D0%B0%D1%82%D0%B8%D0%BA%D0%B0))

## Гринлеты (Greenlets)

Основной паттерн, который используется в gevent -- это Greenlet (гринлет), этакие микропотоки. Все гринлеты  запускаются внутри основного процесса программы и координируются внутри него, представляя собой некоторое подобие потоков OS, но не использующих ресурсы OS.
Поэтому в любой момент времени работает только один гринлет.
Изначально у гринлетов нет своего планировщика исполнения.

Greenlet реализован на С в виде модуля расширения.

Ссылки
https://greenlet.readthedocs.io/en/latest/
http://www.slideshare.net/saghul/understanding-greenlet

## Синхронное и асинхронное исполнение

Основная идея одноврменных вычислений в том, что большую задачу можно разбить на несколько маленьких подзадач и затем выполнить их либо асинхронно, либо одновременно, вместо их последовательного выполнения. Переключение между такими подзадачами называется переключением контекста (context switch)

Переключение контекста в gevent сделано через использование yield. В следующем примере у нас два контекста, которые переключаются между собой с помощью вызова gevent.sleep(0)      

[[[cog
import gevent

def foo():
    print('Running in foo')
    gevent.sleep(0)
    print('Explicit context switch to foo again')

def bar():
    print('Explicit context to bar')
    gevent.sleep(0)
    print('Implicit context switch back to bar')

gevent.joinall([
    gevent.spawn(foo),
    gevent.spawn(bar),
])

]]]
[[[end]]]

На картинке ниже показан процесс переключения между контекстами. Его также можно отследить с использованием отладчика.

![Greenlet Control Flow](flow.gif)

### Поток исполнения Greenlet

Свои настоящие возможности gevent показывает, когда применятся для работы с сетью (network) или задачами ввода-вывода (I/O tasks), процесс выполнения которых может эффективно координироваться планировщиком.

Gevent заботится обо всех деталях, гарантируя, что ваши сетевые библиотеки будут переключать свой greenlet-контекст в любой подвернувшийся для этого момент времени.

Я не знаю как выразить насколько это сильная идея. Однако, может следующий пример покажет.
В нём функция ``select()``, которая обычно блокирует поток исполнения, опрашивает различные файловые дескрипторы.

[[[cog
import time
import gevent
from gevent import select

start = time.time()
tic = lambda: 'at %1.1f seconds' % (time.time() - start)

def gr1():
    # Busy waits for a second, but we don't want to stick around...
    print('Started Polling: %s' % tic())
    select.select([], [], [], 2)
    print('Ended Polling: %s' % tic())

def gr2():
    # Busy waits for a second, but we don't want to stick around...
    print('Started Polling: %s' % tic())
    select.select([], [], [], 2)
    print('Ended Polling: %s' % tic())

def gr3():
    print("Hey lets do some stuff while the greenlets poll, %s" % tic())
    gevent.sleep(1)

gevent.joinall([
    gevent.spawn(gr1),
    gevent.spawn(gr2),
    gevent.spawn(gr3),
])
]]]
[[[end]]]

Далее очередной искусственный пример, в котором определяется недетерминированная функция ``task`` (т.е. результат работы этой функции не будет гарантированно тем же самым на одних и тех же входных данных). В данном случае побочный эффект заключается в том, что функция останавливает своё исполнение на случайное число секунд:

[[[cog
import gevent
import random

def task(pid):
    """
    Some non-deterministic task
    """
    gevent.sleep(random.randint(0,2)*0.001)
    print('Task %s done' % pid)

def synchronous():
    for i in range(1,10):
        task(i)

def asynchronous():
    threads = [gevent.spawn(task, i) for i in xrange(10)]
    gevent.joinall(threads)

print('Synchronous:')
synchronous()

print('Asynchronous:')
asynchronous()
]]]
[[[end]]]

 В случае синхронного исполнения все задачи выполняются последовательно, одна за другой. Это блокирует исполнение основной программы, пока исполняется каждая подзадача.

Важная часть программы выше ф-я gevent.spawn, которая помещает исполнение подзадачи в greenlet-микропоток. Список инициализированных гринлетов хранится в массиве потоков, который передаётся gevent.joinall, которая блокирует исполнение основной программы для выполнения переданных ей гринлетов. Исполнение основной программы будет продолжено лишь тогда, когда будут завершены все гринлеты.

Следует отметить, что порядок выполнения в случае асинхронного исполнения абсолютно случайный, а также что общее время выполнения 
в асинхронном варианте значительно меньше времени выполнения списка задач синхронно. К примеру, общее время исполнения синхронного кода, когда каждая подзадача спит "0.002" сек. равно 0.02 сек, а время выполнение той же очереди задач асинхронно равняется 0.002, так как в этом случае ни одна из подзадач не блокирует выполнение другх.

В более типичном случае, когда мы получаем данные с сервера асинхронно, время выполнения fetch() для каждого запроса будет различаться и зависеть от времени исполнения запроса и общей нагрузки на этот сервер:

<pre><code class="python">import gevent.monkey
gevent.monkey.patch_socket()

import gevent
import urllib2
import simplejson as json

def fetch(pid):
    response = urllib2.urlopen('http://jsontime.herokuapp.com/')
    result = response.read()
    json_result = json.loads(result)
    datetime = json_result['datetime']

    print('Process %s: %s' % (pid, datetime))
    return json_result['datetime']

def synchronous():
    for i in range(1,10):
        fetch(i)

def asynchronous():
    threads = []
    for i in range(1,10):
        threads.append(gevent.spawn(fetch, i))
    gevent.joinall(threads)

print('Synchronous:')
synchronous()

print('Asynchronous:')
asynchronous()
</code>
</pre>

## Детерменированность

Как уже говорилось ранее, гринлеты детерминированы. При одинаковых входных данных мы получим одинаковые выходные результаты. Попробуем распределить задачи между пулом из multiprocessing и сравнить результат работы с обработкой тех же задач с помощью gevent.

<pre>
<code class="python">
import time

def echo(i):
    time.sleep(0.001)
    return i

# Non Deterministic Process Pool

from multiprocessing.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]
run2 = [a for a in p.imap_unordered(echo, xrange(10))]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]

print(run1 == run2 == run3 == run4)

# Детерменированный gevent Pool

from gevent.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]
run2 = [a for a in p.imap_unordered(echo, xrange(10))]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]

print(run1 == run2 == run3 == run4)
</code>
</pre>

<pre>
<code class="python">>>> False # multiprocessing pool
>>> True  # gevent pool</code>
</pre>

Несмотря на то, что gevent обычно детерминирован, источники недетерминированности могут закрасться в программу в те моменты, когда она начинает взаимодействовать с внешними сервисами. Такими как сокеты или файлы. 
Поэтому несмотря на то, что "зеленые потоки" (greenlets, green threads) представляют собой детерминированную форму одновременного исполнения, они все равно могут иметь те же проблемы, что потоки и процессы POSIX.

Одной из вечных проблем одновременного исполнения является "состояние гонки" (race condition). Она возникает, когда два одновременно выполняющихся потока/процесса имеют общий ресурс, который они хотят изменить. В итоге содержание этих ресурсов становятся зависимым от порядка исполнения задач в общих для них процессах/потоках. Это серьезная проблема, поэтому следует избегать её возникновения, так как она ведёт к возникновению общей недетерменированности результата исполнения программы.

Одним из подходов к решению проблемы "гонки" является отказ от использования всех глобальных состояний, которые вместе с побочными эффектами на этапе импорта (import-time side effects) всегда вернуться, чтобы создать проблем.

## Порождение гринлетов (Greenlets)

Gevent предоставляет несколько обёрток вокруг инициализации гринлетов. Самыми часто используемыми шаблонами являются:

[[[cog
import gevent
from gevent import Greenlet

def foo(message, n):
    """
    Каждый поток будет получать при инициализации
    сообщение и время на которое надо заснуть.
    """
    gevent.sleep(n)
    print(message)

# Инициализация экземпляра Greenlet c передачей именнованой функции и её аргументов
# foo
thread1 = Greenlet.spawn(foo, "Hello", 1)

# Инициализация Greenlet с помощью обёртки от gevent
# функция foo с аргументами
thread2 = gevent.spawn(foo, "I live!", 2)

# Инициализация с помощью безымянной лямбда-функции
thread3 = gevent.spawn(lambda x: (x+1), 2)

threads = [thread1, thread2, thread3]

# Блокировка до момента, когда задачи во всех потоках будут завершены
gevent.joinall(threads)
]]]
[[[end]]]

<pre>
<code class="python">
>>> Hello
>>> I live!  
</code>
</pre>

В дополнение к использованию класса Greenlet можно также унаследоваться от него и переопределить метод ``_run``

[[[cog
import gevent
from gevent import Greenlet

class MyGreenlet(Greenlet):

    def __init__(self, message, n):
        Greenlet.__init__(self)
        self.message = message
        self.n = n

    def _run(self):
        print(self.message)
        gevent.sleep(self.n)

g = MyGreenlet("Hi there!", 3)
g.start()
g.join()
]]]
[[[end]]]


## Состояния гринлетов

Как и любой другой код, гринлеты могут ломаться. Гринлет может не выкинуть исключение, не остановиться вовремя или сожрать слишком много системных ресурсов. 

Внутреннее состояние гринлета зависит от времени. Существует небольшой набор флагов для контроля за состоянием гринлета. 

- ``started`` -- boolean, свойство, указывает запущен ли гринлет
- ``ready()`` -- метод, возвращает boolean, указывает заквершил ли работу  
          гринлет
- ``successful()`` -- метод, возвращает boolean, указывает что гринлет     
               завершился без исключений.
- ``value`` -- необязательный, значение, которое возвращает гринлет
- ``exception`` -- исключение, непойманное исключение, которое произошло 
            во время исполнения гринлета

[[[cog
import gevent

def win():
    return 'You win!'

def fail():
    raise Exception('You fail at failing.')

winner = gevent.spawn(win)
loser = gevent.spawn(fail)

print(winner.started) # True
print(loser.started)  # True

# исключения, возникшие внутри гринлета, остаются внутри этого гринлета
try:
    gevent.joinall([winner, loser])
except Exception as e:
    print('This will never be reached')

print(winner.value) # 'You win!'
print(loser.value)  # None

print(winner.ready()) # True
print(loser.ready())  # True

print(winner.successful()) # True
print(loser.successful())  # False

# The exception raised in fail, will not propagate outside the
# greenlet. A stack trace will be printed to stdout but it
# will not unwind the stack of the parent.
print(loser.exception)

# It is possible though to raise the exception again outside
# raise loser.exception
# or with
# loser.get()
]]]
[[[end]]]

## Завершение программы

Гринлеты могут пораждать "зомби"-процессы, если после получения сигнала SIGQUIT основной программой были завершены неверно.

Общепринятый способ завершения заключается в том, чтобы после получения сигнала SIGQUIT выполнять ``gevent.shutdown``

<pre>
<code class="python">import gevent
import signal

def run_forever():
    gevent.sleep(1000)

if __name__ == '__main__':
    gevent.signal(signal.SIGQUIT, gevent.kill)
    thread = gevent.spawn(run_forever)
    thread.join()
</code>
</pre>

## Таймауты

Таймауты -- это ограничения времени исполнения блока кода или гринлета.

<pre>
<code class="python">
import gevent
from gevent import Timeout

seconds = 10

timeout = Timeout(seconds)
timeout.start()

def wait():
    gevent.sleep(10)

try:
    gevent.spawn(wait).join()
except Timeout:
    print('Could not complete')

</code>
</pre>

Их также можно использовать в контекстных менеджерах совместно с ``with`` 

<pre>
<code class="python">import gevent
from gevent import Timeout

time_to_wait = 5 # seconds

class TooLong(Exception):
    pass

with Timeout(time_to_wait, TooLong):
    gevent.sleep(10)
</code>
</pre>

В дополнение gevent предоставляет возможность передавать таймаут в качестве аргумента некоторых видов вызовов. Например:

[[[cog
import gevent
from gevent import Timeout

def wait():
    gevent.sleep(2)

timer = Timeout(1).start()
thread1 = gevent.spawn(wait)

try:
    thread1.join(timeout=timer)
except Timeout:
    print('Thread 1 timed out')

# --

timer = Timeout.start_new(1)
thread2 = gevent.spawn(wait)

try:
    thread2.get(timeout=timer)
except Timeout:
    print('Thread 2 timed out')

# --

try:
    gevent.with_timeout(1, wait)
except Timeout:
    print('Thread 3 timed out')

]]]
[[[end]]]

## Обезьяний патч (Monkeypatching)

Если кто-то не в курсе что это такое, то википедия даёт такое пояснение:
"Monkey patch (обезьяний патч) — в программировании возможность подмены методов и значений атрибутов классов программы во время её выполнения "

Мы подошли к тёмной стороне gevent. Мы избегали разговора про обезьяние патчи до этого момента, чтобы показать силу шаблоны применения одновременного исполнения кода и замотивировать к его применению. Но пришло время обсудить тёмное искуство обезьяних патчей. 

Если ты замечал, то раньше мы не раз вызывали ``monkey.patch_socket()`` -- это функция для модификации стандартной библиотеки для работы с сокетами.

<pre>
<code class="python">import socket
print(socket.socket)

print("After monkey patch")
from gevent import monkey
monkey.patch_socket()
print(socket.socket)

import select
print(select.select)
monkey.patch_select()
print("After monkey patch")
print(select.select)
</code>
</pre>

<pre>
<code class="python">class 'socket.socket'
After monkey patch
class 'gevent.socket.socket'

built-in function select
After monkey patch
function select at 0x1924de8
</code>
</pre>

Python позволяет модифицировать объекты прямо во время исполнения: модули, функции, классы и т.д. Вообще это плохая идея, так как можно получить неожиданные и малоприятные побочные эффекты. 
Gevent в состоянии "пропатчить" в стандартной библиотеке большинство блокирующих системных вызовов, включая те, что находятся в библиотеках ``socket``, ``ssl``, ``threading``, ``select``. Предоставляя им возможность одновременного исполнения. 

К примеру, обычная библиотека для работы с Redis в Python использует обычные TCP-сокеты для общения с сервером Redis. Но после вызова  ``gevent.monkey.patch_all()`` мы получаем возможность библиотеку для работы с Redis координировать обращения к серверу и использовать для этого ``gevent``

Это позволяет нам интегрировать без единой строчки кода библиотеки, которые не умеют работать c gevent. Поэтому несмотря на всю опасность обезьяних патчей, они могут принести существенную пользу,когда применяешь их с умом.

# Структуры данных

## События

События  -- это форма общения между гринлетами.

<pre>
<code class="python">import gevent
from gevent.event import Event

'''
Код иллюстрирует использование событий
'''


evt = Event()

def setter():
    '''Разбудить все потоки, ожидающие событие evt, через 3 сек'''
	print('A: Hey wait for me, I have to do something')
	gevent.sleep(3)
	print("Ok, I'm done")
	evt.set()


def waiter():
	'''Вызов get разблокирует потоки через 3 секунды'''
	print("I'll wait for you")
	evt.wait()  # blocking
	print("It's about time")

def main():
	gevent.joinall([
		gevent.spawn(setter),
		gevent.spawn(waiter),
		gevent.spawn(waiter),
		gevent.spawn(waiter),
		gevent.spawn(waiter),
		gevent.spawn(waiter)
	])

if __name__ == '__main__': main()

</code>
</pre>

У объекта Event есть расширение AsyncResult, которое позволяет передавать данные вместе с пробуждающим поток вызовом. Иногда это называют future (будущее) или deffered (отложенное)  значение, так как оно связано с будущим значение, которое будет установлено в будущем в произвольном порядке.

<pre>
<code class="python">import gevent
from gevent.event import AsyncResult
a = AsyncResult()

def setter():
    """
    After 3 seconds set the result of a.
    """
    gevent.sleep(3)
    a.set('Hello!')

def waiter():
    """
    After 3 seconds the get call will unblock after the setter
    puts a value into the AsyncResult.
    """
    print(a.get())

gevent.joinall([
    gevent.spawn(setter),
    gevent.spawn(waiter),
])

</code>
</pre>

## Очереди

Очередь -- это упорядоченная последовательность данных, которая поддерживает ``put`` / ``get`` операции и реалиована для безопасного совместного использования гринлетами.

К примеру, если несколько гринлетов выполняется одновременно и при этом один из них вытащит элемент из очереди, то этот элемент станетнедоступен другим гринлетам.

[[[cog
import gevent
from gevent.queue import Queue

tasks = Queue()

def worker(n):
    while not tasks.empty():
        task = tasks.get()
        print('Рабочий %s получил задание %s' % (n, task))
        gevent.sleep(0)

    print('Пора заканчивать!')

def boss():
    for i in xrange(1,25):
        tasks.put_nowait(i)

gevent.spawn(boss).join()

gevent.joinall([
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'nancy'),
])
]]]
[[[end]]]

Очереди могут блокировать ``put`` / ``get`` операцию, если это требуется. 

У put и get есть асинхронные аналоги, которые называются соответсвенно ``put_nowait`` и ``get_nowait``. Их выполнение не блокируется очередью, однако в случае невозможности выполнения операции они будут генерировать исключения ``gevent.queue.Empty`` или ``gevent.queue.Full``

Ниже приведён пример, где очередь имеет глубину в 3 элемента. Благодаря этому ``boss`` будет помещать задания в ``tasks`` только до тех пор пока их количество меньше трёх. В противном случае операция put блокируется пока до тех пор, пока в очереди появится место. 
Напротив ``get`` будет блокировать исполнение, если в очереди нет заданий. ``get`` также принимате аргумент ``timeout``, благодаря чему очередь будет выкидывать исключение ``gevent.queue.Empty``, если ``timeout`` истёк, а задач не было.

[[[cog
import gevent
from gevent.queue import Queue, Empty

tasks = Queue(maxsize=3)

def worker(name):
    try:
        while True:
            task = tasks.get(timeout=1) # decrements queue size by 1
            print('Worker %s got task %s' % (name, task))
            gevent.sleep(0)
    except Empty:
        print('Quitting time!')

def boss():
    """
    Boss will wait to hand out work until a individual worker is
    free since the maxsize of the task queue is 3.
    """

    for i in xrange(1,10):
        tasks.put(i)
    print('Assigned all work in iteration 1')

    for i in xrange(10,20):
        tasks.put(i)
    print('Assigned all work in iteration 2')

gevent.joinall([
    gevent.spawn(boss),
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'bob'),
])
]]]
[[[end]]]

## Группы и объединения

Изначально этот раздел назывался "Groups & Pools". Если для слова group в русском языке есть понятный перевод, то для слова pool есть более 17 значений из которых наиболее подходит по смыслу ``объединение``.

Обе структуры: и ``группа`` и ``объединение`` представлют собой способ управления несколькими гринлетами сразу. 

Группа предназначена для управления набором гринлетов как одним целым. С её помощью удобно управлять целым набором асинхронных задач. Пример:

[[[cog
import gevent
from gevent.pool import Group

def talk(msg):
    for i in xrange(3):
        print(msg)

g1 = gevent.spawn(talk, 'bar')
g2 = gevent.spawn(talk, 'foo')
g3 = gevent.spawn(talk, 'fizz')

group = Group()
group.add(g1)
group.add(g2)
group.join()

group.add(g3)
group.join()
]]]
[[[end]]]

Это очень удобнйы инструмент для управления группами асинхронных задач.
Группа также предоставляет инструменты для отправки задач сгруппированным гринлетам и сбора результатов их работы.

[[[cog
import gevent
from gevent import getcurrent
from gevent.pool import Group

group = Group()

def hello_from(n):
    print('Size of group %s' % len(group))
    print('Hello from Greenlet %s' % id(getcurrent()))

group.map(hello_from, xrange(3))


def intensive(n):
    gevent.sleep(3 - n)
    return 'task', n

print('Ordered')

ogroup = Group()
for i in ogroup.imap(intensive, xrange(3)):
    print(i)

print('Unordered')

igroup = Group()
for i in igroup.imap_unordered(intensive, xrange(3)):
    print(i)

]]]
[[[end]]]

``Объединение`` (pool) же представляет собой структуру, которая позволяет ограничить число одновременно выполняющихся гринлетов. Их удобно использовать при обработке сетевых событий или задач ввода/вывода

[[[cog
import gevent
from gevent.pool import Pool

pool = Pool(2)

def hello_from(n):
    print('Size of pool %s' % len(pool))

pool.map(hello_from, xrange(3))
]]]
[[[end]]]

Частым решением при построении сервиса на основе gevent является выбор такой архитектуры, в основе которой лежит использование объединения (pool) как основной структуры данных.
Для примера рассмотрим класс, который опрашивает набор сокетов:

<pre>
<code class="python">from gevent.pool import Pool

class SocketPool(object):

    def __init__(self):
        self.pool = Pool(1000)
        self.pool.start()

    def listen(self, socket):
        while True:
            socket.recv()

    def add_handler(self, socket):
        if self.pool.full():
            raise Exception("At maximum pool size")
        else:
            self.pool.spawn(self.listen, socket)

    def shutdown(self):
        self.pool.kill()

</code>
</pre>

## Блокировки и семафоры

``Семафор`` -- это низкоуровневый примитив синхронизации, который позволяет координировать работу гринлетов, контролировать одновременный доступ или исполнение. Семафор предоставляет два методы: ``acquire``, ``release``. Разница между числом захватов и особождения семафора называется связанностью семафора. Если готовность равна "0", тогда она заблокируется до момента, пока другой гринлет не освободит семафор.

[[[cog
from gevent import sleep
from gevent.pool import Pool
from gevent.coros import BoundedSemaphore

sem = BoundedSemaphore(2)

def worker1(n):
    sem.acquire()
    print('Worker %i acquired semaphore' % n)
    sleep(0)
    sem.release()
    print('Worker %i released semaphore' % n)

def worker2(n):
    with sem:
        print('Worker %i acquired semaphore' % n)
        sleep(0)
    print('Worker %i released semaphore' % n)

pool = Pool()
pool.map(worker1, xrange(0,2))
pool.map(worker2, xrange(3,6))
]]]
[[[end]]]

Семафор, у которого связанность равна "1" называется блокировкой. Блокировка предоставлят одному гринлету эксклюзивное право исполнения. Блокировки используются, чтобы в процессе работы программы, ограничить одновременный доступ к какому-либо ресурсу только для одного гринлета.

## Локальные данные (Thread locals)

``Gevent`` предоставляет возможность определять какие данные будут считать локальными внутри гринлета. 

Это реализовано через глобальный поиск атрибута внутри контекста гринлета, доступ к которому осуществляется по ключу равному значению возвращаемому ``getcurrent()``

[[[cog
import gevent
from gevent.local import local

stash = local()

def f1():
    stash.x = 1
    print(stash.x)

def f2():
    stash.y = 2
    print(stash.y)

    try:
        stash.x
    except AttributeError:
        print("x is not local to f2")

g1 = gevent.spawn(f1)
g2 = gevent.spawn(f2)

gevent.joinall([g1, g2])
]]]
[[[end]]]

Многие фреймворки используют local для хранения объектов сессии HTTP. К примеру, возьмём Werkzeug и воспользуемся его прокси-объектом и создадим оъект запроса в стиле Flask.

<pre>
<code class="python">from gevent.local import local
from werkzeug.local import LocalProxy
from werkzeug.wrappers import Request
from contextlib import contextmanager

from gevent.wsgi import WSGIServer

_requests = local()
request = LocalProxy(lambda: _requests.request)

@contextmanager
def sessionmanager(environ):
    _requests.request = Request(environ)
    yield
    _requests.request = None

def logic():
    return "Hello " + request.remote_addr

def application(environ, start_response):
    status = '200 OK'

    with sessionmanager(environ):
        body = logic()

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    return [body]

WSGIServer(('', 8000), application).serve_forever()


<code>
</pre>

Flask конечно более сложен, чем показано в этом примере, однако идея организации хранения сессии всё та же. 

## Подпроцессы (Subprocess)

В gevent 1.0 появился ``gevent.subprocess`` -- пропатченная версия модуля ``subprocess``, которая поддерживает одновременное исполнение.

<pre>
<code class="python">
import gevent
from gevent.subprocess import Popen, PIPE

def cron():
    while True:
        print("cron")
        gevent.sleep(0.2)

g = gevent.spawn(cron)
sub = Popen(['sleep 1; uname'], stdout=PIPE, shell=True)
out, err = sub.communicate()
g.kill()
print(out.rstrip())
</pre>

<pre>
<code class="python">
cron
cron
cron
cron
cron
Linux
<code>
</pre>

Множество людей хотело использовать ``gevent`` вместе ``subprocess``, однако межпроцессное взаимодействие, которое реализовано на основе модуля ``multiprocssing``, от природы не было предназначено для одновременного исполнения. Но так как объекты на основе ``multiprocessing.Connection`` (например ``Pipe```) предоставляют доступ к своим файловым дескрипторам, то оказалось возможнм использовать ``gevent.socket.wait_read`` и ``gevent.wait_write`` для одновременного ожидания получения событий ``ready-to-read`` / ``ready-to-write`` перед тем, как приступить к реальным чтению или записи.   

<pre>
<code class="python">
import gevent
from multiprocessing import Process, Pipe
from gevent.socket import wait_read, wait_write

# To Process
a, b = Pipe()

# From Process
c, d = Pipe()

def relay():
    for i in xrange(10):
        msg = b.recv()
        c.send(msg + " in " + str(i))

def put_msg():
    for i in xrange(10):
        wait_write(a.fileno())
        a.send('hi')

def get_msg():
    for i in xrange(10):
        wait_read(d.fileno())
        print(d.recv())

if __name__ == '__main__':
    proc = Process(target=relay)
    proc.start()

    g1 = gevent.spawn(get_msg)
    g2 = gevent.spawn(put_msg)
    gevent.joinall([g1, g2], timeout=1)
</code>
</pre>

### Внимание: 

Следует помнить, что использование связки gevent-subprocess вносит в ваш код проблемы, связанные с используемой ОС, среди которых такие: 

* На POSIX-совместимых системах состояние gevent в дочернем    процессе будет некорректным. Одним из побочных эффектов этого является, что гринлет, порожденный перед созданием multiprocessing.Process будет работать и в дочернем и в родительском процессе.

* В примере выше a.send() в put_msg() может блокировать вызывающий поток: событие ready-to-write говорит лишь о том, что один байт может быть записан. При этом буффер может быть переполнится между этим событием и реальной попыткой записи.

* Подход с использованием wait_write() / wait_read() не работает на Windows. (IOError: 3 is not a socket (files are not supported)), так как у Windows нет механизма слежения за событиями в каналах (pipes)

Существует библиотека  [gipc](http://pypi.python.org/pypi/gipc), которая решает большинство таких проблем. Она использует multiprocessing.Process, который учитывает наличие gevent, а также межпроцессное взаимодействие, основанное на каналах. 

## Актёры (Actors)

Это высокоуровневая модель одновременных вычислений, которую популяризирует Erlang. Суть заключаетя в том, что у вас имеется набор независимых актёров, у каждого актера есть условный контейнер для входящих сообщений от других актёров. Основной циклу внутри каждого актёра проверяет регулярно этот контейнер входящих сообщений на наличие новых сообщений и выполняет действия на основе этих сообщений.

У gevent нет примитива Актёр, но его легко создать на основе gevent.Greenlet и gevent.Queue

<pre>
<code class="python">import gevent
from gevent.queue import Queue


class Actor(gevent.Greenlet):

    def __init__(self):
        self.inbox = Queue()
        Greenlet.__init__(self)

    def receive(self, message):
        """
        Define in your subclass.
        """
        raise NotImplemented()

    def _run(self):
        self.running = True

        while self.running:
            message = self.inbox.get()
            self.receive(message)

</code>
</pre>

In a use case:

<pre>
<code class="python">import gevent
from gevent.queue import Queue
from gevent import Greenlet

class Pinger(Actor):
    def receive(self, message):
        print(message)
        pong.inbox.put('ping')
        gevent.sleep(0)

class Ponger(Actor):
    def receive(self, message):
        print(message)
        ping.inbox.put('pong')
        gevent.sleep(0)

ping = Pinger()
pong = Ponger()

ping.start()
pong.start()

ping.inbox.put('start')
gevent.joinall([ping, pong])
</code>
</pre>

# Примеры использования в реальных приложениях

## Gevent ZeroMQ

Авторы [ZeroMQ](http://www.zeromq.org/) заявляют её как библиотеку для работы с сокетами, которая ведет себя как фреймворк одновременного исполнения ("a socket library that acts as a concurrency framework"). Это очень мощный библиотека для работы с сообщениями и построения распределённых и параллельных приложений.

Библиотека предоставляет разные примитивы для работы с сокетами. Самый простой из них -- это пара Request-Response. Сокет имеет два метода: ``send`` & ``recv``. Оба блокирующие. Это поведение устраняет отличная [библиотека](https://github.com/tmc/gevent-zeromq) (теперь часть pyzmq) от Трэвиса Кляйна. В ней используется ``gevent.socket`` для неблокирующего опроса ZeroMQ-сокетов.

[[[cog
# Note: Remember to ``pip install pyzmq``
import gevent
import zmq.green as zmq

# Глобальный контекст
context = zmq.Context()

def server():
    server_socket = context.socket(zmq.REQ)
    server_socket.bind("tcp://127.0.0.1:5000")

    for request in range(1,10):
        server_socket.send("Hello")
        print('Switched to Server for %s' % request)
        # Implicit context switch occurs here
        server_socket.recv()

def client():
    client_socket = context.socket(zmq.REP)
    client_socket.connect("tcp://127.0.0.1:5000")

    for request in range(1,10):

        client_socket.recv()
        print('Switched to Client for %s' % request)
        # Implicit context switch occurs here
        client_socket.send("World")

publisher = gevent.spawn(server)
client    = gevent.spawn(client)

gevent.joinall([publisher, client])

]]]
[[[end]]]

## Простые серверы

<pre>
<code class="python">
# On Unix: Access with ``$ nc 127.0.0.1 5000``
# On Window: Access with ``$ telnet 127.0.0.1 5000``

from gevent.server import StreamServer

def handle(socket, address):
    socket.send("Hello from a telnet!\n")
    for i in range(5):
        socket.send(str(i) + '\n')
    socket.close()

server = StreamServer(('127.0.0.1', 5000), handle)
server.serve_forever()
</code>
</pre>

## WSGI сервер

Из коробки Gevent предоставляет два WSGI-сервера: wsgi, pywsgi

* gevent.wsgi.WSGIServer
* gevent.pywsgi.WSGIServer



До версии "1.0" gevent использовал libevent вместо libev сейчас. В libevent был быстрый    HTTP-сервер, который использовался в wsgi сервере gevent.

## Потоковые сервера (Sreaming servers)

**If you are using gevent 1.0.x, this section does not apply**

For those familiar with streaming HTTP services, the core idea is
that in the headers we do not specify a length of the content. We
instead hold the connection open and flush chunks down the pipe,
prefixing each with a hex digit indicating the length of the
chunk. The stream is closed when a size zero chunk is sent.

    HTTP/1.1 200 OK
    Content-Type: text/plain
    Transfer-Encoding: chunked

    8
    <p>Hello

    9
    World</p>

    0

The above HTTP connection could not be created in wsgi
because streaming is not supported. It would instead have to
buffered.

<pre>
<code class="python">from gevent.wsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'
    body = '&lt;p&gt;Hello World&lt;/p&gt;'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    return [body]

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre>

Using pywsgi we can however write our handler as a generator and
yield the result chunk by chunk.

<pre>
<code class="python">from gevent.pywsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    yield "&lt;p&gt;Hello"
    yield "World&lt;/p&gt;"

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre>

But regardless, performance on Gevent servers is phenomenal
compared to other Python servers. libev is a very vetted technology
and its derivative servers are known to perform well at scale.

To benchmark, try Apache Benchmark ``ab`` or see this
[Benchmark of Python WSGI Servers](http://nichol.as/benchmark-of-python-web-servers)
for comparison with other servers.

<pre>
<code class="shell">$ ab -n 10000 -c 100 http://127.0.0.1:8000/
</code>
</pre>

## Long Polling

<pre>
<code class="python">import gevent
from gevent.queue import Queue, Empty
from gevent.pywsgi import WSGIServer
import simplejson as json

data_source = Queue()

def producer():
    while True:
        data_source.put_nowait('Hello World')
        gevent.sleep(1)

def ajax_endpoint(environ, start_response):
    status = '200 OK'
    headers = [
        ('Content-Type', 'application/json')
    ]

    start_response(status, headers)

    while True:
        try:
            datum = data_source.get(timeout=5)
            yield json.dumps(datum) + '\n'
        except Empty:
            pass


gevent.spawn(producer)

WSGIServer(('', 8000), ajax_endpoint).serve_forever()

</code>
</pre>

## Websockets

Websocket example which requires <a href="https://bitbucket.org/Jeffrey/gevent-websocket/src">gevent-websocket</a>.


<pre>
<code class="python"># Simple gevent-websocket server
import json
import random

from gevent import pywsgi, sleep
from geventwebsocket.handler import WebSocketHandler

class WebSocketApp(object):
    '''Send random data to the websocket'''

    def __call__(self, environ, start_response):
        ws = environ['wsgi.websocket']
        x = 0
        while True:
            data = json.dumps({'x': x, 'y': random.randint(1, 5)})
            ws.send(data)
            x += 1
            sleep(0.5)

server = pywsgi.WSGIServer(("", 10000), WebSocketApp(),
    handler_class=WebSocketHandler)
server.serve_forever()
</code>
</pre>

HTML Page:

    <html>
        <head>
            <title>Minimal websocket application</title>
            <script type="text/javascript" src="jquery.min.js"></script>
            <script type="text/javascript">
            $(function() {
                // Open up a connection to our server
                var ws = new WebSocket("ws://localhost:10000/");

                // What do we do when we get a message?
                ws.onmessage = function(evt) {
                    $("#placeholder").append('<p>' + evt.data + '</p>')
                }
                // Just update our conn_status field with the connection status
                ws.onopen = function(evt) {
                    $('#conn_status').html('<b>Connected</b>');
                }
                ws.onerror = function(evt) {
                    $('#conn_status').html('<b>Error</b>');
                }
                ws.onclose = function(evt) {
                    $('#conn_status').html('<b>Closed</b>');
                }
            });
        </script>
        </head>
        <body>
            <h1>WebSocket Example</h1>
            <div id="conn_status">Not Connected</div>
            <div id="placeholder" style="width:600px;height:300px;"></div>
        </body>
    </html>


## Chat Server

The final motivating example, a realtime chat room. This example
requires <a href="http://flask.pocoo.org/">Flask</a> ( but not necessarily so, you could use Django,
Pyramid, etc ). The corresponding Javascript and HTML files can
be found <a href="https://github.com/sdiehl/minichat">here</a>.


<pre>
<code class="python"># Micro gevent chatroom.
# ----------------------

from flask import Flask, render_template, request

from gevent import queue
from gevent.pywsgi import WSGIServer

import simplejson as json

app = Flask(__name__)
app.debug = True

rooms = {
    'topic1': Room(),
    'topic2': Room(),
}

users = {}

class Room(object):

    def __init__(self):
        self.users = set()
        self.messages = []

    def backlog(self, size=25):
        return self.messages[-size:]

    def subscribe(self, user):
        self.users.add(user)

    def add(self, message):
        for user in self.users:
            print(user)
            user.queue.put_nowait(message)
        self.messages.append(message)

class User(object):

    def __init__(self):
        self.queue = queue.Queue()

@app.route('/')
def choose_name():
    return render_template('choose.html')

@app.route('/&lt;uid&gt;')
def main(uid):
    return render_template('main.html',
        uid=uid,
        rooms=rooms.keys()
    )

@app.route('/&lt;room&gt;/&lt;uid&gt;')
def join(room, uid):
    user = users.get(uid, None)

    if not user:
        users[uid] = user = User()

    active_room = rooms[room]
    active_room.subscribe(user)
    print('subscribe %s %s' % (active_room, user))

    messages = active_room.backlog()

    return render_template('room.html',
        room=room, uid=uid, messages=messages)

@app.route("/put/&lt;room&gt;/&lt;uid&gt;", methods=["POST"])
def put(room, uid):
    user = users[uid]
    room = rooms[room]

    message = request.form['message']
    room.add(':'.join([uid, message]))

    return ''

@app.route("/poll/&lt;uid&gt;", methods=["POST"])
def poll(uid):
    try:
        msg = users[uid].queue.get(timeout=10)
    except queue.Empty:
        msg = []
    return json.dumps(msg)

if __name__ == "__main__":
    http = WSGIServer(('', 5000), app)
    http.serve_forever()
</code>
</pre>
