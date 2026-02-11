## Проектирование GraphQL API
- текущее
  - схема API
    - GET /clients/{id} - Получить информацию о клиенте по ID
    - GET /clients/{id}/documents - Список документов клиента
    - GET /clients/{id}/relatives - Информация о родственниках клиента
  - модели
    - Client: id, name, age
    - Document: id, type, number, issueDate, expiryDate
    - Relative: id, relationType, name, age
  - проблемы
    - web, core-app запрашивают разные наборы данных
    - много атрибутов в ответе в карточке клиентов
    - приходится дергать несколько ресурсов в рамках одного процесса
      - получили клиента
      - запросили документы
      - запросили родтсвенников


## Запросы через graphQL
- гибкий выбор полей


- определены типы
  - Client
  - Document
  - Relative


- в типе Client связи c
  - documents: [Document!]!
  - relatives: [Relative!]!


- сделаны запросы по текущему REST API
  - согласно [as_is.yaml](as_is.yaml)
  - схема graphQL
    - [client-info.graphql](client-info.graphql)
  - client(id: ID!): Client
    - эквивалент GET `/clients/{id}`
    - в одном можно запросе запросить и клиента, и документы (documents, relatives)
  - clientDocuments(clientId: ID!): [Document!]!
      - эквивалент GET `/clients/{id}/documents`
  - clientRelatives(clientId: ID!): [Relative!]!
      - эквивалент GET `/clients/{id}/relatives`


- обоснование
  - клиент может запросить только нужные поля в одном запросе
    - клиента, документы, родственников
  - уменьшение RPS 


### Примеры запросов
- картояка клиента
```
query {
  client(id: "123") {
    id
    name
    age
  }
}
```


- клиент + доки
  - можно менять (запрашивать) разные поля как для клиента так и связанных полей
```
query {
  client(id: "123") {
    name
    documents {
      type
      number
    }
    relatives {
      relationType
      name
    }
  }
}
```

- запрос родственников
```
query {
  clientRelatives(clientId: "123") {
    id
    relationType
    name
    age
  }
}

```