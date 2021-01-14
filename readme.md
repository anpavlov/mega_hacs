# MegaD HomeAssistant custom component

Интеграция с [MegaD-2561](https://www.ab-log.ru/smart-house/ethernet/megad-2561)

## Основные особенности:
- Все порты автоматически добавляются как устройства (для обычных релейных выходов создается 
  `light`, для шим - `light` с поддержкой яркости, для цифровых входов `binary_sensor`, для температурных датчиков
  `sensor`)
- Возможность работы с несколькими megad
- Обратная связь по mqtt
- Команды выполняются друг за другом без конкурентного доступа к ресурсам megad
- Поддержка температурных датчиков в режиме шины

## Зависимости
**Важно!!** Перед использованием необходимо настроить интеграцию mqtt в HomeAssistant

## Установка
Рекомендованнй способ с поддержкой обновлений - через [HACS](https://hacs.xyz/docs/installation/installation).
После установки перейти в меню HACS - Integrations - Explore, в поиске ищем MegaD

Альтернативный способ установки:
```shell
# из папки с конфигом
wget -q -O - https://raw.githubusercontent.com/andvikt/mega_hacs/master/install.sh | bash -
```
Не забываем перезагрузить HA
## Устройства
Поддерживаются устройства: light, switch, binary_sensor, sensor. light может работать как диммер

## Настройка из веб-интерфейса
`Настройки` -> `Интеграции` -> `Добавить интеграцию` в поиске ищем mega

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
