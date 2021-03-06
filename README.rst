Импорт данных из ФИАС в Postgresql
----------------------------------

**[!]** Скрипты импорта обновлены 9 января 2019, спасибо *xlazex*. 

Старая версия: https://github.com/asyncee/fias2pgsql/tree/54567d2b6bf618d734b64749b3b2930d2f9caf39

В качестве документации использовались данные статьи:
http://wiki.gis-lab.info/w/%D0%A4%D0%98%D0%90%D0%A1

Исходные скрипты были взяты из gist: https://gist.github.com/wiz/4244207


Все манипуляции проводились на ``postgresql`` версии 10, однако всё должно
работать и на младших версиях 9.x.

Краткая суть импорта данных заключается в следующем:


1. Импорт данных из ``*.dbf`` файлов в postgresql используя утилиту ``pgdbf 0.6.2``.
2. Приведение полученной схемы данных в надлежащее состояние, в т.ч. изменение типа данных колонок.
3. Удаление исторических данных (неактуальных адресных объектов) - опционально, см. пункт 6 секции "Использование"
4. Создание индексов


В скрипте импортируются только следующие таблицы:

- ``addrobj`` - Классификатор адресообразующих элементов (край > область >
  город > район > улица)
- ``socrbase`` - Типы адресных объектов (условные сокращения и уровни
  подчинения)

Если необходимо импортировать и другие данные, то это можно сделать с помощью
правки скрипта ``import.sh``.


Использование
-------------

1. Скачать rar-архив с DBF-файлами сайта ФИАС
2. Распаковать его в директорию `_data`
3. ВМЕСТО ПУНКТОВ 4,6,7 (если не требуется пункт 5) можно выполнить::

    bash index.sh <DB>

4. Создать бд и провести начальный импорт данных::

    bash import.sh <DB>

5. Если нужно, изменить настройки обновления схемы данных в schema.json и
   выполнить скрипт ``update_schema.py``. Это создаст обновлённый файл
   ``update_schema.sql``.

6. Обновить схему данных::

    psql -f update_schema.sql -d <DB>

7. Удалить неактуальные данные и создать индексы::

    psql -f indexes.sql -d <DB>

8. Проверить скорость выполнения следующих запросов::

    -- вывести полный адрес
    WITH RECURSIVE child_to_parents AS (
    SELECT addrobj.* FROM addrobj
        WHERE aoid = '51f21baa-c804-4737-9d5f-9da7a3bb1598'
    UNION ALL
    SELECT addrobj.* FROM addrobj, child_to_parents
        WHERE addrobj.aoguid = child_to_parents.parentguid
            AND addrobj.currstatus = 0
    )
    SELECT * FROM child_to_parents ORDER BY aolevel;

    -- поиск по части адреса
    SELECT * FROM addrobj WHERE formalname ILIKE '%Ульян%';
