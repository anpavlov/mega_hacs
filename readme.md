# MegaD HomeAssistant integration

Интеграция с [MegaD-2561](https://www.ab-log.ru/smart-house/ethernet/megad-2561)

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

## Зависимости
**Важно!!** Для максимальной совместимости необходимо настроить интеграцию [mqtt](https://www.home-assistant.io/integrations/mqtt/) 
в HomeAssistant, а так же обновить ваш контроллер до последней версии, тк были важные обновления в части mqtt

## HTTP in
Начиная с версии `0.3.1` интеграция стала поддерживать обратную связь без mqtt, используя http-сервер. Для этого в настройках
интеграции необходимо снять галку с `использовать mqtt`

В самой меге необходимо прописать настройки:
```yaml
srv: "192.168.1.4:8123" # ip:port вашего HA
script: "mega" # это api интеграции, к которому будет обращаться контроллер
```

Входы будут доступны как binary_sensor, а так же в виде событий `mega.sensor`.
События можно обрабатывать так:
```yaml
- alias: some double click
  trigger:
    - platform: event
      event_type: mega.sensor
      event_data:
        pt: 1
  action:
    - service: light.toggle
      entity_id: light.some_light
```
Для binary_sensor имеет смысл использовать режим P&R, для остальных режимов - лучше пользоваться событиями.

## Ответ на входящие события от контроллера
Контроллер ожидает ответ от сервера, который может быть сценарием (по умолчанию интеграция отвечает `d`, что означает 
запустить то что прописано в поле act в настройках порта).

Поддерживаеются шаблоны HA. Это может быть использовано, например, для запоминания яркости (тк сам контроллер этого не 
умеет). В шаблоне можно использовать параметры, которые передает контроллер (m, click, pt, value)

Примеры:
```yaml
mega:
  mega1: # id меги, который вы сами придумываете в конфиге в UI
    4: # номер порта, с которого ожидаются события
      response_template: 5:2 # простейший пример без шаблона. Каждый раз когда будет приходить сообщение на этот порт, 
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
    33:
      # для датчиков можно кастомизировать только имя и unit_of_measurement
      # для температуры и влажность unit определяется автоматически, для остальных юнита нет 
      name:
        hum: "влажность"
        temp: "температура"
      unit_of_measurement:
        hum: "%" # если датчиков несколько, то можно указывать юниты по их ключам
        temp: "°C"
    14:
      name: какой-то датчик
      unit_of_measurement: "°C" # если датчик один, то просто строчкой
```

## События
`binary_sensor` срабатывает когда цифровой выход принимает значение 'ON'. `binary_sensor` имеет смысл использовать
только с режимом входа P&R

При каждом срабатывании `binary_sensor` так же сообщает о событии типа `mega.sensor`.
События можно использовать в автоматизациях, например так:
```yaml
- alias: some double click
  trigger:
    - platform: event
      event_type: mega.sensor
      event_data:
        pt: 1
        cnt: 2
  action:
    - service: light.toggle
      entity_id: light.some_light
```
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
