[[_TOC_]]


# Saga模型中寻找对应saga数据

### Saga模型配置寻找对应数据

```csharp
public class SagaOrderProcessManager : Saga<OrderSagaData>,
        IAmStartedByMessages<OrderCreateEventData>,
        IHandleMessages<SeatsReservedEventData>,
        IHandleMessages<PaymentAcceptedEventData>,
        IHandleMessages<SeatsNotReservationEventData>
    {
        // 此继承方法是确保saga模型找到对应的saga数据和对应的事件数据     
        protected override void ConfigureHowToFindSaga(
            SagaPropertyMapper<OrderSagaData> mapper)
        {
            //每一个事件对应的OrderId都应该和SagaData的OrderId 关键字对应
            mapper.ConfigureMapping<OrderCreateEventData>(e => e.OrderId)
                .ToSaga(sagadata => sagadata.OrderId);
            mapper.ConfigureMapping<SeatsReservedEventData>(e => e.OrderId)
                .ToSaga(sagadata => sagadata.OrderId);
            mapper.ConfigureMapping<PaymentAcceptedEventData>(e => e.OrderId)
                .ToSaga(sagadata => sagadata.OrderId);
            mapper.ConfigureMapping<SeatsNotReservationEventData>(e => e.OrderId)
                .ToSaga(sagadata => sagadata.OrderId);
        }
}
```

### 不只一个关键字，多关键字判断对应SagaData

```csharp
  protected override void ConfigureHowToFindSaga(SagaPropertyMapper<MySagaData> mapper)
{
    //官网建议，多个关键字拼接成一个关键字，用一个关键字对应一个Saga模型数据
    //逻辑简单，并且寻找快速
    mapper.ConfigureMapping<MyMessage>(message => $"{message.Part1}_{message.Part2}")
        .ToSaga(sagaData => sagaData.SomeId);
}
```

>If correlating on more than one saga property is necessary, or matched properties are of different types, use a [custom saga finder](https://docs.particular.net/nservicebus/sagas/saga-finding).
It is possible to specify the mapping to the message using expressions if the correlation information is split between multiple fields.

### 使用自定义查找接口
>不到万不得已，不推荐，需要考虑不同的数据持久化和事务等问题
```csharp
public class MySagaFinder :
    IFindSagas<MySagaData>.Using<MyMessage>
{
    public Task<MySagaData> FindBy(MyMessage message, SynchronizedStorageSession storageSession, ReadOnlyContextBag context)
    {
        // SynchronizedStorageSession will have a persistence specific extension method
        // For example GetDbSession is a stub extension method
        var dbSession = storageSession.GetDbSession();
        return dbSession.GetSagaFromDB(message.SomeId, message.SomeData);
        // If a saga can't be found Task.FromResult(null) should be returned
    }
}
```
&emsp;&emsp;此方法，需要注意并发和事务等问题。参考[saga.finder](https://docs.particular.net/persistence/sql/saga-finder)。多种saga.finder的方法：[SQL Persistence Saga Finding Logic](https://docs.particular.net/samples/saga/sql-sagafinder/)
&emsp;&emsp;**此方法针对的是特指的单独消息事件进行的**，默认查找将按照配置`ConfigureHowToFindSaga`的关键字方法查找 ,制定的**Using中的消息**才是按照自定义接口查找。定义了自定义查询接口后，不需要设置其他，系统能够自己查找初始化。

**例子：**
```csharp
//自定义的sqlserver的特殊属性查找，这里用了Using的接口，泛型制定了CompletePaymentTransaction消息才是走此查找
class OrderSagaFinder :
    IFindSagas<OrderSagaData>.Using<CompletePaymentTransaction>
{
    public Task<OrderSagaData> FindBy(CompletePaymentTransaction message, SynchronizedStorageSession session, ReadOnlyContextBag context)
    {
        return session.GetSagaData<OrderSagaData>(
            context: context,
            whereClause: "JSON_VALUE(Data,'$.PaymentTransactionId') = @propertyValue",
              // 这里可以加入多种查询条件
            appendParameters: (builder, append) =>
            {
                var parameter = builder();
                parameter.ParameterName = "propertyValue";
                parameter.Value = message.PaymentTransactionId;
                append(parameter);
            });
    }
}

//消息数据：
public class CompletePaymentTransaction :
    IMessage
{
    public string PaymentTransactionId { get; set; }
}

//Saga模型
public class OrderSaga :
    Saga<OrderSagaData>,
    IAmStartedByMessages<StartOrder>,
    IHandleMessages<CompletePaymentTransaction>,
    IHandleMessages<CompleteOrder>
{
    static ILog log = LogManager.GetLogger<OrderSaga>();
    // 默认还是按照OrderId进行查找，只有CompletePaymentTransaction消息才是按照自定义查找
    protected override void ConfigureHowToFindSaga(SagaPropertyMapper<OrderSagaData> mapper)
    {
        mapper.ConfigureMapping<StartOrder>(msg => msg.OrderId).ToSaga(saga => saga.OrderId);
        mapper.ConfigureMapping<CompleteOrder>(msg => msg.OrderId).ToSaga(saga => saga.OrderId);
    }

    public Task Handle(StartOrder message, IMessageHandlerContext context)
    {
        Data.PaymentTransactionId = Guid.NewGuid().ToString();

        log.Info($"Saga with OrderId {Data.OrderId} received StartOrder with OrderId {message.OrderId}");
        var issuePaymentRequest = new IssuePaymentRequest
        {
            PaymentTransactionId = Data.PaymentTransactionId
        };
        return context.SendLocal(issuePaymentRequest);
    }

    public Task Handle(CompletePaymentTransaction message, IMessageHandlerContext context)
    {
        log.Info($"Transaction with Id {Data.PaymentTransactionId} completed for order id {Data.OrderId}");
        var completeOrder = new CompleteOrder
        {
            OrderId = Data.OrderId
        };
        return context.SendLocal(completeOrder);
    }

    public Task Handle(CompleteOrder message, IMessageHandlerContext context)
    {
        log.Info($"Saga with OrderId {Data.OrderId} received CompleteOrder with OrderId {message.OrderId}");
        MarkAsComplete();
        return Task.CompletedTask;
    }
}

```

>When using custom saga finders, users are expected to configure any additional indexes needed to handle concurrent access to saga instances properly using the tooling of the selected storage engine. Due to this constraint, persisters are not all able to support custom saga finders to the same degree.