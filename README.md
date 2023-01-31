**Домашнее задание к занятию "2. Работа с Playbook"**

Данный плейбук предназначен для установки `Clickhouse`, `Vector` на хосты, указанные в `inventory` файлах.

**group_vars**

| Variable              | Details                                                            |
|:----------------------|:-------------------------------------------------------------------|
| `clickhouse_version`  | версия `Clickhouse`                                                |
| `clickhouse_packages` | `RPM` пакеты `Clickhouse`, которые необходимо скачать              |
| `vector_url`          | URL адрес для скачивания `RPM` пакетов `Vector`                    |
| `vector_version`      | версия `Vector`                                                    |
| `vector_config`       | раздел, содержащий справку по всем параметрам конфигурации вектора |

**Inventory файл**

Группа `clickhouse` состоит из хоста `centos7` и метод коннекта `docker`.

Группа `vector` состоит из хоста `centos7` и метод коннекта `docker`.

**Playbook**

Playbook состоит из 2 `play`.

Play `Install Clickhouse` применяется на группу `clickhouse` и предназначен для установки и запуска `Clickhouse`.

`handler` для запуска `clickhouse-server`:

```
  handlers:
    - name: Start clickhouse service
      become: true
      become_method: enable
      ansible.builtin.command: "cd bin && clickhouse-server"
```

| Task name                                   | Details                                                                                                                                                                                                                                        |
|---------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Get clickhouse distrib`       | Скачивание `RPM` пакетов, используется `clickhouse_packages` с указанием пакетов. Так как не у всех пакетов есть `noarch` версии, добавляется перехват ошибки `rescue`                                                                         |
| `Install clickhouse packages` | Установка `RPM` пакетов, в `notify` указывается, что task требует запуск handler `Start clickhouse service`, в `become: true` и `become_method: enable` для эскалации привилегий           |
| `Flush clickhouse handlers`   | Добавляем обработчик `Start clickhouse service` для того, чтобы handler выполнился после install task, а не по завершению всех задач. Если пропустить этот этап, то задача создания таблиц завершится ошибкой, тк не запущен clickhouse-server |
| `Create clickhouse database`             | Создаем в `Clickhouse` БД с названием "logs", добавляем условия `failed_when` и `changed_when`, при которых таск будет иметь состояние `failed` и `changed`                                                                                    |

Play `Install Vector` применяется на группу хостов `Vector` и нужен для установки и запуска `Vector`.

`handler` для запуска `vector`:

```
  handlers:
    - name: Start Vector service
      become: true
      become_method: enable
      ansible.builtin.command: "cd bin && vector"
```

| Task name                    | Details                                                                                                          |
|------------------------------|------------------------------------------------------------------------------------------------------------------|
| `Vector download packages`   | Скачивание `RPM` пакетов                                                                                         |
| `Vector install packages`    | Установка `RPM` пакетов                                                                                          |
| `Vector apply template`      | Применяется шаблон конфига `vector`, задается путь конфига, владелец - `ansible`, затем запуск валидации конфига |
| `Change vector system unit` | Изменяется модуль службы `vector`, в `notify` указывается, что task требует запуск handler `Start Vector service`                     |


**Template**

Шаблон `vector.service.j2` нужен для изменения модуля службы `vector`. 
В нем определяется строка запуска `vector`и есть указание, что unit должен быть запущен под пользователем `ansible`.

Шаблон `vector.yml.j2` используется для настройки конфига `vector`. 
В нем указывается, что файл конфига находится в переменной vector_config и его надо преобразовать в `YAML`.
