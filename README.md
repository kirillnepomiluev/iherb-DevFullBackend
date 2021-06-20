## iHerb -- backend

Серверная часть решения от команды 1DefFull состоит из трёх частей:

1. Нейронная сеть для предупреждения о риске выбранных добавок.
2. Конфигурация базы данных Firebase для поиска по первым буквам болезней.
3. Конвертор справочника болезней в вид, пригодный для загрузки в Firebase. 

### Нейронная сеть предупреждения о риске

Находится в `/python-safety`. Используется Python 3.8 и TensorFlow.

Есть представляет собой последовательную сеть и трёх слоёв.
На вход подаём:
- Массив фич пользователя: принадлежности к классам пищевых привычек, пол, возраст, рост, масса, подтверждённые врачом диагнозы и результаты анализов.
- Дозировки каждого вещества в одном товаре или в сумме по нескольким.

На выходе получаем один скаляр, который предупредает о риске. Отутствие риска нельзя воспринимать как рекомендацию, потому что заключение о пользе может дать только врач.

Установка на Ubuntu:
```bash
apt install python3-pip
pip3 install tensorflow pyyaml h5py virtualenv gunicorn flask
```

- Входные данные для обучения: `data.tsv`
- Обучение: `python3 fit.py`
- Запуск сервера: `python3 serve.py`

После запуска сервер доступен на порте `5000` и принимает POST-запросы:

```
POST / HTTP/1.1

{"features":[1, 0],"substances":[1, 1, 0]}
```
Здесь:
- `features` -- массив фич пользователя: принадлежности к классам пищевых привычек, пол, возраст, рост, масса, подтверждённые врачом диагнозы и результаты анализов.
- `substances` -- дозировки каждого вещества в одном товаре или в сумме по нескольким.

Временная версия доступна здесь:
http://84.252.141.221:5000

Сервер развёрнут на виртуальной машине от Яндекса:
- Intel Cascade Lake, 2 vCPU.
- 2 GB RAM.

### Триггер и индекс Firebase для справочника болезней

Находятся в `/firebase`

Триггер используется, чтобы при добавлении или изменении записи проиндексировать её подстроки -- чтобы работал поиск по частичному вводу.

Деплой:

```bash
firebase deploy --only functions
```

Индексы в `firestore.indexes.json` необходимы для поиска. Деплой:
```bash
firebase deploy --only firestore:indexes
```


### Конвертор справочника болезней

Находится в `/conditions-to-json`. Для загрузки в Firebase записи должны быть в JSON, но оригинал справочника доступен в интернете в CSV. Данный скрипт конвертирует данные.

Запуск:
```bash
python3 conditions-to-json.py --tsv FILE [--limit N] > conditions.json
```

На выходе программа выдаёт JSON, который можно загрузить в Firebase многими инсрументами.

Для загрузки в Firebase нужно установить пакет:
```bash
npm install -g node-firestore-import-export
```

Далее получить файл с ключом, как описано здесь:
https://www.youtube.com/watch?v=gPzs6t3tQak

И выполнить:
```bash
firestore-import --accountCredentials credentials.json --backupFile conditions.json --nodePath conditions
```

где `credentials.json` -- полученный файл с клочом. 
