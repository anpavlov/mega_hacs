

# MegaD HomeAssistant integration
[![hacs_badge](https://img.shields.io/badge/HACS-Custom-orange.svg)](https://github.com/custom-components/hacs)
[![Donate](https://img.shields.io/badge/donate-Yandex-red.svg)](https://yoomoney.ru/to/410013955329136)

Интеграция с [MegaD-2561, MegaD-328](https://www.ab-log.ru/smart-house/ethernet/megad-2561)

Если вам понравилась интеграция, не забудьте поставить звезду на гитхабе - вам не сложно, а мне приятно ) А если
интеграция очень понравилась - еще приятнее, если вы воспользуетесь кнопкой доната )

Обновление прошивки MegaD можно делать прямо из HA с помощью [аддона](https://github.com/andvikt/mega_addon.git)
## Основные особенности:
- Настройка в веб-интерфейсе + yaml
- Все порты автоматически добавляются как устройства (для обычных релейных выходов создается 
  `light`, для шим - `light` с поддержкой яркости, для цифровых входов `binary_sensor`, для датчиков
  `sensor`)
- Возможность работы с несколькими megad
- Обратная связь по mqtt или http (на выбор)
- События на двойные/долгие нажатия
- Команды выполняются друг за другом без конкурентного доступа к ресурсам megad, это дает гарантии надежного исполнения
  большого кол-ва команд (например в сценах). Каждая следующая команда отправляется только после получения ответа о
  выполнении предыдущей.
- поддержка ds2413 (начиная с версии 0.4.1)

## Установка
Рекомендованный способ с поддержкой обновлений - [HACS](https://hacs.xyz/docs/installation/installation):

HACS - Integrations - Explore, в поиске ищем MegaD. 

Альтернативный способ установки:
```shell
# из папки с конфигом
wget -q -O - https://raw.githubusercontent.com/andvikt/mega_hacs/master/install.sh | bash -
```
Не забываем перезагрузить HA

## Настройка
`Настройки` -> `Интеграции` -> `Добавить интеграцию` в поиске ищем mega

Все имеющиеся у вас порты будут настроены автоматически. Вы можете менять названия, иконки и entity_id так же из интерфейса.

#### Кастомизация устройств с помощью yaml:
```yaml
# configuration.yaml

mega:
  hello: # ID меги, как в UI 
    7: # номер порта
      domain: switch # тип устройства (switch или light, по умолчанию для цифровых выходов используется light)
      invert: true # инвертировать или нет (по умолчанию false)
      name: Насос # имя устройства
    8:
      # исключить из сканирования
      skip: true
    10:
      skip: false # если нужен skip, то он работает только целиком для всего порта
      # в случае если порт настроен как ds2413  
      address_a:  #address - это адрес ds2413
        # здесь внутри работают все стандартные поля конфигурирования порта
        invert: true
        domain: switch
      c6c439000000_b:
        name: какое-то имя
    33:
      # для датчиков можно кастомизировать только имя и unit_of_measurement
      # для температуры и влажность unit определяется автоматически, для остальных юнита нет 
      name:
        hum: "влажность"
        temp: "температура"
      unit_of_measurement:
        hum: "%" # если датчиков несколько, то можно указывать юниты по их ключам
        temp: "°C"
      # можно так же указать шаблон для конвертации значения, может быть полезно для ацп-входа
      # текущее значение порта передается в шаблон в переменной "value"
      conv_template: "{{(value|float)/100}}"
    14:
      name: какой-то датчик
      unit_of_measurement: "°C" # если датчик один, то просто строчкой
```

## Зависимости
Для совместимости c mqtt необходимо настроить интеграцию [mqtt](https://www.home-assistant.io/integrations/mqtt/) 
в HomeAssistant, а так же обновить ваш контроллер до последней версии, тк были важные обновления в части mqtt

## HTTP in
Начиная с версии `0.3.1` интеграция стала поддерживать обратную связь без mqtt, используя http-сервер. Для этого в настройках
интеграции необходимо снять галку с `использовать mqtt`

**Внимание!** Не используйте srv loop на контроллере, это может приводить к ложным срабатываниям входов. Вместо srv loop 
интеграция будет сама обновлять все состояния портов с заданным интервалом

В самой меге необходимо прописать настройки:
```yaml
srv: "192.168.1.4:8123" # ip:port вашего HA
script: "mega" # это api интеграции, к которому будет обращаться контроллер
```

#### Ответ на входящие события от контроллера
Контроллер ожидает ответ от сервера, который может быть сценарием (по умолчанию интеграция отвечает `d`, что означает 
запустить то что прописано в поле act в настройках порта).

Поддерживаются шаблоны HA. Это может быть использовано, например, для запоминания яркости (тк сам контроллер этого не 
умеет). В шаблоне можно использовать параметры, которые передает контроллер (m, click, pt, mdid, mega_id)

Примеры:
```yaml
mega:
  mega1: # id меги, который вы сами придумываете в конфиге в UI
    4: # номер порта, с которого ожидаются события
      response_template: "5:2" # простейший пример без шаблона. Каждый раз когда будет приходить сообщение на этот порт, 
                             # будем менять состояние на противоположное
    5:
      # пример с использованием шаблона, порт 1 будет выключен если он сейчас включен и включен с последней сохраненной 
      # яркостью если он сейчас выключен     
      response_template: >-
        {% if is_state('light.some_port_1', 'on') %}
        1:0
        {% else %}
        1:{{state_attr('light.some_port_1', 'brightness')}}
        {% endif %}
    6:
      # в шаблон так же передаются все параметры, которые передает контроллер (pt, cnt, m, click)
      # эти параметры можно использовать в условиях или непосредственно в шаблоне в виде {{pt}}
      response_template: >-
        {% if m==2 %}1:0{% else %}d{% endif %}
```

Начиная с версии v0.3.17 ответ можно слать так же и в режиме MQTT. Аналогично, темплейт должен возвращать готовую команду
такую же как требует команда cmd, так же можно использовать d, но d не отправляется по умолчанию, это сделано чтобы не 
сломать текущую логику у пользователей предыдущих версий. Чтобы включить для всех входов в режиме mqtt отправку команды 
d необходимо в конфиге прописать следующее:
```yaml
mega:
  mega1:
    force_d: true
```
**Внимание!** Нельзя использовать чекбокс напротив поля act если планируется использовать ответ сервера - у вас и 
сработает act и команда от сервера, а вслучае ответа d сработает act два раза.

Так же следует понимать, что это не "ответ" в нормальном понимании - это вызов следом за полученным mqtt-сообщением
http команды такого вида `http://megaurl/?pt=port&cmd=rendered_template`, где `port` - это номер порта сработавшего входа,
а `cmd` - текст команды, который получен из темплейта. Те это имитация ответа. У этого подхода есть минус - задержка в 
исполнении будет значительно выше чем при ответе в режиме http, но тем не менее эта задержка скорее всего не будет 
сильно заметна.

## binary_sensor и события

Входы будут доступны как binary_sensor, а так же в виде событий `mega.sensor` и `mega.binary`.
Для корректной работы binary_sensor имеет смысл использовать режим P&R, для остальных режимов - лучше пользоваться 
событиями.

События можно использовать в автоматизациях, например так:
```yaml
# Пример события с полями как есть прямо из меги
- alias: some double click
  trigger:
    - platform: event
      event_type: mega.sensor
      event_data:
        pt: 1
        click: 2
  action:
    - service: light.toggle
      entity_id: light.some_light
```
События могут содержать следующие поля: 
- `mega_id`: id как в конфиге HA
- `pt`: номер порта
- `cnt`: счетчик срабатываний
- `mdid`: if как в конфиге контроллера
- `click`: клик (подробнее в документации меги)
- `value`: текущее значение (только для mqtt)
- `port`: номер порта

Начиная с версии 0.3.7 появилось так же событие типа `mega.binary`:
```yaml
# Пример события с полями как есть прямо из меги
- alias: some long click
  trigger:
    - platform: event
      event_type: mega.binary
      event_data:
        entity_id: binary_sensor.some_id
        type: long
  action:
    - service: light.toggle
      entity_id: light.some_light
```
Возможные варианты поля `type`:
- `long`: долгое нажатие
- `release`: размыкание (с гарантией** что не было долгого нажатия)
- `long_release`: размыкание после долгого нажатия
- `press`: замыкание
- `single`: одинарный клик (в режиме кликов)
- `double`: двойной клик

**гарантия есть только при использовании http-метода синхронизации, mqtt не гарантирует порядок доставки сообщений, хотя 
маловероятно, что порядок будет нарушен, но все же сам протокол не дает таких гарантий.

Чтобы понять, какие события происходят, лучше всего воспользоваться панелью разработчика и подписаться
на вкладке события на событие `mega.sensor`, понажимать кнопки.

## Сервисы
Все сервисы доступны в меню разработчика с описанием и примерами использования
```yaml
mega.save:
  description: Сохраняет текущее состояние портов (?cmd=s)
  fields:
    mega_id:
      description: ID меги, можно оставить пустым, тогда будут сохранены все зарегистрированные меги
      example: "mega"

mega.get_port:
  description: Запросить текущий статус порта (или всех)
  fields:
    mega_id:
      description: ID меги, можно оставить пустым, тогда будут порты всех зарегистрированных мег
      example: "mega"
    port:
      description: Номер порта (если не заполнять, будут запрошены все порты сразу)
      example: 1

mega.run_cmd:
  description: Выполнить любую произвольную команду
  fields:
    mega_id:
      description: ID меги
      example: "mega"
    port:
      description: Номер порта (это не порт, которым мы управляем, а порт с которого шлем команду)
      example: 1
    cmd:
      description: Любая поддерживаемая мегой команда
      example: "1:0"
```

## Отладка
Интеграция находится в активной разработке, при возникновении проблем [заводите issue](https://github.com/andvikt/mega_hacs/issues/new/choose)

Просьба прикладывать детальный лог, который можно включить в конфиге так:
```yaml
logger:
  default: info
  logs:
    custom_components.mega: debug
```

#### Отладка ответов http-сервера
Для отладки ответов сервера можно самим имитировать запросы контроллера, если у вас есть доступ к консоли
HA:
```shell
curl -v -X GET 'http://localhost:8123/mega?pt=5&m=1'
```
Если доступа нет, нужно в файл конфигурации добавить ip, с которого вы хотите делать запросы, например:
```yaml
mega:
  allow_hosts:
    - 192.168.1.1
```
И тогда можно с локальной машины делать запросы на ваш сервер HA:
```shell
curl -v -X GET 'http://192.168.88.1.4:8123/mega?pt=5&m=1'
```
В ответ будет приходить либо `d`, либо скрипт, который вы настроили