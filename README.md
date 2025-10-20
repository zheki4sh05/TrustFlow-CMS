# **TrustFlow**

Проект связан с комплаенс‑контролем, управлением закупками, рисками, расследованиями и аналитикой. 
Цель проекта - упростить процесс мониторинга комплаенс-рисков в закупочной деятельности, чтобы сократить время выявления нарушений до 7 дней, обеспечить 100% охват ключевых процедур и снизить количество повторяющихся нарушений на 50%.

Ссылки на репозитории сервера и клиента:
Клиент: https://github.com/zheki4sh05/cms-system-frontend
Серверная часть: https://github.com/zheki4sh05/cms-system-backend

---

## **Содержание**

1. [Архитектура](#Архитектура)
	1. [C4-модель](#C4-модель)
	2. [Схема данных](#Схема_данных)
2. [Функциональные возможности](#Функциональные_возможности)
	1. [Диаграмма вариантов использования(#Диаграмма_вариантов_использования)]
	2. [User-flow диаграммы](#User-flow_диаграммы)
3. [Детали реализации](#Детали_реализации)
	1. [UML-диаграммы](#UML-диаграммы)
	2. [Спецификация API](#Спецификация_API)
	3. [Безопасность](#Безопасность)
	4. [Оценка качества кода](#Оценка_качества_кода)
4. [Тестирование](#Тестирование)
	1. [Unit-тесты](#Unit-тесты)
	2. [Интеграционные тесты](#Интеграционные_тесты)
5. [Установка и  запуск](#installation)
	1. [Манифесты для сборки docker образов](#Манифесты_для_сборки_docker_образов)
	2. [Манифесты для развертывания k8s кластера](#Манифесты_для_развертывания_k8s_кластера)
6. [Лицензия](#Лицензия)
7. [Контакты](#Контакты)

---
## **Архитектура**

### C4-модель

Иллюстрация и описание архитектура ПС

Ссылка на диаграммы: https://drive.google.com/drive/folders/1uWe9hcZbqnv3CznHv-AGVfdU6bdAXzq5
На контекстом уровне Менеджеры по закупкам: ведут случаи, отвечают на инциденты, подтверждают корректирующие действия. В свою очередь Руководитель закупок управляет исправлениями, контролирует инциденты, получает еженедельные отчеты. Топ‑менеджмент смотрит KPI и тренды, а также ежемесячные дашборды.    
На контейнерном уровне Monitoring Service имеет интеграцию с ERP/CRM/1С для осуществления пакетного сбора данных с целью проверки бизнес‑правил и генерации соответствующих инцидентов. Rules Service является централизованной нормативной базой и движком правил. Workflow Service выполняет управление случаями и расследованиями, корректирующими действиями, эскалациями, аудит‑трейл. Notification Service выполняет маршрутизацию и доставку уведомлений (email/in‑app), напоминания, еженедельные/месячные отчеты.   
На компонентном уровне изображен подход Clean Architecture. Данный подход к архитектуре будет использоваться и в других сервисах. Стрелки показывают направление зависимостей  от внешних слоёв к внутренним. Внешние слои (Infrastructure и WebComponent) зависят от внутренних, но не наоборот. Слой Core Component использует доменные объекты и сервисы, но не знает о деталях инфраструктуры. DDD наполняет внутренние слои (Entities, Value Objects, Aggregates, Domain Services) содержанием, отражающим бизнес-логику.  
Кодовый уровень описан на основе микросервиса WorkFlow.  
Микросервис Workflow Service предназначен для управления инцидентами комплаенс-рисков, расследованиями, корректирующими действиями и эскалациями. Он будет реализован как и другие микросервисы на Spring Boot, строго по принципам Clean Architecture, с доменной моделью, оформленной в стиле DDD. В корне проекта находится пакет com.example.workflow, который делится на три основных модуля: core, infrastructure и web. 
Модуль core бизнес-ядро. Модуль core содержит всю бизнес-логику, разделённую на три слоя: domain, application и port. Пакет domain реализует предметную модель системы. Здесь находятся агрегаты, сущности, value objects и доменные сервисы. Класс Incident представляет инцидент комплаенс-нарушения. Он хранит статус, критичность, SLA и историю изменений. Класс Case объединяет инциденты в единое расследование, управляет их жизненным циклом и фиксирует действия участников. Класс ActionPlan содержит корректирующие действия, представленные в виде задач с дедлайнами и ответственными. Класс VendorRiskProfile отражает риск-профиль поставщика, включая количество рекламаций и просрочек. Value Object Severity определяет уровень критичности инцидента, а SLA вычисляет срок реакции. ActionItem и TimelineEntry объекты-значения для задач и истории кейса. 
Пакет event содержит доменные события, такие как IncidentRaised, ActionDueSoon, CaseClosed, которые публикуются при изменении состояния агрегатов.
Пакет policy включает доменные сервисы, реализующие бизнес-правила, не принадлежащие конкретной сущности:
 AffiliationPolicyService проверяет контрагента на аффилированность.
 EscalationPolicyService определяет, когда инцидент должен быть эскалирован.
 VendorRiskPolicyService вычисляет статус риска поставщика.
Все классы в domain не зависят от Spring, JPA, Kafka или других технологий  это чистый Java-код, отражающий бизнес.
Пакет application реализует сценарии использования (use cases) и оркестрацию бизнес-логики. OpenIncidentUseCase создаёт инцидент, применяет политики и публикует событие. StartInvestigationUseCase переводит инцидент в расследование и назначает ответственного. CreateActionPlanUseCase формирует план действий по кейсу. VerifyRemediationUseCase подтверждает устранение нарушений и закрывает кейс. CloseCaseUseCase завершает кейс после верификации. ReassignOrEscalateUseCase перераспределяет или эскалирует инцидент по SLA. Пакет scheduler содержит планировщики:
 ReminderScheduler ежедневно проверяет задачи с приближающимся дедлайном;
 OverdueScheduler отслеживает просроченные задачи и инициирует эскалации;
 KpiReportScheduler агрегирует KPI и публикует отчёты.
Use cases используют интерфейсы из port, агрегаты из domain и публикуют события. Они не знают, как устроена инфраструктура — только бизнес-логика.
Пакет port содержит интерфейсы, которые реализуются в infrastructure.
IncidentRepository, CaseRepository, ActionPlanRepository, VendorRiskRepository контракты для доступа к данным. DomainEventPublisher  интерфейс публикации событий домена.
Таким образом, core полностью изолирован от технологий и может быть протестирован независимо.
Модуль web реализует REST-интерфейс для пользователей.
IncidentController отвечает за создание, просмотр, назначение инцидентов. CaseController в свою очередь предназначен для управления кейсами, комментарии, вложения. ActionPlanController же нужен для создания и верификация плана действий. IncidentDto, CaseDto, ActionPlanDto это все модели для передачи данных. IncidentInputDto, IncidentViewDto это все входные и выходные DTO. IncidentDtoMapper, CaseDtoMapper, ActionPlanDtoMapper выполняют преобразование между DTO и доменными объектами.    
	Таким образом, Clean Architecture реализована через строгое разделение слоёв: 
	domain находится в центре, не зависит ни от чего;
 use cases отвечают за оркестрацию, зависит от домена и портов;
 infrastructure реализует порты, зависит от домена;
 web это API сервиса, зависит от use cases и домена.
	DDD реализован через: 
 Агрегаты (Incident, Case, ActionPlan), которые управляют инвариантами и состоянием;
 Value Objects (Severity, SLA, ActionItem) неизменяемые бизнес-объекты;
 Domain Services (AffiliationPolicyService) логика, не принадлежащая сущностям;
 Domain Events отражают значимые изменения в бизнесе;
 Repository интерфейсы абстрагируют доступ к данным;
 Use Cases реализуют сценарии, не нарушая инварианты агрегатов.
	Таким образом, DDD позволяет выразить бизнес-логику в терминах предметной области, а Clean Architecture  изолировать её от технических деталей.


### Схема данных

Описание отношений и структур данных, используемых в ПС. Представлен скрипт (программный код), который необходим для генерации БД

Схема для всех сервисов: https://drive.google.com/drive/folders/1M62G38tBb4aeFgIsBe7L7UaXWzHaz0jJ

Скрипты:


 Monitoring Service schema

```

CREATE TYPE source_system AS ENUM ('1C');
create type event_flow_type as enum ('contract_signed');

CREATE TABLE procurement_event (
	id UUID PRIMARY KEY,
	source_system source_system NOT NULL,
	event_type event_flow_type NOT NULL,
	payload_json JSONB NOT NULL,
	occurred_at TIMESTAMP NOT NULL
);

CREATE TABLE rule_evaluation (
	id UUID PRIMARY KEY,
	event_id UUID NOT NULL REFERENCES procurement_event(id)
		ON DELETE CASCADE ON UPDATE CASCADE,
	rule_id UUID NOT NULL,
	result BOOLEAN NOT NULL,
	severity Integer NOT NULL,
	evaluated_at TIMESTAMP NOT NULL,
	CONSTRAINT uq_rule_eval UNIQUE (event_id, rule_id)
);

create type outbox_status as enum ('ACTIVE', 'DRAFT');

CREATE TABLE incident_outbox (
	id UUID PRIMARY KEY,
	event_id UUID NOT NULL REFERENCES procurement_event(id)
		ON DELETE CASCADE ON UPDATE CASCADE,
	incident_payload JSONB NOT NULL,
	status outbox_status NOT NULL,
	published_at TIMESTAMP
);

-- Indexes
CREATE INDEX idx_procurement_event_type ON procurement_event(event_type);
CREATE INDEX idx_procurement_event_date ON procurement_event(occurred_at);
CREATE INDEX idx_rule_eval_severity ON rule_evaluation(severity);
CREATE INDEX idx_incident_outbox_status ON incident_outbox(status);

```


-- Rules Service schema

```

CREATE TABLE requirement (
    id UUID PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    source VARCHAR(100) NOT NULL,
    area VARCHAR(50),
    created_at TIMESTAMP NOT NULL,
    CONSTRAINT uq_requirement_title_source UNIQUE (title, source)
);

CREATE TABLE control (
    id UUID PRIMARY KEY,
    requirement_id UUID NOT NULL REFERENCES requirement(id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT
);

create type rule_status as enum ('PENDING', 'SENT', 'FAILED');

CREATE TABLE rule (
    id UUID PRIMARY KEY,
    control_id UUID NOT NULL REFERENCES control(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    expression TEXT NOT NULL,
    version INTEGER NOT NULL,
    status rule_status  NOT NULL,
    activated_at TIMESTAMP,
    CONSTRAINT uq_rule_control_version UNIQUE (control_id, version)
);

create type process_code_type as enum ('PROCUREMENT_TENDER', 'PROCUREMENT_AUCTION', 'DIRECT_CONTRACT', 'CONTRACT_EXECUTION');

CREATE TABLE rule_process_binding (
    rule_id UUID NOT NULL REFERENCES rule(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    process_code process_code_type NOT NULL,
    PRIMARY KEY (rule_id, process_code)
);

CREATE TABLE rule_version_history (
    id UUID PRIMARY KEY,
    rule_id UUID NOT NULL REFERENCES rule(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    version INTEGER NOT NULL,
    change_log TEXT,
    changed_at TIMESTAMP NOT NULL
);

-- Indexes
CREATE INDEX idx_control_requirement ON control(requirement_id);
CREATE INDEX idx_rule_status ON rule(status);
CREATE INDEX idx_rule_version_history_rule ON rule_version_history(rule_id);
```



-- Workflow Service schema

```

create type incident_status as enum ('OPEN', 'RESOLVED'); 

CREATE TABLE incident (
    id UUID PRIMARY KEY,
    rule_id UUID NOT NULL,
    severity VARCHAR(20) NOT NULL,
    status incident_status NOT NULL,
    assigned_to UUID,
    sla_due_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL
);

create type case_status as enum ('OPEN', 'CLOSED');

CREATE TABLE workflow_case (
    id UUID PRIMARY KEY,
    owner_id UUID NOT NULL,
    status case_status NOT NULL,
    opened_at TIMESTAMP NOT NULL,
    closed_at TIMESTAMP
);

CREATE TABLE case_incident_link (
    case_id UUID NOT NULL REFERENCES workflow_case(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    incident_id UUID NOT NULL REFERENCES incident(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    PRIMARY KEY (case_id, incident_id)
);

create type action_plan_status as enum('OPEN', 'COMPLETED');

CREATE TABLE action_plan (
    id UUID PRIMARY KEY,
    case_id UUID NOT NULL REFERENCES workflow_case(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    status action_plan_status NOT NULL,
    verified_by UUID,
    verified_at TIMESTAMP
);

create type action_item_status as enum ('TODO', 'IN_PROGRESS', 'DONE');

CREATE TABLE action_item (
    id UUID PRIMARY KEY,
    action_plan_id UUID NOT NULL REFERENCES action_plan(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    title VARCHAR(100) NOT NULL,
    due_date DATE NOT NULL,
    status action_item_status NOT NULL
);

CREATE TABLE action_item_owner (
    action_item_id UUID NOT NULL REFERENCES action_item(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    owner_id UUID NOT NULL,
    PRIMARY KEY (action_item_id, owner_id)
);

CREATE TABLE action_item_evidence (
    action_item_id UUID NOT NULL REFERENCES action_item(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    evidence_url TEXT NOT NULL,
    PRIMARY KEY (action_item_id, evidence_url)
);

create type risk_status as enum ('NORMAL', 'WATCHLIST', 'HIGH_RISK', 'CRITICAL');

CREATE TABLE vendor_risk_profile (
    vendor_id UUID PRIMARY KEY,
    complaints_count INTEGER NOT NULL,
    delays_count INTEGER NOT NULL,
    risk_status risk_status NOT NULL,
    updated_at TIMESTAMP NOT NULL
);

create type outbox_event_type as enum('IncidentCreated', 'ActionItemOverdue', 'CaseClosed');
create type outbox_status as enum('PENDING','SENT','FAILED');

CREATE TABLE domain_event_outbox (
    id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    event_type outbox_event_type NOT NULL,
    payload_json JSONB NOT NULL,
    status outbox_status NOT NULL,
    published_at TIMESTAMP
);

-- Indexes
CREATE INDEX idx_incident_status ON incident(status);
CREATE INDEX idx_incident_sla ON incident(sla_due_at);
CREATE INDEX idx_incident_rule ON incident(rule_id);
CREATE INDEX idx_case_owner ON workflow_case(owner_id);
CREATE INDEX idx_action_plan_case ON action_plan(case_id);
CREATE INDEX idx_action_item_plan ON action_item(action_plan_id);
CREATE INDEX idx_vendor_risk_status ON vendor_risk_profile(risk_status);
CREATE INDEX idx_domain_event_outbox_status ON domain_event_outbox(status);

```


-- Notification Service schema
```

create type notification_status_type as enum ('PENDING', 'SENT', 'FAILED');

CREATE TABLE notification (
    id UUID PRIMARY KEY,
    type VARCHAR(50) NOT NULL,
    payload_json JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL,
    status notification_status_type NOT NULL
);

create type recipient_role as enum ('MANAGER', 'HEAD_OF_DEPARTMENT', 'EXECUTOR', 'SYSTEM_ADMIN');

CREATE TABLE recipient (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    role recipient_role NOT NULL,
    CONSTRAINT uq_recipient_user UNIQUE (user_id)
);

create type channel as enum ('EMAIL', 'APP', 'SMS');

CREATE TABLE recipient_channel (
    recipient_id UUID NOT NULL REFERENCES recipient(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    channel_type channel NOT NULL,
    address VARCHAR(100) NOT NULL,
    PRIMARY KEY (recipient_id, channel_type)
);

CREATE TABLE notification_recipient_link (
    notification_id UUID NOT NULL REFERENCES notification(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    recipient_id UUID NOT NULL REFERENCES recipient(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    PRIMARY KEY (notification_id, recipient_id)
);

CREATE TABLE template (
    id UUID PRIMARY KEY,
    type VARCHAR(50) NOT NULL,
    subject VARCHAR(100) NOT NULL,
    body_html TEXT NOT NULL,
    channel channel NOT NULL,
    CONSTRAINT uq_template_type_channel UNIQUE (type, channel)
);

create type delivery_status_type as enum ('PENDING', 'SENT', 'FAILED');

CREATE TABLE delivery_log (
    id UUID PRIMARY KEY,
    notification_id UUID NOT NULL REFERENCES notification(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    recipient_id UUID NOT NULL REFERENCES recipient(id)
        ON DELETE CASCADE ON UPDATE CASCADE,
    channel channel NOT NULL,
    status VARCHAR(20) NOT NULL,
    attempts INTEGER NOT NULL,
    sent_at TIMESTAMP
);

-- Indexes
CREATE INDEX idx_notification_status ON notification(status);
CREATE INDEX idx_recipient_channel_type ON recipient_channel(channel_type);
CREATE INDEX idx_delivery_log_status ON delivery_log(status);
CREATE INDEX idx_delivery_log_recipient ON delivery_log(recipient_id);
```
---

## **Функциональные возможности**

### Диаграмма вариантов использования

Диаграмма вариантов использования и ее описание

### User-flow диаграммы

Описание переходов между части ПС для всех ролей из диаграммы ВИ (название ролей должны совпадать с тем, что указано на c4-модели и диаграмме вариантов использования)


---

## **Детали реализации**

### UML-диаграммы

Представить все UML-диаграммы , которые позволят более точно понять структуру и детали реализации ПС

### Спецификация API

Представить описание реализованных функциональных возможностей ПС с использованием Open API (можно представить либо полный файл спецификации, либо ссылку на него)

### Безопасность

Описать подходы, использованные для обеспечения безопасности, включая описание процессов аутентификации и авторизации с примерами кода из репозитория сервера

### Оценка качества кода

Используя показатели качества и метрики кода, оценить его качество

---

## **Тестирование**

### Unit-тесты

Представить код тестов для пяти методов и его пояснение

### Интеграционные тесты

Представить код тестов и его пояснение

---

## **Установка и  запуск**

### Манифесты для сборки docker образов

Представить весь код манифестов или ссылки на файлы с ними (при необходимости снабдить комментариями)

### Манифесты для развертывания k8s кластера

Представить весь код манифестов или ссылки на файлы с ними (при необходимости снабдить комментариями)

---

## **Лицензия**

Этот проект лицензирован по лицензии MIT - подробности представлены в файле [[License.md|LICENSE.md]]

---

## **Контакты**

Автор: e.shostak05@gmail.com