# Гарантированное удаление данных (Secure Erase)

| Атрибут | Значение |
|---------|----------|
| **ID** | REQ-NFR.DATA.secure-erase |
| **Уровень** | NFR |
| **Атрибут качества** | Functionality |
| **Приоритет** | P0 |
| **Статус** | УТВЕРЖДЕНО |
| **Источник** | — |
| **Критерии приёмки** | — |

---

## Назначение

Фиксирует ответ на вопрос D1.3: есть ли требования к физическому уничтожению данных (secure erase) при выводе VEDO Core из эксплуатации.

Secure erase - это процесс гарантированного удаления информации с носителя HDD, SSD или NVMe с целью невозможности последующего восстановления даже специализированными лабораторными методами. Требования к такому уничтожению возникают у клиентов из государственного сектора, оборонной промышленности, финансовых учреждений с высшим уровнем секретности или при смене подрядчика по обслуживанию инфраструктуры.

Ключевой вывод: VEDO Core в SaaS не осуществляет физическое уничтожение данных. Это задача инфраструктуры: облачного провайдера или хостинга. Для on-premise заказчик выполняет требования своей организации. VEDO Core предоставляет процедуру логического удаления и инструменты для подтверждения, что данных больше нет в системе.

## Нормативная основа

| Нормативный документ | Требование к secure erase | Область применения |
|----------------------|---------------------------|--------------------|
| Приказ ФСТЭК N 21 | Уничтожение информации с подтверждением | Госинформсистемы до 1-го класса защищенности |
| ГОСТ Р 50739-95 | Методы гарантированного удаления, перезапись 3-7 раз | Остаточная намагниченность |
| NIST SP 800-88 Rev. 1 | Clear, Purge, Destroy | Международный стандарт |
| GDPR | Right to be forgotten требует физического уничтожения при невозможности анонимизации | Европейские клиенты |
| 152-ФЗ | Уничтожение при невозможности обезличивания | Операторы ПДн, ст. 21 ч. 5 |

## Матрица ответственности

| Вид развертывания | Кто отвечает за secure erase | Действия VEDO | Действия заказчика |
|-------------------|------------------------------|---------------|--------------------|
| SaaS (мультитенант) | Облачный провайдер: AWS, Yandex Cloud или аналог | Удостовериться, что провайдер поддерживает secure erase и имеет сертификаты | Не требуется |
| Private cloud / VEDO Managed | VEDO по контракту | Выполнить secure erase по договору за дополнительную плату | Не требуется |
| On-premise у заказчика | Заказчик | Предоставить инструкции и инструменты для логического удаления | Выполнить физическое уничтожение носителей |

Рекомендация для on-premise: в договоре с заказчиком прописать, что VEDO Core гарантирует удаление данных на логическом уровне, а физическое уничтожение носителей является ответственностью заказчика.

## Hardware Recycling, E-Waste И Vendor Retirement

Для MVP требования к hardware recycling, e-waste и vendor retirement отсутствуют и не блокируют поставку.

| Модель развертывания | Hardware recycling / e-waste | Vendor retirement | Сертификат утилизации | Ответственный |
|----------------------|------------------------------|-------------------|----------------------|---------------|
| SaaS multitenant MVP | Не является требованием VEDO Core; покрывается политикой cloud provider и сертификатами, если они доступны | Не требуется для MVP | Сертификат cloud provider, если доступен | Cloud provider |
| Customer-managed on-premise | Ответственность заказчика | Не требуется для MVP, потому что ПО распространяется под MIT license и может продолжать работать или быть forked | Ответственность заказчика | Заказчик |
| Managed private cloud | Отложено до Enterprise/P3 contract scope | Отложено до Enterprise/P3 contract scope | Требуется только если contract включает managed hardware retirement | VEDO или заказчик по contract RACI |

Если будущие managed private cloud contracts включают hardware retirement, договор должен определить licensed e-waste operator, inventory серийных номеров дисков, destruction protocol, disposal certificate, cost owner и customer acceptance evidence. Это находится вне MVP.

## Область логического secure erase

Даже без доступа к физическому оборудованию VEDO Core может минимизировать риск восстановления данных.

| Действие | Эффективность | Применимость |
|----------|---------------|--------------|
| Перезапись БД специальными паттернами: `0x00`, `0xFF`, случайные данные | Средняя; SSD могут сохранять данные | Все данные |
| Шифрование данных на уровне диска: LUKS, BitLocker | Высокая; без ключа данные невозможно прочитать | Весь on-premise cluster |
| Удаление ключей шифрования | Высокая, если ключи хранятся отдельно | Ключи хранятся отдельно от данных |
| Триггер на удаление после депровижининга VM | Низкая; VM может быть восстановлена | Виртуальные среды |

## Ограничения удаления backup и cold archive

Secure erase для backup/cold archive имеет отдельные ограничения:

- Данные на WORM-носителях, S3 Object Lock, tape и аналогичных immutable хранилищах нельзя удалить до истечения retention.
- Немедленное удаление одного tenant из общего full backup может нарушить восстановимость других tenant; такие удаления выполняются через backup purge SLA или через уничтожение всего backup storage unit при отдельном согласовании.
- Если backups шифруются tenant-isolated keys, cryptographic erasure через уничтожение ключа допускается как ускоренный способ сделать backup data нечитаемым.
- Glacier-like cold archive может требовать retrieval перед удалением и иметь early-delete cost; плательщик фиксируется в договоре.
- Audit evidence должен явно различать статусы `hard_deleted`, `pending_until_retention_expiry`, `crypto_erased`, `failed` и `out_of_scope`.

## Процедура для on-premise

Административная документация VEDO Core должна содержать процедуру уничтожения данных при выводе из эксплуатации для on-premise.

1. Шифрование данных обязательно: LUKS, BitLocker или шифрование на уровне СУБД.
2. Ключи шифрования должны храниться отдельно от данных.
3. Выполнить логическое удаление: `vedo-cli decommission --purge-data`.
4. Удалить ключи шифрования, если данные хранились в зашифрованном виде.
5. Выполнить физическое уничтожение носителя при необходимости по политике заказчика согласно ГОСТ Р 50739-95 или NIST SP 800-88.
6. Оформить акт уничтожения, подписанный комиссией заказчика; представитель VEDO участвует при необходимости.

## Пример скрипта secure erase

```bash
#!/bin/bash
# secure_erase_data_dirs.sh - запускается при депровижининге on-premise инстанса

shred -f -z -u /etc/vedo/encryption/master.key

for dir in /var/lib/neo4j/data /var/lib/postgresql/data; do
    find "$dir" -type f -exec shred -f -z -u {} \;
    dd if=/dev/urandom of="$dir/dummy" bs=1M count=1024
    rm -f "$dir/dummy"
done

shred -f -z -u /etc/vedo/secrets.env
```

## Подтверждение и evidence

| Метод подтверждения | Сертификация | Применимость | Кому предоставляется |
|---------------------|--------------|--------------|----------------------|
| Логи удаления (audit log) | Нет | Всегда | Клиент |
| Справка от облачного провайдера | SOC 2, ISO 27001 или аналог | SaaS | По запросу |
| Акт уничтожения, подписанный комиссией | Внутренний документ | On-premise | Клиент по факту |
| Сертификат на утилизацию дисков | Лицензия провайдера услуги | On-premise | Клиент, платная услуга |

Рекомендуемые провайдеры и инструменты secure erase для on-premise: Dell Secure Erase, Blancco, KillDisk.

## Формулировка для заказчика

VEDO Core предоставляет средства для логического удаления данных: многократная перезапись случайными данными и удаление ключей шифрования. Физическое уничтожение носителя HDD или SSD осуществляется заказчиком в соответствии с его внутренними политиками.

При on-premise развертывании рекомендуется включить шифрование данных на уровне диска, например LUKS или BitLocker. Тогда для гарантированного уничтожения достаточно удалить ключи шифрования.

Для SaaS клиентов физическое уничтожение данных обеспечивается облачным провайдером, например AWS или Yandex Cloud. По запросу VEDO предоставляет сертификаты и справки о методах уничтожения данных.

## Бизнес-правила

- VEDO Core гарантирует логическое удаление, а не физическое уничтожение, если физическое уничтожение явно не включено в managed/private-cloud contract.
- Для SaaS physical secure erase является ответственностью cloud provider.
- VEDO должен проверять, что SaaS cloud providers поддерживают secure erase и могут предоставить релевантные compliance certificates.
- VEDO Managed / private cloud secure erase выполняется VEDO только если это включено в contract, и может быть платной услугой.
- Для on-premise physical secure erase является ответственностью customer.
- Документация для on-premise должна требовать disk encryption: LUKS, BitLocker или database-level encryption.
- Encryption keys должны храниться отдельно от encrypted data.
- VEDO должен предоставлять tooling для logical deletion через `vedo-cli decommission --purge-data`.
- Если customer policy требует physical destruction, customer следует ГОСТ Р 50739-95, NIST SP 800-88 или internal policy.
- Evidence может включать audit logs, cloud provider certificate, destruction act или disk disposal certificate.
- Backup/cold archive deletion follows backup purge SLA; immediate deletion from immutable media is not promised unless the whole storage unit is destroyed or tenant-isolated encryption keys are destroyed.
- Hardware recycling, e-waste и vendor retirement не являются MVP requirements.
- Для SaaS hardware recycling/e-waste является ответственностью cloud provider.
- Для customer-managed on-premise hardware recycling/e-waste и physical media disposal являются ответственностью customer.
- Managed private cloud hardware retirement requirements относятся к Enterprise/P3 contract scope и должны определять disposal certificate и licensed operator requirements, если они включены.
- Vendor retirement не требует MVP process, потому что MIT license позволяет customers продолжать эксплуатацию VEDO Core или fork.

## Cryptographic erasure — проверка после decommission

После decommission и purge данных должны проверяться:

| Компонент | Что проверять | Макс. время |
|-----------|---------------|-------------|
| Neo4j | Ключи шифрования диска удалены; индексы TBox/ABox не восстанавливаются без ключа | 10 минут |
| PostgreSQL | Ключи шифрования (LUKS/TDE) удалены; WAL очищены | 10 минут |
| Redis | RDB/AOF файлы удалены, память очищена (FLUSHALL) | 5 минут |
| S3/MinIO | Object paths tenant-специфичных ключей проверены на отсутствие | 15 минут |
| API Gateway | Токены/сессии tenant аннулированы, кэш очищен | 5 минут |
| Backup | Все backup-копии tenant удалены (или исключены из restore policy) | 30 минут |

Инструмент: `vedo-cli decommission --verify-purge` запускает все проверки и выдаёт отчёт.

## Открытые вопросы

- Нет открытых вопросов по D1.3.
