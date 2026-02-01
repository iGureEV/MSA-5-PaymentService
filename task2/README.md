### <a name="_b7urdng99y53"></a>**Название задачи:** Применение State Machine на практике
### <a name="_hjk0fkfyohdk"></a>**Автор:** Гуреев Евгений
### <a name="_uanumrh8zrui"></a>**Дата:** январь 2026


Реестр транзакций (ядро):
- `CREATE_PAYMENT` (повторяемая)
- `RESERVE_FUNDS` (компенсируемая --> RELEASE_FUNDS)
- `DEBIT_FUNDS` (компенсируемая --> REFUND_FUNDS)
- `REQUEST_FRAUD_CHECK` (повторяемая)
- `PROCESS_MANUAL_REVIEW` (повторяемая)
- `CREDIT_MERCHANT` (необратимая) <-- PIVOT POINT
- `REFUND_FUNDS` (необратимая, компенсация)
- `RELEASE_FUNDS` (повторяемая, компенсация)
- `NOTIFY_PARTIES` (повторяемая)


## State Machine

| Исходное состояние | Переходное состояние | Событие |
|--------------------|----------------------|---------|
| NONE | CREATED | CREATE_PAYMENT_SUCCESS |
| CREATED | RESERVED | RESERVE_FUNDS_SUCCESS |
| CREATED | CANCELLED | CUSTOMER_CANCELLED |
| RESERVED | DEBITED | DEBIT_FUNDS_SUCCESS |
| RESERVED | CANCELLED | DEBIT_FUNDS_FAILED |
| DEBITED | FRAUD_CHECK | FRAUD_CHECK_STARTED |
| FRAUD_CHECK | APPROVED | FRAUD_CHECK_APPROVED |
| FRAUD_CHECK | DECLINED | FRAUD_CHECK_DECLINED |
| FRAUD_CHECK | MANUAL_REVIEW | FRAUD_CHECK_MANUAL |
| FRAUD_CHECK | REFUNDING | FRAUD_CHECK_TIMEOUT |
| MANUAL_REVIEW | APPROVED | MANUAL_REVIEW_APPROVED |
| MANUAL_REVIEW | DECLINED | MANUAL_REVIEW_DECLINED |
| MANUAL_REVIEW | APPROVED | MANUAL_REVIEW_TIMEOUT |
| APPROVED | TRANSFERRING | CREDIT_MERCHANT_STARTED |
| DECLINED | REFUNDING | REFUND_FUNDS_STARTED |
| TRANSFERRING | COMPLETED | CREDIT_MERCHANT_SUCCESS |
| TRANSFERRING | REFUNDING | CREDIT_MERCHANT_FAILED |
| REFUNDING | REFUNDED | REFUND_FUNDS_SUCCESS |
| REFUNDING | FAILED | REFUND_FUNDS_FAILED |
| CANCELLED | - (финал) | NOTIFY_PARTIES_SUCCESS |
| COMPLETED | - (финал) | NOTIFY_PARTIES_SUCCESS |
| REFUNDED | - (финал) | NOTIFY_PARTIES_SUCCESS |
| FAILED | - (финал) | SYSTEM_ALERT_SENT |

#### Основные состояния:

- `CREATED`: Платёж создан (`CREATE_PAYMENT`)
- `RESERVED`: Средства зарезервированы (`RESERVE_FUNDS`)
- `DEBITED`: Средства списаны (`DEBIT_FUNDS`)
- `FRAUD_CHECK`: Идёт антифрод проверка (`REQUEST_FRAUD_CHECK`)
- `MANUAL_REVIEW`: Ручная проверка оператором (`PROCESS_MANUAL_REVIEW`)
- `APPROVED`: Все проверки пройдены
- `DECLINED`: Проверки не пройдены
- `TRANSFERRING`: Перевод контрагенту (`CREDIT_MERCHANT`)
- `REFUNDING`: Возврат средств (`REFUND_FUNDS`)
- `COMPLETED`: Успешное завершение
- `REFUNDED`: Средства возвращены
- `CANCELLED`: Отмена до списания (`RELEASE_FUNDS`)
- `FAILED`: Технический сбой

### Ручная проверка с таймаутом:

```
DEBITED --> FRAUD_CHECK --> MANUAL_REVIEW
```

- При одобрении оператором: ```MANUAL_REVIEW_APPROVED --> APPROVED```
- При отклонении оператором: ```MANUAL_REVIEW_DECLINED --> DECLINED --> REFUNDING```
- При бездействии оператора 20 мин (таймаут): ```MANUAL_REVIEW_TIMEOUT --> APPROVED (cut-off)``

### Компенсационные пути:

1. Отказ после списания: ```FRAUD_CHECK_DECLINED --> DECLINED --> REFUNDING```
2. Сбой перевода: ```CREDIT_MERCHANT_FAILED --> REFUNDING```
3. Отмена до списания: ```CUSTOMER_CANCELLED --> CANCELLED```

### Точка невозврата:

Переход в `TRANSFERRING` начинается только после события `CREDIT_MERCHANT_STARTED`.  
После успешного завершения (`CREDIT_MERCHANT_SUCCESS`) возврат невозможен.

