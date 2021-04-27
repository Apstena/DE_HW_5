Проверкe на not null не стал дубилировать как типовую почти по всем полям.
Проверки на типы не рассматривал т.к. тут должны работать ограничения СУБД.

Дополнительно: 
возникла проблема в ДЗ 3 где часть данных была загружена некорретно.
Недостающие значения заполнил самостоятельно:


12:27
```sql
update adubinsky.ods_issue i
set start_time = p.pay_date + (random()*6)::smallint * '1 year'::interval + (random()*11)::smallint * '1 month'::interval + (random()*30)::smallint * '1 day'::interval
from (select user_id, min(pay_date) pay_date from adubinsky.ods_payment
group by user_id)p 
where p.user_id=i.user_id;

update adubinsky.ods_issue i
set end_time = start_time + (random()*7)::smallint * '1 day'::interval;

update adubinsky.ods_payment
set pay_doc_type = case when right(account, 1)::int between 0 and 3 then 'VISA'
                    when right(account, 1)::int between 4 and 7 then 'MASTERCARD'
                    when right(account, 1)::int between 8 and 8 then 'MIR'
                    when right(account, 1)::int between 9 and 9 then 'MAESTRO'
       end;
```




### Payment
#### `user_id`
Просто проверка на NOT NULL т.к. по таблице платежей в дальнейшем строятся пользователи
```python 
batch.expect_column_values_to_not_be_null(column='user_id')
```
#### `pay_doc_type`
Проверка что тип документа входит в определенный набор значений.
```python 
batch.expect_column_distinct_values_to_be_in_set(column='pay_doc_type', value_set=['MAESTRO', 'MASTERCARD', 'MIR', 'VISA', 'AE'])
```
#### `pay_doc_num`
Выбрана проверка на уникальность т.к. ожидается что у каждого платежного документа будет свой уникальный номер. Найдено 5 дублей по атрибуту.
```python batch.expect_column_values_to_be_unique(column='pay_doc_num')```
#### `account`
Выбрана проверка на то что лицевой счет должен быть в определенном формате
```python 
batch.expect_column_values_to_match_regex(column='account',regex='^FL-[0-9]*')
```
#### `phone`
Проверка на корректость телефонного номера в определенном формате
```python 
batch.expect_column_values_to_match_regex(column='phone',regex='^(\+7)[\d]{10}')
```
#### `billing_period`
Проверка что все значения биллингового периода записаны единообразно и в корректном диапазоне
```python 
batch.expect_column_values_to_match_regex(column='billing_period',regex='^(20)(1|2)[0-9]{1}(-)(0?[1-9]|1[0-2])')
```
#### `pay_date`
Проверка на то что дата платежа в корректном временом промежутке. В идеале конечно бы хотелось проверять с текущей датой как с максимальным значением.
```python 
batch.expect_column_values_to_be_between(column='pay_date',min_value='2013-01-01',max_value='2021-12-31')
```
#### `sum`
Проверка что сумма платежа должна быть положительной.
```python 
batch.expect_column_values_to_be_between(column='sum',min_value='0',max_value='999999999999')
```
#### `src_name`
Проверка что данные загруженны из нужного источника
```python 
batch.expect_column_distinct_values_to_be_in_set(column='src_name', value_set=['stg_payment'])
```


### Billing
#### `user_id`
Добавил проверку распределения - допустил ситуацию что по этому полю идет дистрибьюция в БД и если обнаружится перекос в данных - пересмотреть дистрибьюцию таблицы для более корректного распределения данных по сегментам в БД.
```python 
batch.expect_column_kl_divergence_to_be_less_than(column='user_id')
```
#### `billing_period`
Проверка что все значения биллингового периода записаны единообразно и в корректном диапазоне
```python 
batch.expect_column_values_to_match_regex(column='billing_period', regex='^(20)(1|2)[0-9]{1}')
```
#### `service`
Проверка что длина строки услуги равна 14 символам. Можно было это сделать в виде регулярного выражения, но выбрал длину только из-за разнообразия в тестовом задании.
```python 
batch.expect_column_value_lengths_to_equal(column='service', value='14')
```
#### `tariff`
Проверка что тариф находится в определенном диапазоне значений
```python 
batch.expect_column_distinct_values_to_be_in_set(column='tariff', value_set=['Gigabyte', 'Maxi', 'Mini', 'Megabyte'])
```
#### `sum`
Проверка что сумма начислений положительная (хотя в реальности скорее всего бы это было некорректно, раз у нас нет таблицы корректировок)
```python 
batch.expect_column_values_to_be_between(column='sum', max_value='10', min_value='0')
```
#### `created_at`
Проверка что дата создания в корректном диапазоне
```python 
batch.expect_column_values_to_be_between(column='created_at', max_value='2021-12-31', min_value='2013-01-01')
```
#### `src_name`
Проверка что данные загруженны из нужного источника
```python 
batch.expect_column_distinct_values_to_be_in_set(column='src_name', value_set=['stg_billing'])
```

### Traffic
#### `user_id`
Аналогичная биллингу проверка распределения, но только с указанным явно трешхолдом
```python 
batch.expect_column_kl_divergence_to_be_less_than(column='user_id', threshold=0.01)
```
#### `device_id`
Сделал проверку на то что идентификатор устройства имеем корректный типовой формат
```python 
batch.expect_column_values_to_match_regex(column='device_id', regex='^(d)[\d]{3}')
```
#### `device_ip_addr`
Проверка IP-адреса на валидность
```python 
batch.expect_column_values_to_match_regex(column='device_ip_addr', regex='((?:(?:25[0-5]|2[0-4]\d|((1\d{2})|([1-9]?\d)))\.){3}(?:25[0-5]|2[0-4]\d|((1\d{2})|([1-9]?\d))))')
```
#### `bytes_sent`
Проверка что количество полученных байт положительное, решил оформить в такой вариации чтобы использовать агрегатные проверки
```python 
batch.expect_column_min_to_be_between(column='bytes_sent', max_value=10000, min_value=0)
```
#### `bytes_received`
Принял условность что количество отправленных байт не может быть больше 50000 и должна создаваться новая сессия, в таком случае применил проверку максимальное значение
```python 
batch.expect_column_max_to_be_between(column='bytes_received', max_value=50000, min_value=0)
```
#### `src_name`
Проверка что данные загруженны из нужного источника
```python 
batch.expect_column_distinct_values_to_be_in_set(column='src_name', value_set=['stg_traffic'])
```


### Issue
#### `user_id`
Простая проверка на NOT NULL. Каких-то координально новых и интересных проверок на нашел чтобы применить.
```python 
batch.expect_column_values_to_not_be_null(column='user_id')
```
#### `start_time`
Очень хотелось применить проверку на то что значение A больше B с флагом эквивалентности для того чтобы проверить что end_time не ранее start_time, но, к сожалению, согласно документации даная проверка не поддерживается для SQL.
Проверка на формат даты в SQL тоже не поддерживается (ну это логично), поэтому просто проверка на попадание в корректны диапазон.
```python 
batch.expect_column_values_to_be_between(column='start_time',min_value='2013-01-01',max_value='2021-12-31')
```
#### `end_time`
Просто проверка на попадание в корректны диапазон.
```python
batch.expect_column_values_to_be_between(column='end_time',min_value='2013-01-01',max_value='2021-12-31')
```
#### `title`
Проверка что строка заполненена только буквами латинского алфавита
```python 
batch.expect_column_values_to_match_regex(column='title',regex='^[A-Za-z]+')
```
#### `description`
Проверка что строка заполненена только буквами латинского алфавита
```python 
batch.expect_column_values_to_match_regex(column='description',regex='^[A-Za-z]+')
```
#### `service`
Проверка что услуга находится в определенном корректном списке
```python 
batch.expect_column_distinct_values_to_be_in_set(column='service', value_set=['Setup Environment','Connect','Disconnect'])
```
#### `src_name`
Проверка что данные загруженны из нужного источника
```python 
batch.expect_column_distinct_values_to_be_in_set(column='src_name', value_set=['stg_issue'])
```
