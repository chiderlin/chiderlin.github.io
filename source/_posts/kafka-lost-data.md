---
title: kafka漏接訊息的問題?
date: 2022-07-31 13:40:02
tags:
    - kafka
categories: Message Queue
---

### 問題

接起來 Kafka 之後，Consumer 卻遇到有些資料會接收不到的情況。

### 解決

1. Producer Acknowledgements
   _argment: acks_

    -1 = 等所有的 comsumer server 接收到訊息並回覆確認
    0 = 不等
    1 = 等 leader 那台 comsumer 接收到訊息並回覆確認

    ```node
    async function audProcessor(config) {
        console.log('.........sending');
        return await producer.send({
            topic: KAFKA.TOPIC,
            messages: [{ value: JSON.stringify(config) }],
            // avoid kafka losing data
            acks: -1, //acks=all
        });
    }
    ```

2. Producer retries

    - 設定 `delivery.timeout.ms`
      _argment: transactionTimeout_
    - 設定 `retries`
      _argment: retry_

    ```node
    const producer = kafka.producer({
    ...
    // avoid kafka losing data
    transactionTimeout: 60000, // 1min
    retry: {
        retries: 10,
    },
    });

    ```

### 補充

**kafkajs max.in.flight.requests.per.connection 設定**
如果資料讀取的順序性對你的 application 很重要的話，要另外設定

_max.in.flight.requests.per.connection to 1_
**but not for my case**

# 資料來源

[Help, Kafka ate my data!](https://blog.softwaremill.com/help-kafka-ate-my-data-ae2e5d3e6576)
[Producing Messages](https://kafka.js.org/docs/producing)
[Does Kafka really guarantee the order of messages?](https://blog.softwaremill.com/does-kafka-really-guarantee-the-order-of-messages-3ca849fd19d2)
