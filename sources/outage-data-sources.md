# Источники данных об отключениях электроэнергии в Украине

Обзор составлен 2026-10-22. Пометка «проверено» означает, что эндпоинт ответил на запрос в этот день.

## Главный вывод

- Единого официального государственного API с графиками отключений нет. Наборов данных на data.gov.ua с графиками ГПВ найти не удалось.
- Укренерго определяет только количество очередей, которые применяются в регионе. Конкретное расписание по группам публикует каждый оператор системы распределения (ОСР, облэнерго) у себя.
- Из официальных источников машиночитаемый JSON без авторизации дают только **Yasno** (Киев, Днепр) и **Львовобленерго**. Остальные облэнерго публикуют графики в HTML, картинками или через закрытые AJAX-эндпоинты.
- На практике большинство ботов и интеграций берут данные из **JSON-зеркал сообщества на GitHub**: это парсеры официальных сайтов, которые обновляются каждые несколько минут.

## 1. Официальные источники

### Yasno: Киев (ДТЕК КЕМ), Днепр (ДнЕМ, ЦЕК). Публичный API, проверено

База: `https://app.yasno.ua/api/blackout-service/public/shutdowns`. Авторизация не нужна, документации нет: API используется фронтендом Yasno.

| Метод | Путь |
|---|---|
| Регионы и ОСР | `GET /addresses/v2/regions` |
| График на сегодня и завтра | `GET /regions/{regionId}/dsos/{dsoId}/planned-outages` |
| Вероятный недельный график | `GET /probable-outages?regionId={regionId}&dsoId={dsoId}` |

Известные ID: Киев `regionId=25`, `dsoId=902`. Днепр `regionId=3`, `dsoId=301` (ДнЕМ) и `dsoId=303` (ЦЕК).

Адресный поиск (проверено, тоже без авторизации):

| Метод | Путь (от `/shutdowns`) |
|---|---|
| Улицы по подстроке | `GET /addresses/v2/streets?regionId=25&dsoId=902&query=Хрещ` → `[{id, value}]` |
| Дома улицы | `GET /addresses/v2/houses?regionId=25&dsoId=902&streetId=946&query=` → `[{id, value}]` |
| Группа дома | `GET /addresses/v2/group?regionId=25&dsoId=902&streetId=946&houseId=41151` → `{"group":20,"subgroup":1}` |

**Важно:** в Киеве сейчас **60 групп** (`1.1` … `60.1`), а не 6×2. Графики фактически назначаются по домам, поэтому без адреса подписаться на «очередь» не получится.

Формат planned-outages:
- объект с ключами-группами `"1.1"` … `"6.2"`;
- в каждой группе есть `today` и `tomorrow`, у каждого `slots[]` вида `{start, end, type}`;
- `start` и `end` — минуты от 0 до 1440, `type` принимает значения `Definite` или `NotPlanned`;
- `date` — дата дня, `status` — `ScheduleApplies`, `WaitingForSchedule` или `EmergencyShutdowns`;
- `updatedOn` — время последнего изменения графика.

### Львовобленерго: poweron.loe.lviv.ua. Публичный JSON, проверено

- API: `https://api.loe.lviv.ua/api/menus?page=1&type=photo-grafic`. Это API Platform (hydra), параметр `page` обязателен.
- В ответе пункты меню «Today», «Tomorrow» и архив. Внутри PNG-картинка графика (`imageUrl`) и текстовое описание групп (`rawHtml`). Текст нужно парсить.
- Сайт: https://poweron.loe.lviv.ua

### Сайты ДТЕК: КЕМ, КРЕМ, ДнЕМ, ОЕМ, ДЕМ, ЗЕМ. Публичного API нет

- Сайты: `dtek-kem.com.ua`, `dtek-krem.com.ua`, `dtek-dnem.com.ua`, `dtek-oem.com.ua`, `dtek-dem.com.ua`, `dtek-zem.com.ua`. Страница графиков у всех `/ua/shutdowns`.
- Данные отдаются через WordPress AJAX: `POST /ua/ajax` с `method=getStreets` или `method=getHomeNum`. В ответе группа дома (`GPV3.1`), `preset` (недельный план) и `fact` (график на сегодня и завтра).
- Сайты закрыты WAF Imperva/Incapsula. Простой `curl` получает 403, поэтому без headless-браузера не обойтись. Для серверного бота этот путь хрупкий.
- Официальные боты: @DTEKbot, @Yasno_bot, а также приложение «Київ Цифровий».

### Остальные облэнерго

Графики публикуются на сайтах облэнерго в виде HTML-таблиц, картинок или файлов. Пример — [перечень очередей ГПВ Черкассыоблэнерго](https://www.cherkasyoblenergo.com/static/perelik-gpv). Документированных API нет.

- **Сумыоблэнерго:** [soe.com.ua](https://www.soe.com.ua/consumers/shutdowns?tab=hourly-schedules), публикует только текущий день. Есть кабинет E-Svitlo (`sm.e-svitlo.com.ua`), но для него нужны логин и пароль абонента.
- Приложение **E-Svitlo / єСвітло** обслуживает часть облэнерго: Сумы, Винницу, Чернигов, Донетчину.

### Укренерго и Минэнерго: только контекст

- Укренерго ([ua.energy](https://ua.energy)) публикует сводки: применяются ли ГОП, ГПВ или ГАВ и сколько очередей. Расписания по группам там нет.
- Минэнерго ([mev.gov.ua](https://mev.gov.ua)) публикует ежедневные PDF «Ситуація в енергосистемі».
- Правила: объём ограничений делится на 6 очередей. Одна очередь означает примерно 4 часа без света в сутки, две — 8 часов, три — 12 часов.

## 2. JSON-зеркала сообщества (GitHub)

Это парсеры официальных сайтов, которые выкладывают данные в единый формат. Оба репозитория активны: последний коммит 2026-10-22.

### Baskerville42/outage-data-ua (MIT, 62★). Проверено

- Регионы: `kyiv`, `kyiv-region`, `dnipro`, `odesa` (ДТЕК).
- URL: `https://raw.githubusercontent.com/Baskerville42/outage-data-ua/main/data/<region>.json`.
- Формат:
  - `fact.data[<unix-ts дня>]["GPV1.1"]["1".."24"]` — для каждого часа значение `yes`, `no`, `first` или `second` (первая или вторая половина часа без света);
  - `preset` — недельный план;
  - служебные поля: `lastUpdated`, `lastUpdateStatus`, `meta.contentHash`.
- Ограничение: может отставать от сайта ДТЕК и не содержит аварийных отключений.

### yaroslav2901/OE_OUTAGE_DATA (лицензия не указана, 30★)

- 12 облэнерго: Черкассы, Чернигов, Харьков, Хмельницкий, Львов, Прикарпатье, Полтава, Ровно, Тернополь, Закарпатье, Запорожье, Житомир.
- URL: `https://raw.githubusercontent.com/yaroslav2901/OE_OUTAGE_DATA/main/data/<Name>.json`, например `Lvivoblenerho.json`, `Kharkivoblenerho.json`.
- Формат совместим с outage-data-ua (`fact.data`, `GPV*`). Парсер опрашивает сайты каждые 5 минут.

### Прочие

- Винница: `olnet93/gpv-voe-vinnytsia` и `vn-progr/gpv-voe-vinnytsia` (`data/Vinnytsiaoblenerho.json`).

## 2а. Адреса по очередям (для карты)

Готового набора «адрес → очередь» с координатами нет, его нужно собирать.

**Где взять соответствие адреса и очереди:**
- **Yasno (Киев, Днепр):** адресный API выше. Можно перебрать улицы → дома → группы. Это тысячи запросов, поэтому нужно ограничивать частоту и кэшировать.
- **Сайты ДТЕК:** AJAX `getStreets` и `getHomeNum` возвращают группы для всех домов улицы. Мешает WAF.
- **Львовобленерго:** на poweron.loe.lviv.ua есть поиск по адресу. Endpoint пока не исследован.
- **Перечни облэнерго в PDF/XLSX:**
  - [Черкассыоблэнерго](https://www.cherkasyoblenergo.com/static/perelik-gpv): 12 PDF по очередям и подочередям на период 01.10.2026–01.04.2027;
  - Черниговоблэнерго: [GPV.pdf](https://chernihivoblenergo.com.ua/files/other/GPV.pdf), [Grafik_obmezhen.pdf](https://chernihivoblenergo.com.ua/files/other/Grafik_obmezhen.pdf), адреса по участкам и населённым пунктам;
  - [Хмельницкоблэнерго](https://hoe.com.ua/page/pogodinni-vidkljuchennja): XLSX «Черги ГПВ» по РЭС;
  - у других облэнерго ищите раздел «Перелік адрес по чергах».

  Перечни обновляются раз в сезон, актуальность нужно проверять.

**Геокодирование адресов:**
- OpenStreetMap: здания с `addr:street` и `addr:housenumber` выгружаются через Overpass API одним запросом на город. Получаются полигоны домов, и это лучше всего подходит для карты.
- Nominatim — для единичных адресов: не больше 1 запроса в секунду, массово через него не геокодируют.
- Visicom API — украинский геокодер, точнее по украинским адресам, но платный.
- Сопоставление: нормализовать названия улиц (вул./просп., переименования улиц) и номер дома, затем связать с домами из OSM.

**Карта:** на полигоны домов из OSM положить группу, на группу — текущий слот из `planned-outages`. Светлый — свет есть, тёмный — отключение. Обновлять раз в 5–15 минут.

## 2б. Покрытие по всей Украине (состояние на 2026-10-22)

`fact.update` — время последнего изменения графика в зеркале. Старая дата означает одно из двух: либо график не менялся, либо парсер сломался. Отличить по одной дате нельзя.

| Регион | Оператор | Источник | Свежесть / заметки |
|---|---|---|---|
| Киев | ДТЕК КЕМ | **Yasno API** (официальный) + `Baskerville42/kyiv` | живые данные; 60 групп, по адресу |
| Киевская обл. | ДТЕК КРЕМ | `Baskerville42/kyiv-region` | 22.10 21:44 |
| Днепр и обл. | ДТЕК ДнЕМ, ЦЕК | **Yasno API** + `Baskerville42/dnipro` | Yasno живой; зеркало 17.10 |
| Одесса и обл. | ДТЕК ОЕМ | `Baskerville42/odesa` | 17.10 |
| Донецкая обл. (подконтр.) | ДТЕК ДЕМ | только сайт ДТЕК (WAF) | **зеркала нет** |
| Львовская | ЛОЭ | **api.loe.lviv.ua** (официальный) + yaroslav2901 | зеркало 17.10 |
| Черкасская | Черкассыоблэнерго | yaroslav2901 | 22.10 22:00 |
| Черниговская | ЧОЭ | yaroslav2901 | 22.10 22:21 |
| Полтавская | ПОЭ | yaroslav2901 | 22.10 21:21 |
| Харьковская | ХОЭ | yaroslav2901 | 22.10 00:01 |
| Житомирская | ЖОЭ | yaroslav2901 | 22.10 00:00 |
| Ивано-Франковская | Прикарпатьеоблэнерго | yaroslav2901, [samokisha/outage-data-if](https://github.com/samokisha/outage-data-if) (каждые 10 мин) | 21.10 |
| Ровенская | РОЭ | yaroslav2901 | 20.10 |
| Закарпатская | ЗОЭ | yaroslav2901 | 20.10 |
| Хмельницкая | ХОЭ | yaroslav2901 | 17.10 |
| Тернопольская | ТОЭ | yaroslav2901 | 17.10 |
| Запорожская | ЗОЭ | yaroslav2901, [milkmann/gpv-zoe-zaporizhzhia](https://github.com/milkmann/gpv-zoe-zaporizhzhia) | 17.10 |
| Винницкая | ВОЭ | `olnet93`, `vn-progr/gpv-voe-vinnytsia` (из e-svitlo) | не проверено |
| Сумская | СОЭ | сайт soe.com.ua (только сегодня), E-Svitlo (нужен логин) | **зеркала нет** |
| Волынская | Волыньоблэнерго | только сайт | **зеркала нет** |
| Черновицкая | Черновцыоблэнерго | только сайт | **зеркала нет** |
| Николаевская | Николаевоблэнерго | только сайт | **зеркала нет** |
| Кировоградская | Кировоградоблэнерго | только сайт | **зеркала нет** |
| Херсонская (подконтр.) | Херсоноблэнерго | только сайт | **зеркала нет** |
| Луганская | — | почти полностью оккупирована | вне охвата |

Итог: 17 регионов можно подключить через готовые JSON, ещё 7 потребуют собственных парсеров.

## 3. Готовые сервисы и проекты (для ориентира)

| Сервис | Что делает | Источник данных |
|---|---|---|
| [СвітлоБот](https://svitlobot.online/) | Мониторит факт наличия света: устройство пингует сервис и бот постит в канал. Также показывает графики | Свои устройства, графики с svitlo.live |
| svitlo.live | Агрегатор графиков | Закрыт от curl (403), API не документирован |
| [alerts.energy](https://alerts.energy/kyiv) | Графики по регионам, журнал изменений | Не раскрыт |
| [energy-ua.info](https://energy-ua.info/) | Неофициальный агрегатор: поддомены по регионам (`kyiv.energy-ua.info`), страницы по очередям (`/cherga/1-1`) | Не раскрыт; на запросы без браузера отвечает 403, API не найден |
| [bezsvitla.com.ua](https://bezsvitla.com.ua/kyiv) | Графики по очередям | Не раскрыт |
| @svitloua_bot | Telegram-агрегатор нескольких операторов | — |
| Приложение «Світло» (iOS) | Таймеры до включения и выключения, несколько локаций | Открытые официальные источники |
| [ha-yasno-outages](https://github.com/denysdovhan/ha-yasno-outages) (194★) | Интеграция Home Assistant | API Yasno |
| [ha-svitlo-yeah](https://github.com/milkmann/ha-svitlo-yeah) | Интеграция Home Assistant, около 20 регионов | Yasno, E-Svitlo, JSON-зеркала выше |
| [dtek-blackout-client](https://github.com/Frizoid/dtek-blackout-client) | Python-клиент для сайтов ДТЕК с обходом WAF | Сайты ДТЕК |
| [UA-power-outages-monitor](https://github.com/chaichuk/UA-power-outages-monitor) | Монитор по нескольким регионам | ДТЕК и Yasno |
| [okoua](https://github.com/iluaii/okoua) | PWA с тревогами и графиками, свой `GET /api/power/{uid}` | Агрегирует |

## 4. Рекомендация для бота

1. **Киев и Днепр:** API Yasno как основной источник. Это официальный источник, JSON, есть статус аварийных отключений.
2. **Другие регионы:** JSON-зеркала сообщества. У них один формат на 16+ регионов. Нужно контролировать свежесть по `lastUpdated` и считать данные устаревшими, если они старше примерно 2 дней.
3. **Львов:** можно брать данные напрямую из `api.loe.lviv.ua`, если нужна официальность.
4. Источник держать за интерфейсом адаптера, как требует ТЗ: форматы неофициальные и могут меняться без предупреждения.
5. Нормализовать данные во внутреннюю модель: `группа "X.Y"` → список интервалов `[start, end)` в Europe/Kyiv плюс статус дня.

## Ссылки

- https://github.com/denysdovhan/ha-yasno-outages/blob/main/agents.md — описание API Yasno
- https://github.com/milkmann/ha-svitlo-yeah — перечень источников по регионам
- https://github.com/Baskerville42/outage-data-ua
- https://github.com/yaroslav2901/OE_OUTAGE_DATA
- https://github.com/Frizoid/dtek-blackout-client — протокол сайтов ДТЕК
- https://github.com/topics/yasno, https://github.com/topics/power-outage
- https://poweron.loe.lviv.ua
- https://www.cherkasyoblenergo.com/static/perelik-gpv
- https://www.soe.com.ua/consumers/shutdowns?tab=hourly-schedules
- https://svitlobot.online/
- https://kosht.media/planuy-napered-servisy-shcho-dopomozhut-pry-vidkliuchenni-svitla/
- https://bzh.life/ua/gorod/5-dodatkiv-chatbotiv-i-kart-yaki-dopomozhut-pri-vimknennyah-svitla-spisok/
- https://biz.ligazakon.net/news/232623_v-ukrenergo-poyasnili-yak-dyatimut-nov-pravila-skladannya-grafkv-vdklyuchennya-elektroenerg
