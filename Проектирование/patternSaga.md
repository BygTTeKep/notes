# Saga pattern

Патерн Saga применятеся в приложения с микросервисной архитектурой.
Проблема которая возникает в таких приложения и какую решает этот патерн заключается в том в, что в микросервисной архитектуре возникают проблемы с управлением транзакциями.

## Суть патерна

Суть патерна заключается в том чтобы разбить большие транзакции на ряд более мелких, локальных транзакций внутри каждого микросервиса.

Saga сосредотачивается на достижении атомарности(**A**CID) каждого шага транзакции.
Атомарность говорит о том что транзакция должна выполниться полностью, тоесть успешно, либо полностью откатиться.

## Реализация

1. Определение границ сервисов и определение границ саги
2. Разделение саги на локальные транзации
3. Обработка компенсирующих действий при ошибки

```
const axios = require('axios');

// Координатор Saga
class SagaCoordinator {
    constructor() {
        this.steps = [];
    }

    addStep(action, compensation) {
        this.steps.push({ action, compensation });
    }

    async executeSaga() {
        const completedSteps = [];
        try {
            for (const step of this.steps) {
                console.log(`Executing action: ${step.action.name}`);
                await step.action();
                completedSteps.push(step);
            }
            console.log('Saga completed successfully.');
        } catch (error) {
            console.error('Error occurred, compensating...');
            for (const step of completedSteps.reverse()) {
                console.log(`Executing compensation: ${step.compensation.name}`);
                try {
                    await step.compensation();
                } catch (compError) {
                    console.error('Error during compensation:', compError.message);
                }
            }
        }
    }
}

// Пример работы с API
async function orderProduct() {
    await axios.post('http://service1/api/order', { productId: 1, quantity: 2 });
    console.log('Product ordered.');
}

async function cancelOrder() {
    await axios.post('http://service1/api/cancelOrder', { productId: 1 });
    console.log('Order canceled.');
}

async function reserveInventory() {
    await axios.post('http://service2/api/reserve', { productId: 1, quantity: 2 });
    console.log('Inventory reserved.');
}

async function releaseInventory() {
    await axios.post('http://service2/api/release', { productId: 1 });
    console.log('Inventory released.');
}

async function processPayment() {
    await axios.post('http://service3/api/pay', { amount: 100 });
    console.log('Payment processed.');
}

async function refundPayment() {
    await axios.post('http://service3/api/refund', { amount: 100 });
    console.log('Payment refunded.');
}

// Создаем Saga
const saga = new SagaCoordinator();

// Добавляем шаги
saga.addStep(orderProduct, cancelOrder);
saga.addStep(reserveInventory, releaseInventory);
saga.addStep(processPayment, refundPayment);

// Запускаем Saga
saga.executeSaga();

```

## Инструменты и подходы для реализации

Координаторы транзакций - Это ключевой элемент при реализации патерна saga. Они отвечают за управление выполнением каждого шага саги, отслеживание его статуса и реагирование на возможные ошибки.

Координатор выполняет роль аркистратора, который согласовывает действия различных сервисов, чтобы обеспечить атомарность и целостность транзакций.

В микросервисной архитектуре координатор может быть реализован как отдельный сервис, который общается с другими микросервисами по API.

Когда Saga запускается координатор отправляет запросы на выполнение каждого шага и следит за успешным завершением. В случае ошибки, он активирует компенсирующие действие

## Брокеры сообщений и Saga

```
const { Kafka } = require('kafkajs');

// Настройка Kafka
const kafka = new Kafka({
    clientId: 'saga-coordinator',
    brokers: ['localhost:9092']
});

const producer = kafka.producer();
const consumer = kafka.consumer({ groupId: 'saga-group' });

async function sendMessage(topic, message) {
    await producer.send({
        topic,
        messages: [{ value: JSON.stringify(message) }]
    });
    console.log(`Message sent to topic ${topic}:`, message);
}

// Saga Coordinator
class KafkaSagaCoordinator {
    constructor() {
        this.steps = [];
        this.responses = {};
    }

    addStep(commandTopic, compensationTopic, successTopic, failureTopic) {
        this.steps.push({ commandTopic, compensationTopic, successTopic, failureTopic });
    }

    async executeSaga() {
        await producer.connect();
        await consumer.connect();

        try {
            for (const step of this.steps) {
                console.log(`Sending command to topic: ${step.commandTopic}`);
                await sendMessage(step.commandTopic, { action: 'execute', data: {} });

                const result = await this.waitForResponse(step.successTopic, step.failureTopic);
                if (!result.success) {
                    throw new Error('Step failed');
                }
            }
            console.log('Saga completed successfully.');
        } catch (error) {
            console.error('Saga failed, starting compensation...');
            for (const step of this.steps.reverse()) {
                console.log(`Sending compensation to topic: ${step.compensationTopic}`);
                await sendMessage(step.compensationTopic, { action: 'compensate', data: {} });
            }
        } finally {
            await producer.disconnect();
            await consumer.disconnect();
        }
    }

    async waitForResponse(successTopic, failureTopic) {
        return new Promise((resolve) => {
            const onMessage = async ({ topic, message }) => {
                const parsedMessage = JSON.parse(message.value.toString());
                if (topic === successTopic) {
                    consumer.off('message', onMessage);
                    resolve({ success: true, data: parsedMessage });
                } else if (topic === failureTopic) {
                    consumer.off('message', onMessage);
                    resolve({ success: false, data: parsedMessage });
                }
            };

            consumer.on('message', onMessage);
        });
    }
}

// Настройка топиков и шагов
(async () => {
    const saga = new KafkaSagaCoordinator();

    saga.addStep(
        'order-command', // Команда на заказ
        'order-compensate', // Откат заказа
        'order-success', // Успех заказа
        'order-failure' // Ошибка заказа
    );

    saga.addStep(
        'inventory-command', // Команда на резерв
        'inventory-compensate', // Откат резерва
        'inventory-success', // Успех резерва
        'inventory-failure' // Ошибка резерва
    );

    saga.addStep(
        'payment-command', // Команда на оплату
        'payment-compensate', // Откат оплаты
        'payment-success', // Успех оплаты
        'payment-failure' // Ошибка оплаты
    );

    await saga.executeSaga();
})();

```
