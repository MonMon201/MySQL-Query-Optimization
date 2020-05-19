# Оптимізація запитів в MySQL

    У цій статті пояснено, як оптимізувати деякі запити MySQL, з навединими прикладами. Оптимізація включає в себе в першу чергу конфігурування, налаштуваня та вимірювання продуктивності. Оптимізація потрібна для того, щоб використовувати ресурси процесора та пам'яті максимально ефективно, покращувати масштабованість бази даних, дозволяючи, їй, також, обробляти більше навантаження з меншими затримками.
<br>
    Продуктивність бази даних залежить від кількох факторів, на рівні бази даних, її таблиць, запитів, та налаштувань конфігурації. Ці конструкції приводять у рух операції процесора на апаратному рівні, і ви маєте зробити ці операції максимально ефективними. Отже, для того, щоб максимально ефективно використовувати ресурси процесору та часу, потрібно для початку оптимізувати запити. 

## 1. Оптимізація запитів з WHERE

    У цьому розділі розглядаються оптимізації, які можна зробити для обробки запитів з WHERE. У прикладах використовуються оператори SELECT, але ті ж самі оптимізації застосовуються у операторах DELETE та UPDATE.

    Можливо, вам подобається писати запити, таким чином, щоб зробити арифметичні операції швидшими, при цьому приносячи в жертву читабельність. Але, оскільки MySQL робить подібні оптимізації автоматично, ви можете уникати цієї роботи та залишати запит у більш зрозумілій формі. 

### 1.1 Відкидання непотрібних дужок для виразів



**Неоптимізований запит**

```SQL
. . .
select * from geodata._cities where (((city_id < 1000000) and (country_id > 100)) or ((city_id <
1000000) and (country_id < 100)));
. . .
```

**Оптимізований запит**

```SQL
. . .
select * from geodata._cities where city_id < 1000000 and country_id > 100 or city_id < 1000000 and country_id < 100;
. . .
```
**висновок**
    Краще писати зрозуміліше та прибрати непотрібні дужки, що роблять запит громіздким.

### 1.2 Згортання непотрібних констант

**Неоптимізований запит**

```SQL
. . .
select * from geodata._cities where city_id < region_id and region_id = '3767477' and city_id = '3767455'
; -- Поганий вид запиту
. . .
```

**Оптимізований запит**

```SQL
. . .
select * from geodata._cities where '3767455' < region_id and region_id = '3767477' and city_id = '3767455'
; -- Оптимальний вид запиту
. . .
```

**висновок**
Якщо є можливість уникнути додаткової умови для константи і значення виразу відоме, то це значення можна зразу записати в необхідну умову.

### 1.3 Відкидання умов

**Неоптимізований запит**

```SQL
. . .
select * from geodata._cities where city_id < region_id and region_id = '3767477' and city_id = '3767455'
; -- Поганий вид запиту
. . .
```

**Оптимізований запит**

```SQL
. . .
select * from geodata._cities where region_id = '3767477' and city_id = '3767455'
; -- Оптимальний вид запиту
. . .
```

**висновок**
Іноді краще переписати умову запиту, знаючи необхідні дані, аніж доповнювати запит новими підумовами, нагромаджуючи запит.

## 2. Оптимізація запитів з діапазоном

### 2.1 Оптимізація мало- та багатозначних конструкцій

**Неоптимізований запит**

```SQL
. . .
select * from geodata._cities where city_id in ('3772513', '3772277');
. . .
```

**вивід EXPLAIN**
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | 
|  1 | SIMPLE      | _cities | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1851220 |    20.00 | Using where |


**Оптимізований запит**

```SQL
. . .
select * from geodata._cities where city_id = '3772513' or city_id = '3772277';
. . .
```

**вивід EXPLAIN**
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  1 | SIMPLE      | _cities | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1851220 |    19.00 | Using where |


**висновок**
Якщо для запиту портібно декілька (небагато) значень для перевірки індекса, то краже використовувати логічне додавання (перебір) значень за допомогою OR, ніж пошук в колекції з IN оператором, так як відсоток фільтрації значень в першому випадку менший, що свідчить про вищу якіть фільтрації значень. 

### 2.1 Оптимізація конструкторів ряду

**Неоптимізований запит**

```SQL
. . .
select * from geodata._cities where (city_id, country_id) in ( ('3772493', '119'), ('5418924', '200') );. . .
```

| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  1 | SIMPLE      | _cities | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1851220 |     2.00 | Using where |


**Оптимізований запит**

```SQL
. . .
select * from gselect * from geodata._cities where (city_id = '3772493' and country_id = '119') or (city_id = '5418924' and country_id = '200');eodata._cities where city_id = '3772513' or city_id = '3772277';
. . .
```

**вивід EXPLAIN**
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  1 | SIMPLE      | _cities | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1851220 |     1.99 | Using where |


**висновок**
Оптимізація працює аналогічно попередньому прикладу. 

## 3. Оптимізація запитів з GROUP BY

### 3.1 Loose Index Scan
Використовується при вибірці, під час виконання якої значення одного з індексів може залишатись незмінним для певної кількості значень, що дозволяє згрупувати такі значення і використовувати індекс, визначений в першому значенні з цієї групи.

**Реалізація методу наявна в подібних запитах:**
```SQL
select distinct country_id, region_ru from geodata._cities;
select country_id, region_id, min(city_id) from geodata._cities group by country_id, region_id;
```

**Запити, в яких (та подібих їм) даний метод не може бути реалізованим:**
```SQL
select count(distinct country_id, region_id), count(distinct city_id, region_id) from _cities;
select country_id, region_id, count(*) from geodata._cities group by country_id, region_id;
```

### 3.2 Tight Index Scan
На відміну від Loose Index Scan, вибирає підходящі за умовою значення, а вже потім іх групує. Тобто, всі подібні приклади, що підходять до попередьного індексу, не підходять до цього.

**Реалізація методу наявна в подібних запитах:**
```SQL
select country_id, region_id, city_id from geodata._cities where region_id = '4024696' group by city_id, country_id;
```

## 4. Індекси в MySQL

### 4.1 Класифікація Індексів

Індекси бувають **кластерні** та **некластерні**.

У чому різниця? 

Некластерний індекс зберігає тільки посилання на записи таблиці. Коли відбувається робота з індексом, визначається тільки список записів (точніше список їх первинних ключів), що підходять під запит. Після цього відбувається ще один запит - для отримання даних кожного запису з цього списку.

Кластерні індекси зберігають дані записів цілком, а не посилання на них. При роботі з таким індексом не потрібно додаткової операції читання даних.

Некластерні в свою чергу поділяються на декілька видів.

**Найпростіший** з них, приклад якого ми вже бачили в попередніх слайдах, створюється для тих колонок, які присутні в умові WHERE та в умові яких є ORDER BY.

Також існують **унікальні індекси** для колонок, значення в яких повинні бути унікальними по всій таблиці. Такі індекси покращують ефективність вибірки для унікальних значень.

Ще є **складені індекси**, для запитів, в яких використовується кілька колонок, бо MySQL може використовувати тільки один індекс для запиту (крім випадків, коли MySQL здатний об'єднати результати вибірок за кількома індексами). Вони використовуються для пошуку по діапазону та сортування.

## 4.1 Найпростіші Індекси

### Без Індексу

**Запит без індексу**

```SQL
. . .
select * from geodata._cities where region_id = '4024696';
. . .
```

**вивід EXPLAIN**

| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  1 | SIMPLE      | _cities | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1851220 |    10.00 | Using where |

### Створення Індексу

```SQL
. . .
create index idx_region on _cities(region_id);
. . .
```

### Використання Індексу

**вивід EXPLAIN**

| id | select_type | table   | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  1 | SIMPLE      | _cities | NULL       | ref  | idx_region    | idx_region | 5       | const |  625 |   100.00 | NULL  |

## 4.2 Унікальні Індекси

### Без Індексу

**Запит без індексу**

```SQL
. . .
select * from geodata._cities where city_id = ''4027457';
. . .
```

**вивід EXPLAIN**
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  1 | SIMPLE      | _cities | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1851220 |    10.00 | Using where |

### Створення Індексу

```SQL
. . .
create index idx_city on _cities(city_id);
. . .
```

### Використання Індексу

**вивід EXPLAIN**

| id | select_type | table   | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  1 | SIMPLE      | _cities | NULL       | ref  | idx_city      | idx_city | 4       | const |    1 |   100.00 | NULL  |

## 4.3 Складені Індекси

### Без Індексу

**Запит без індексу**

```SQL
. . .
select * from geodata._cities where country_id = '176' and region_id = '4024696';
. . .
```

**вивід EXPLAIN**

| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  1 | SIMPLE      | _cities | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1851220 |     1.00 | Using where |

### Створення Індексу

```SQL
. . .
create index idx_country on _cities(country_id);
create index idx_region on _cities(region_id);
. . .
```

### Використання Індексу

**вивід EXPLAIN**

| id | select_type | table   | partitions | type        | possible_keys          | key                    | key_len | ref  | rows | filtered | Extra |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  1 | SIMPLE      | _cities | NULL       | index_merge | idx_country,idx_region | idx_region,idx_country | 5,4     | NULL |    3 |   100.00 | Using intersect(idx_region,idx_country); Using where |

### Висновок

З найпростішими індексами можна прискорити роботу . . .

## . . . Індекс

. . .

## 5. Висновок

. . . Більше про оптимізацію ви зможете дізнатися в [офіційній документації MySQL](https://dev.mysql.com/doc/refman/8.0/en/optimization.html "MySQL Optimization Documentaion").

## 6. Автори

*  [Василиненко Даніїл](http://example.com/link "github")

*  [Головко Андрій](http://example.com/link "github")

## 7. Посилання

*  [Документація про оптимізацію в MySQL](https://dev.mysql.com/doc/refman/8.0/en/optimization.html "MySQL Optimization Documentaion")
