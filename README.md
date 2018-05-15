Данная программа написана на языке Java. В качестве среды разработки Maven-проекта  используется IntelliJ IDEA. 
Используются следующие инструменты: 
* JGraphX – mxgraph (https://github.com/jgraph/jgraphx/) – построение графа
* OpenCSV  - запись списка смежности в .csv формат
* SAX parser – инструмент, позволяющий как можно быстрее парсить большой xml документ
- xml документ нужно обязательно(!) скачать [отсюда](https://yadi.sk/d/S6v3w-4D3TNhjM) и вставить в папку проекта(в корневую). Иначе, работать ничего не будет.

# <br>Часть 1  
<br>Программа представляет собой два класса: Graph и LogicClass. Последний отвечает за логику чтения данных, составление списка смежности, правильный отбор данных для построения вершин, а Graph – за визуализацию графа.
Алгоритм работы данной программы:  
#### 1. Разбираем xml-файл “map2.xml”.
В моем случае это город Ставрополь.  
-с помощью класса startElement() мы находим в нашем документе первый тег <way>, который указывает на дорогу. Здесь мы фиксируем id дороги (way_id) для дальнейшего использования в ассоциативном массиве HashMap.
-далее мы обращаемся ко всем тегам <nd>, в который определяем значение атрибута ref как id точки. То есть, одной дороге соответствует множество точек. Здесь же мы сразу определяем точку начала дороги – start_point и точку конца дороги – last. 
-Как только мы прочитали точку, заносим ее в коллекцию nodes, где ключом является id точки, а значением – id пути. То есть, разным ключам-точкам - ставится в соответствие один путь. Если же мы уже имели дело с такой точкой (ключ в коллекции есть) , у нас рассматривается случай перекрестка дорог: если в коллекцию crosses эта точка не занесена, заносим, иначе добавляем id нового пути к списку предыдущих путей. То есть, в crosses одной точке – точке пересечения путей – ставится в соответствие множество путей. Они разделяются знаком «=».
-Далее происходит анализ дороги на однонаправленность, двунаправленность, замкнутость (начальная точка соответствует конечной  = кольцевая дорога). Мы обращаемся к тегу <tag> , берем оттуда атрибут k=”oneway” – указывает на то, что определяется тип дороги, а атрибут v=”yes/no/-1” – на значение. yes-односторонняя, no – двустороняя, -1 – обратная(стартовое nd = конечное nd). Если односторонняя – заносим значения начала и конца дороги в карты start и end_ , где ключом является id дороги, а значением – id точки. Если двусторонняя – в карту both кладем id пути и в качестве значения – начальная и конечная точки. Если же наша дорога в обратном направлении (-1) , то стартовое id меняем на id последней точки и наоборот.
Если не существует атрибута с таким значением, то дорога является двусторонней(ну так поняла я).
-C помощью метода endElement() мы следим за тем, чтобы тег <tag> брался только из тега <way>, а также формируем карту замкнутого пути (last.equals(start_point)).
Таким образом, мы формируем 5 карт: crosses(перекрестки), start(начальные точки),end_(конечные точки), both_way(двунаправленный путь), clothed(замкнутый путь).
По времени это происходит в течение 11-12 секунд.
#### 2. Составляем список смежности  
-За это отвечают методы get(карта1,карта2) и search(карта1,карта2), где за «карту1» мы берем ту карту, каждой ключ со значением которой мы перебираем, «карта2» - карта, с которой мы работаем для поиска смежности.
get() – поиск смежных вершин
search() – добавление несмежных вершин к списку смежных.  
Логика заключается в следующем. Входящие данные: карта crosses(перекрестки), с которой сравниваются данные карт start, end_ и т.д. Исходящие данные: карта “adjacency”, в которой ключом является id начальной точки пути, а значением – список всех перекрестков. То есть одна дорога может проходить через несколько перекрестков(значение=id точки перекрестка1;id точки перекрестка 2;…id точки перекрестка n),  а может и не проходить (тогда мы начальную точку соединяем с конечной).
метод get() – находим id путей в других списках точек и соединяем.  
	1. Берем каждое значение карты crosses и разбиваем на несколько id для дальнейшего поиска:  map1.get(pair.getKey())).split("=");
-Учитываем случай, когда та карта, с которой мы сравниваем перекрестки – both_way. Там у нас указывается начальная и конечная точка как начальная<->конечная. Нужно разбить по значку “<->” и правильно занести каждое id точки в массив ray. Про этот массив ниже.
	2. Перебираем полученный список s и заполняем массив ray(чем?) id вершин рассматриваемых путей (по ключу= id пути).
	3. Далее, если такая вершина найдена (она может находиться в одном из всех исходных карт), формируем новую карту :
    adjacency.put(ray[i], pair.getKey().toString());  
	здесь ключ – id вершины, значение – id точки пересечения путей. Аналогично карте crosses в данной карте в качестве значения может выступать список перекрестков. Один перекресток отделяется от другого с помощью символа «;».  
Таким образом, 
s[i] – id пути
ray[i] – id точки начала пути(это может быть как концом дороги, так и началом, так и концом и началом сразу).
adjacency – карта смежности, которая каждой точке ставит в соответствие набор точек перекрестков.
Перебор ключей карты crosses происходит с помощью итерации.  
В методе search() мы проверяем, все ли точки были рассмотрены(а именно все ли id точек присутствуют в нашей карте adjacency). Если нет, то дополняем ее id = id точки и значением – конца данного пути. Имеется ввиду, что дорога начальную точку соединяет с конечной и не имеет пересечений с другими дорогами.
3. Переносим данные из нашей карты смежности в файл типа .csv в виде id вершины = id смежной вершины 2, id смежной вершины 2,…, id смежной вершины n.
4. Построение графа  
Суть заключается в том, что мы берем каждую вершину по id из карты adjacency и соединяем ее ребрами со смежными вершинами. Тут важна проверка на то, чтобы одна и та же вершина не строилась несколько раз. В связи с этим мы создаем вспомогательный список search, в который заносим рисуемые вершины. Если такая вершина есть в карте, то во второй раз ее заносить уже не надо. 
Строятся вершины с помощью и метода insertVertex(), а дуги – insertEdge().
Граф строится по данным о расположении точек = координатам. С помощью класса View и соответствующих методов (mercX,mercY) мы переводим сферические координаты в евклидовы 
Файл списка смежности – data.csv

# <br>Часть 2  
### 1.Начало работы.
Реализован интерфейс для ввода координат местоположения клиента (передаем введенные x и y в метод поиска кратчайшего пути – search_way(x,y) ), а также создана обработка нажатия на конкретную точку на самом графе для определения местоположения клиента (mouseClicked: getX,getY + search_way(x,y))
### 2.Выбор 10 ресторанов
В классе GetRestaurants мы работаем с ресторанами. Занесены 10 штук, которые берутся из всех  возможных ресторанов (карта objects). В начале работы я брала случайные 10 ресторанов, но потом решила учесть то, что будет лучше, если данные рестораны расположены в теге, чтобы связать ресторан с ближайшей дорогой.
В карту ten_object в качестве ключа добавляется id точки расположения ресторана, а значение - id пути (метод get)
Затем для каждого ресторана мы ищем id пути в карте nodes с целью определения начальной точки пути. Добавляем в карту смежности adjacency ключ value = id точки, что близка к ресторану, значениие = начальная точка пути. Это далее помогает нам в поиске пути до всех ресторанов от расположения клиента
### 3.Определение кратчайшего пути и отображение.
Определяются кратчайшие расстояния сначала до всех ресторанов с помощью алгоритма Дейкстры(LogicClass.Dijkstra()), Левита(LogicClass.Levita()), A*(LogicClass.A2)  
  
### Алгоритм Дейкстры:  
Данный алгоритм был реализован с использованием очереди LinkedHashSet. Данный класс был выбран потому, что добавление и удаление элемента имеет сложность O(1). За счет этого алгоритм работает быстро.  В среднем, время работы данного алгоритма – 8.07 секунд.
  
### Алгоритм Левита:
Здесь использовалось две очереди:
1.Очередь класса ArrayDeque – главная очередь (добавление элемента O(1), удаление элемента – O(n))
2.Очередь класса  LinkedHashSet – срочная очередь(добавление и удаление элемента – O(1)) 
в связи с этим он работает чуть дольше алгоритма Дейкстры, хотя должно быть наоборот 
В среднем, время работы данного алгоритма – 9,30 секунд
  
### Алгоритм А*: 
Используются следующие очереди:
Очередь класса HashSet – прошедшие рассмотрение вершины ( O(1))
Очередь класса ArrayDeque – ожидающие рассмотрения вершины(добавление элемента O(1), удаление элемента – O(n))
Долгое выполнение алгоритма А* обусловлено тем, что он работает с каждой точкой отдельно (один цикл на одну точку), то есть происходят лишние затраты времени. Время выполнения алгоритма  - 11,2 секунды
  
Метод get_way() возвращает кратчайший путь для каждого из алгоритмов. 
near_restaurant – ближайший ресторан. Данные о точках, составляющих путь, определяется с помощью массива предков - p для каждого алгоритма.
  
## 5.Тестирование работы алгоритмов на 100 произвольных точках на карте.  
Производится оно с помощью метода test(random_points). Точки в рандомном порядке выбираются из карты adjacency, затем к ним применяются алгоритмы, запускается отсчет времени выполнения метода для каждой точки, а затем считается среднее время для каждого алгоритма и выбирается тот, для которого время -> min.  
Результатом оказалось то, что алгоритм Дейкстры работает быстрее.  
#### Результаты:  
1.Алгоритм Дейкстры  
Среднее выполнения алгоритма = 2152 мс=2,15 секунд на одну точку
  
2.Алгоритм Левита  
Среднее выполнения алгоритма =3,04 секунды  на одну точку
  
3.Алгоритм A*  
Среднее выполнения алгоритма =  5.18 секунд на одну точку
Время, затрачиваемое на путь, считается с помощью метода get_time(double distance). Здесь при расчете расстояния широта * 111.3, а долгота - 62.25. Таким образом мы получаем расстояние в километрах. Затем делаем преобразования и получаем время в минутах. Средним временем при тестировании получилось 6 минут 7 секунд.
  
## 6.data6.csv -  файл содержащий кратчайшие пути до 10 точек (клиентов);
## 7.Визуализация  
в папке [visualization](https://github.com/Snezzz/Graph_project/tree/master/visualization) находятся скриншоты полученных кратчайших путей до 10 точек

# <br>Часть 3  
Главный метод - search_way_from()):  
1.Определяем точку расположения склада, которую задал пользователь (клик или в меню) и отрисовывем ее.  
2.Обращаемся к алгоритму и находим кратчайшие пути (заполнение карты main_way)  
3.Перебираем все пути из нашей карты и отрисовываем их  
Реализован алгоритм поиска ближайшего соседа (neighbor_algorithm())  
В качестве входных данных передаем точку расположения склада = начальная точка.  
Карта main_way  ( ключ типа «от -> до» , значение – множество точек кратчайшего пути) предназначена для восстановления всех кратчайших путей.  
Карта visited_point (ключ- id ресторана, значение – true/false посещен/не посещен).  
Карта distance – матрица расстояний между ресторанами  
Здесь все основано на работе с очередью LinkedHashSet(), в которую мы добавляем рассматриваемый нами ресторан.  
Пока мы не обошли все десять ресторанов, для точки current (текущая) ищем ближайшего соседа из всех возможных ресторанов. Делается это с помощью использования алгоритма по поиску кратчайшего пути – алгоритма Дейкстры. Итак, мы определили ближайший из всех ресторанов – “to”, занесли данный путь в карту main_way и пометили его как посещенный. Далее мы ищем кратчайшее расстояние до тех точек, которые остались как непосещенные.  
В конце концов мы получаем заполненную карту main_way с размером 10 (так как ресторанов у нас 10 штук по условию). Здесь же формируется data7.csv файл всех кратчайших путей.  
Был также реализован алгоритм ветвей и границ, но данная работа не доделана  
в папке [visualization](https://github.com/Snezzz/Graph_project/tree/master/visualization) находятся скриншоты полученного объезда (файлы 4.jpg и 5.jpg)
