### <a name="_b7urdng99y53"></a>**Название задачи:** Формирование BPM-процесса
### <a name="_hjk0fkfyohdk"></a>**Автор:** Гуреев Евгений
### <a name="_uanumrh8zrui"></a>**Дата:** январь 2026

На основании [шагов Saga](../task1/README.md), [Применения State Machine](../task2/README.md) и [выбора размещения функции оркестрации платежного процесса](../task3/README.md) сформирована следующая схема: [orchestrpay-payment-process.bpmn](./orchestrpay-payment-process.bpmn)

![Логотип проекта](orchestrpay-payment-process.png)

Представление в Mermaid:
```mermaid
flowchart TD
    Start["Payment Initiated"] --> CreatePayment["CREATE_PAYMENT"]
    
    CreatePayment --> ReserveFunds["RESERVE_FUNDS"]
    
    ReserveFunds --> ParallelStart["Parallel Start"]
    
    ParallelStart --> DebitFunds["DEBIT_FUNDS"]
    ParallelStart --> FraudCheck["REQUEST_FRAUD_CHECK"]
    
    DebitFunds --> ParallelJoin["Parallel Join"]
    FraudCheck --> ParallelJoin
    
    ParallelJoin --> FraudDecision{"Fraud Decision?"}
    
    %% PIVOT POINT 1 Annotation
    DebitFunds -. "Опорная точка 1:<br/>Резервирование средств" .-> PivotNote1
    
    %% Fraud Decision Paths
    FraudDecision -->|"ALLOW<br/>(fraudDecision == 'ALLOW')"| MarkApproved["MARK_APPROVED"]
    FraudDecision -->|"MANUAL_REVIEW<br/>(fraudDecision == 'MANUAL_REVIEW')"| MarkManualReview["MARK_MANUAL_REVIEW"]
    FraudDecision -->|"DENY<br/>(fraudDecision == 'DENY')"| InitiateRefund["INITIATE_REFUND"]
    
    %% Manual Review Process
    MarkManualReview --> ManualReview["Manual Fraud Review<br/>(User Task)"]
    
    ManualReview --> ManualDecision{"Manual Decision?"}
    
    %% Boundary Timer for Manual Review
    ManualReview -. "20 минутный таймер" .-> AutoApprove["AUTO_APPROVE<br/>(Cut-off)"]
    AutoApprove --> ManualDecision
    
    ManualDecision -->|"APPROVE<br/>(manualDecision == 'APPROVE')"| MarkApproved
    ManualDecision -->|"DENY<br/>(manualDecision == 'DENY')"| InitiateRefund
    
    %% Success Path
    MarkApproved --> CreditMerchant["CREDIT_MERCHANT"]
    
    %% PIVOT POINT 2 Annotation
    CreditMerchant -. "Опорная точка 2:<br/>Необратимый перевод" .-> PivotNote2
    
    CreditMerchant --> CreditResult{"Credit Result?"}
    
    CreditResult -->|"SUCCESS<br/>(creditSuccessful == true)"| MarkCompleted["MARK_COMPLETED"]
    CreditResult -->|"FAIL<br/>(creditSuccessful == false)"| InitiateRefund
    
    %% Success Completion
    MarkCompleted --> NotifyParallel["Parallel Notifications"]
    
    NotifyParallel --> NotifyCustomer["NOTIFY_CUSTOMER"]
    NotifyParallel --> PublishEvent["PUBLISH_EVENT"]
    
    NotifyCustomer --> NotifyJoin["Join Notifications"]
    PublishEvent --> NotifyJoin
    
    NotifyJoin --> EndSuccess["Payment Completed"]
    
    %% Refund Process (Compensation)
    InitiateRefund --> RefundFunds["REFUND_FUNDS"]
    
    RefundFunds --> RefundResult{"Refund Result?"}
    
    RefundResult -->|"SUCCESS<br/>(refundSuccessful == true)"| MarkRefunded["MARK_REFUNDED"]
    RefundResult -->|"RETRY<br/>(refundSuccessful == false && retryCount < 3)"| RetryRefund["RETRY_REFUND"]
    RefundResult -->|"FAIL<br/>(refundSuccessful == false && retryCount >= 3)"| MarkFailed["MARK_FAILED"]
    
    %% Retry Loop
    RetryRefund --> RetryDelay["Retry Delay (30s)"]
    RetryDelay --> RefundFunds
    
    %% Failed Refund Path
    MarkFailed --> AlertOperator["ALERT_OPERATOR"]
    AlertOperator --> EndFailed["Payment Failed"]
    
    %% Successful Refund Completion
    MarkRefunded --> RefundNotifyParallel["Parallel Refund Notifications"]
    
    RefundNotifyParallel --> NotifyRefunded["NOTIFY_REFUNDED"]
    RefundNotifyParallel --> NotifySecurity["NOTIFY_SECURITY"]
    RefundNotifyParallel --> PublishRefund["PUBLISH_REFUND_EVENT"]
    
    NotifyRefunded --> RefundNotifyJoin["Join Refund Notifications"]
    NotifySecurity --> RefundNotifyJoin
    PublishRefund --> RefundNotifyJoin
    
    RefundNotifyJoin --> EndRefunded["Payment Refunded"]
```
