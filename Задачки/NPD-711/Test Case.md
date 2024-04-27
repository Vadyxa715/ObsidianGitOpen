#NPD-711
# Общее описание процесса

НП встает на учет --> Приходит подтверждение от ЦУН - дальнейшим действиям присваивается `obj_id` (Идентификатор операции из ЦУН) 
--> далее решения из НО оперируют этим идентификатором для принятия решений или их отмены и т.д.
 в npd_decision.ais_obj_id должен совпадать с tp_registration_history.obj_id - в контексте решения: решение применяется к id регистрации. По разным регистрациям/постановкам, применяются разные решения.

при регистрации выдается id регистрации obj_id и решения применяются в рамках этого id - ais_obj_id 

==Важно понимать - чтобы был актуальный tsun_id так как наша tp_pergistration_history это копирка ЦУН-овской таблички==

`tp_registration_history` внутренняя таблица лога постановок и снятия. 

1. Поставиться на учет
2. набить чеков
3. Создать Решение(Decision)
	1. Указать верный obj_id (смотреть в tp.tp_registration_history)
	2. Указать даты аннулирования чеков (чтобы совпадало с днем пробития чеков)
	3. handled_at = null (поллинг подтягивает всех решения у которых handled_at = null)
	4. ais_id = fid
	5. tsun_id - (для тестов любой) *уникальный индекс (sequns из таблички ЦУН по большому счету уникальный номер документа/действия)
4. После отмены решения в таблице tp.tp_registration_history должно быть соответствие полей tp.npd_decision.ais_tsun_id = tp.tp_registration_history.tsun_id по данному fid в рамках последнего примененного без ошибок решения.
5. ==У decision типа CANCEL_NON_COMPLIANCE есть поле ais_cancel_id которое должно совпадать с id решения которое собираемся отменить.==


# Что нужно тестировать
По большому счету 3 случая
1. Одно решение о снятии
2. решение о снятии + решение об отмене снятия
3. решение о снятии + отмена + решение о новом снятии
4. решение + отмена + N... + отмена
5. решение + отмена + N... + решение о снятии


# 1 Кейс (reg_action_needed=true)
##### Описание действия
1.  В рамках одного `obj_id`
	1. НП встает на учет
	2. НП снимается с учета
 ###### Ожидаемый результат
 Одна запись в таблице tp_registration_history по obj_id
Данная запись должна иметь `DELETED = false` 
## Тестовые данные
#### Вставки в таблицы
```sql title:Снятие_по_решению
INSERT INTO tp.npd_decision
(ais_id, fid, "type", created_at, over_income, reg_action_needed, state_time, handled_at, reg_result, ais_inn, ais_tax_organ_code, ais_doc_type_id, ais_doc_number, ais_doc_date, ais_register_time, ais_decision_date, ais_correct_date, ais_process_state_id, ais_reason_id, ais_reason_name, ais_on_uch_date, ais_off_uch_date, ais_cancel_id, ais_obj_id, ais_tsun_id, ais_reg_tax_organ_code, ais_reg_tax_organ_name)
VALUES(270113, 877777777777, 'NON_COMPLIANCE', '2022-04-09 07:00:04.396', false, true, '2022-04-08 15:07:21.000', NULL, NULL, '877777777777', '2463', 3, '270112', '2022-04-08', '2022-04-08 19:05:41.000', '2021-05-06', '2021-05-06', 6, 6, 'Причина', '2021-04-30', '2021-05-06', NULL, 1000327267802, 1001376991255, '2463', 'НО')
```

```sql title:create_decision_type_CANCEL_NON_COMPLIANCE
INSERT INTO tp.npd_decision
(ais_id, fid, "type", created_at, over_income, reg_action_needed, state_time, handled_at, reg_result, ais_inn, ais_tax_organ_code, ais_doc_type_id, ais_doc_number, ais_doc_date, ais_register_time, ais_decision_date, ais_correct_date, ais_process_state_id, ais_reason_id, ais_reason_name, ais_on_uch_date, ais_off_uch_date, ais_cancel_id, ais_obj_id, ais_tsun_id, ais_reg_tax_organ_code, ais_reg_tax_organ_name)
VALUES(301475, 877777777777, 'CANCEL_NON_COMPLIANCE', '2022-06-22 07:00:03.552', false, true, '2022-06-21 10:27:21.000', NULL, NULL, '877777777777', '2463', 2, '301478', '2022-06-21', '2022-06-21 10:26:33.000', NULL, NULL, 6, NULL, NULL, '2021-04-30', NULL, 270112, 1000327267802, 1001812641849, '2463', 'Инспекция Федеральной налоговой службы по Октябрьскому району г.Красноярска');
```
###### Отчет
- [ ] получилось воспроизвести
- [ ] проверен
- [ ] покрыт тестами
# 2 Кейс (reg_action_needed=true)
###### Описание
1)  В рамках одного `obj_id`
	1) НП встает на учет
	2) НП снимается по решению НО
2) Далее не происходит никаких действий

 ###### Ожидаемый результат
 Одна запись в таблице tp_registration_history по obj_id
Данная запись должна иметь `DELETED = false` 
###### Отчет
- [x] получилось воспроизвести
- [x] проверен
- [ ] покрыт тестами

# 3 Кейс (reg_action_needed=true)
###### Описание
1)  В рамках одного `obj_id`
	1) НП встает на учет
	2) НП снимается с учета по решению НО (принятие решения)
	3) НП встает на учет по решению НО (отмена решения)
 
 ###### Ожидаемый результат
 Должно появиться 2 записи в таблице tp_registration_hustory по данному fid и данному obj_id:
* одна с датой постановки и с датой снятия помеченная `DELETED = true`
* вторая с датой постановки без даты снятия помеченная  `DELETED = false`
###### Отчет
- [ ] получилось воспроизвести
- [ ] проверен
- [ ] покрыт тестами

# 4 Кейс (reg_action_needed=true)
###### Описание
1) В рамках одного решения один и тот же `obj_id`
	1)  НП встает на учет
	2) НП снимается с учета по решению НО (принятие решения) 
	3) НП встает на учет по решению НО (отмена решения)
	4) НП снимается с учета по решению НО (принятие решения) *чаще всего встречается с корректировкой дат аннулирования чеков*
 
 ###### Ожидаемый результат
 Должно появиться 2 записи в таблице `tp_registration_hustory` по данному `fid` и данному `obj_id`:
* одна с датой постановки и с датой снятия помеченная `DELETED = true` 
* вторая с датой постановки и с датой снятия помеченная `DELETED = false`

###### Отчет
- [ ] получилось воспроизвести
- [ ] проверен
- [ ] покрыт тестами

# 5 Кейс (reg_action_needed=false)
С флагами `unreg=false` действия по снятию или регистрации не происходит логгирование в таблицу `tp_registration_history` не происходит.  
# Тестовые данные 6 решений последовательно 
обратить внимание на поля: 
	ais_id
	handled_at = null
	ais_tsun_id
	ais_obj_id = tp.tp_reg_hist.obj_id bu fid
	ais_cancel_id
	аннулировать чеки с (ais_decision_date) по (ais_correct_date)
```sql title:decisions
--Принятие решения по fid-131111111111

INSERT INTO tp.npd_decision
(ais_id, fid, ais_obj_id, "type", created_at, over_income, reg_action_needed, state_time, handled_at, reg_result, ais_inn, ais_tax_organ_code, ais_doc_type_id, ais_doc_number, ais_doc_date, ais_register_time, ais_decision_date, ais_correct_date, ais_process_state_id, ais_reason_id, ais_reason_name, ais_on_uch_date, ais_off_uch_date, ais_cancel_id, ais_tsun_id, ais_reg_tax_organ_code, ais_reg_tax_organ_name)
VALUES(88831, 141111111111, 1000420590924, 'NON_COMPLIANCE', '2024-04-24 07:00:06.690', false, true, NULL, NULL, NULL, '141111111111', '2463', 3, '37033798', '2024-04-24', '2024-04-24 16:52:53.000', '2024-04-24', '2024-04-24', 6, 6, 'Причина', '2024-04-24', '2024-04-24', NULL, 91919191, '2463', 'НО');

--отмена решения по fid-131111111111
INSERT INTO tp.npd_decision
(ais_id, fid, ais_obj_id, "type", created_at, over_income, reg_action_needed, state_time, handled_at, reg_result, ais_inn, ais_tax_organ_code, ais_doc_type_id, ais_doc_number, ais_doc_date, ais_register_time, ais_decision_date, ais_correct_date, ais_process_state_id, ais_reason_id, ais_reason_name, ais_on_uch_date, ais_off_uch_date, ais_cancel_id, ais_tsun_id, ais_reg_tax_organ_code, ais_reg_tax_organ_name)
VALUES(88832, 141111111111, 1000420590924, 'CANCEL_NON_COMPLIANCE', '2024-04-24 10:00:03.552', false, true, NULL, NULL, NULL, '141111111111', '2463', 2, '39701474', '2024-04-24', '2024-04-24 10:26:33.000', NULL, NULL, 6, NULL, NULL, '2024-04-24', NULL, 88831, 91919192, '2463', 'НО');

--Принятие 2-го решения
INSERT INTO tp.npd_decision
(ais_id, fid, ais_obj_id, "type", created_at, over_income, reg_action_needed, state_time, handled_at, reg_result, ais_inn, ais_tax_organ_code, ais_doc_type_id, ais_doc_number, ais_doc_date, ais_register_time, ais_decision_date, ais_correct_date, ais_process_state_id, ais_reason_id, ais_reason_name, ais_on_uch_date, ais_off_uch_date, ais_cancel_id, ais_tsun_id, ais_reg_tax_organ_code, ais_reg_tax_organ_name)
VALUES(88833, 141111111111, 1000420590924, 'NON_COMPLIANCE', '2024-04-24 15:00:06.690', false, true, NULL, NULL, NULL, '141111111111', '2463', 3, '37033800', '2024-04-24', '2024-04-24 16:52:53.000', '2024-04-24', '2024-04-24', 6, 6, 'Причина', '2024-04-24', '2024-04-24', NULL, 91919193, '2463', 'НО');

--отмена 2-го решения по fid-131111111111
INSERT INTO tp.npd_decision
(ais_id, fid, ais_obj_id, "type", created_at, over_income, reg_action_needed, state_time, handled_at, reg_result, ais_inn, ais_tax_organ_code, ais_doc_type_id, ais_doc_number, ais_doc_date, ais_register_time, ais_decision_date, ais_correct_date, ais_process_state_id, ais_reason_id, ais_reason_name, ais_on_uch_date, ais_off_uch_date, ais_cancel_id, ais_tsun_id, ais_reg_tax_organ_code, ais_reg_tax_organ_name)
VALUES(88834, 141111111111, 1000420590924, 'CANCEL_NON_COMPLIANCE', '2024-04-24 15:10:03.552', false, true, NULL, NULL, NULL, '141111111111', '2463', 2, '39701801', '2024-04-24', '2024-04-24 10:26:33.000', NULL, NULL, 6, NULL, NULL, '2024-04-24', NULL, 88833, 91919194, '2463', 'НО');

--Принятие 3-го решения пошло поехало...
INSERT INTO tp.npd_decision
(ais_id, fid, ais_obj_id, "type", created_at, over_income, reg_action_needed, state_time, handled_at, reg_result, ais_inn, ais_tax_organ_code, ais_doc_type_id, ais_doc_number, ais_doc_date, ais_register_time, ais_decision_date, ais_correct_date, ais_process_state_id, ais_reason_id, ais_reason_name, ais_on_uch_date, ais_off_uch_date, ais_cancel_id, ais_tsun_id, ais_reg_tax_organ_code, ais_reg_tax_organ_name)
VALUES(88835, 141111111111, 1000420590924, 'NON_COMPLIANCE', '2024-04-24 15:00:06.690', false, true, NULL, NULL, NULL, '141111111111', '2463', 3, '37033800', '2024-04-24', '2024-04-24 16:52:53.000', '2024-04-24', '2024-04-24', 6, 6, 'Причина', '2024-04-24', '2024-04-24', NULL, 91919195, '2463', 'НО');

--отмена 3-го решения по fid-131111111111
INSERT INTO tp.npd_decision
(ais_id, fid, ais_obj_id, "type", created_at, over_income, reg_action_needed, state_time, handled_at, reg_result, ais_inn, ais_tax_organ_code, ais_doc_type_id, ais_doc_number, ais_doc_date, ais_register_time, ais_decision_date, ais_correct_date, ais_process_state_id, ais_reason_id, ais_reason_name, ais_on_uch_date, ais_off_uch_date, ais_cancel_id, ais_tsun_id, ais_reg_tax_organ_code, ais_reg_tax_organ_name)
VALUES(88836, 141111111111, 1000420590924, 'CANCEL_NON_COMPLIANCE', '2024-04-24 15:10:03.552', false, true, NULL, NULL, NULL, '141111111111', '2463', 2, '39701801', '2024-04-24', '2024-04-24 10:26:33.000', NULL, NULL, 6, NULL, NULL, '2024-04-24', NULL, 88835, 91919196, '2463', 'НО');
```

последнее изменение - даты постановки и снятия поменял
