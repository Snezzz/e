Данная программа написана на языке Java. В качестве среды разработки Maven-проекта  используется IntelliJ IDEA. 
Используются следующие инструменты: 
* JGraphX – mxgraph (https://github.com/jgraph/jgraphx/) – построение графа
* OpenCSV  - запись списка смежности в .csv формат
* SAX parser – инструмент, позволяющий как можно быстрее парсить большой xml документ
- xml документ нужно обязательно(!) скачать [отсюда](https://yadi.sk/d/S6v3w-4D3TNhjM) и вставить в папку проекта(в корневую). Иначе, работать ничего не будет.

<br>Программа представляет собой два класса: Graph и LogicClass. Последний отвечает за логику чтения данных, составление списка смежности, правильный отбор данных для построения вершин, а Graph – за визуализацию графа.
Алгоритм работы данной программы:
1. Разбираем xml-файл “map2.xml”. В моем случае это город Ульяновск.  
-с помощью класса startElement() мы находим в нашем документе первый тег <way>, который указывает на дорогу. Здесь мы фиксируем id дороги (way_id) для дальнейшего использования в ассоциативном массиве HashMap.
-далее мы обращаемся ко всем тегам <nd>, в который определяем значение атрибута ref как id точки. То есть, одной дороге соответствует множество точек. Здесь же мы сразу определяем точку начала дороги – start_point и точку конца дороги – last. 
-Как только мы прочитали точку, заносим ее в коллекцию nodes, где ключом является id точки, а значением – id пути. То есть, разным ключам-точкам - ставится в соответствие один путь. Если же мы уже имели дело с такой точкой (ключ в коллекции есть) , у нас рассматривается случай перекрестка дорог: если в коллекцию crosses эта точка не занесена, заносим, иначе добавляем id нового пути к списку предыдущих путей. То есть, в crosses одной точке – точке пересечения путей – ставится в соответствие множество путей. Они разделяются знаком «=».
-Далее происходит анализ дороги на однонаправленность, двунаправленность, замкнутость (начальная точка соответствует конечной  = кольцевая дорога). Мы обращаемся к тегу <tag> , берем оттуда атрибут k=”oneway” – указывает на то, что определяется тип дороги, а атрибут v=”yes/no/-1” – на значение. yes-односторонняя, no – двустороняя, -1 – обратная(стартовое nd = конечное nd). Если односторонняя – заносим значения начала и конца дороги в карты start и end_ , где ключом является id дороги, а значением – id точки. Если двусторонняя – в карту both кладем id пути и в качестве значения – начальная и конечная точки. Если же наша дорога в обратном направлении (-1) , то стартовое id меняем на id последней точки и наоборот.
Если не существует атрибута с таким значением, то дорога является двусторонней(ну так поняла я).
-C помощью метода endElement() мы следим за тем, чтобы тег <tag> брался только из тега <way>, а также формируем карту замкнутого пути (last.equals(start_point)).
Таким образом, мы формируем 5 карт: crosses(перекрестки), start(начальные точки),end_(конечные точки), both_way(двунаправленный путь), clothed(замкнутый путь).
По времени это происходит в течение 11-12 секунд.
2. Составляем список смежности  
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
Граф получился не очень красивым, но там есть возможность нажать на вершину и увидеть ребро, которое выходит из нее, и к какой вершине ведет.  
Файл списка смежности – data.csv
