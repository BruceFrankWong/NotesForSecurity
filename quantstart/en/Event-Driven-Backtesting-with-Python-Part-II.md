# Event-Driven Backtesting with Python - Part II

   *Source: [QuantStart](https://www.quantstart.com/articles/Event-Driven-Backtesting-with-Python-Part-II/)*
   
   *中文：[用Python进行事件驱动的回测-第2部分](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/zh/用Python进行事件驱动的回测-第2部分.md)*

In the [last article](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-I.md) we described the concept of an event-driven backtester. The remainder of this series of articles will concentrate on each of the separate class hierarchies that make up the overall system. In this article we will consider **Events** and how they can be used to communicate information between objects.

As discussed in the previous article the trading system utilises two while loops - an outer and an inner. The inner while loop handles capture of events from an in-memory queue, which are then routed to the appropriate component for subsequent action. In this infrastucture there are four types of events:

+ **MarketEvent** - This is triggered when the outer while loop begins a new "heartbeat". It occurs when the ***DataHandler*** object receives a new update of market data for any symbols which are currently being tracked. It is used to trigger the ***Strategy*** object generating new trading signals. The event object simply contains an identification that it is a market event, with no other structure.

+ **SignalEvent** - The ***Strategy*** object utilises market data to create new ***SignalEvents***. The ***SignalEvent*** contains a ticker symbol, a timestamp for when it was generated and a direction (long or short). The ***SignalEvents*** are utilised by the ***Portfolio*** object as advice for how to trade.

+ **OrderEvent** - When a ***Portfolio*** object receives ***SignalEvents*** it assesses them in the wider context of the portfolio, in terms of risk and position sizing. This ultimately leads to ***OrderEvents*** that will be sent to an ***ExecutionHandler***.

+ **FillEvent** - When an ***ExecutionHandler*** receives an ***OrderEvent*** it must transact the order. Once an order has been transacted it generates a ***FillEvent***, which describes the cost of purchase or sale as well as the transaction costs, such as fees or slippage.

The parent class is called ***Event***. It is a base class and does not provide any functionality or specific interface. In later implementations the Event objects will likely develop greater complexity and thus we are future-proofing the design of such systems by creating a class hierarchy.

```
# event.py

class Event(object):
    """
    Event is base class providing an interface for all subsequent 
    (inherited) events, that will trigger further events in the 
    trading infrastructure.   
    """
    pass
```

The ***MarketEvent*** inherits from ***Event*** and provides little more than a self-identification that it is a 'MARKET' type event.

```
# event.py

class MarketEvent(Event):
    """
    Handles the event of receiving a new market update with 
    corresponding bars.
    """

    def __init__(self):
        """
        Initialises the MarketEvent.
        """
        self.type = 'MARKET'
```

A ***SignalEvent*** requires a ticker symbol, a timestamp for generation and a direction in order to advise a ***Portfolio*** object.

```
# event.py

class SignalEvent(Event):
    """
    Handles the event of sending a Signal from a Strategy object.
    This is received by a Portfolio object and acted upon.
    """
    
    def __init__(self, symbol, datetime, signal_type):
        """
        Initialises the SignalEvent.

        Parameters:
        symbol - The ticker symbol, e.g. 'GOOG'.
        datetime - The timestamp at which the signal was generated.
        signal_type - 'LONG' or 'SHORT'.
        """
        
        self.type = 'SIGNAL'
        self.symbol = symbol
        self.datetime = datetime
        self.signal_type = signal_type
```

The ***OrderEvent*** is slightly more complex than a ***SignalEvent*** since it contains a quantity field in addition to the aforementioned properties of ***SignalEvent***. The quantity is determined by the ***Portfolio*** constraints. In addition the ***OrderEvent*** has a ***print_order()*** method, used to output the information to the console if necessary.

```
# event.py

class OrderEvent(Event):
    """
    Handles the event of sending an Order to an execution system.
    The order contains a symbol (e.g. GOOG), a type (market or limit),
    quantity and a direction.
    """

    def __init__(self, symbol, order_type, quantity, direction):
        """
        Initialises the order type, setting whether it is
        a Market order ('MKT') or Limit order ('LMT'), has
        a quantity (integral) and its direction ('BUY' or
        'SELL').

        Parameters:
        symbol - The instrument to trade.
        order_type - 'MKT' or 'LMT' for Market or Limit.
        quantity - Non-negative integer for quantity.
        direction - 'BUY' or 'SELL' for long or short.
        """
        
        self.type = 'ORDER'
        self.symbol = symbol
        self.order_type = order_type
        self.quantity = quantity
        self.direction = direction

    def print_order(self):
        """
        Outputs the values within the Order.
        """
        print "Order: Symbol=%s, Type=%s, Quantity=%s, Direction=%s" % \
            (self.symbol, self.order_type, self.quantity, self.direction)
```

The ***FillEvent*** is the ***Event*** with the greatest complexity. It contains a timestamp for when an order was filled, the symbol of the order and the exchange it was executed on, the quantity of shares transacted, the actual price of the purchase and the commission incurred.

The commission is calculated using the [Interactive Brokers commissions](https://www.interactivebrokers.com/en/index.php?f=commission&p=stocks2). For US API orders this commission is 1.30 USD minimum per order, with a flat rate of either 0.013 USD or 0.08 USD per share depending upon whether the trade size is below or above 500 units of stock.

```
# event.py

class FillEvent(Event):
    """
    Encapsulates the notion of a Filled Order, as returned
    from a brokerage. Stores the quantity of an instrument
    actually filled and at what price. In addition, stores
    the commission of the trade from the brokerage.
    """

    def __init__(self, timeindex, symbol, exchange, quantity, 
                 direction, fill_cost, commission=None):
        """
        Initialises the FillEvent object. Sets the symbol, exchange,
        quantity, direction, cost of fill and an optional 
        commission.

        If commission is not provided, the Fill object will
        calculate it based on the trade size and Interactive
        Brokers fees.

        Parameters:
        timeindex - The bar-resolution when the order was filled.
        symbol - The instrument which was filled.
        exchange - The exchange where the order was filled.
        quantity - The filled quantity.
        direction - The direction of fill ('BUY' or 'SELL')
        fill_cost - The holdings value in dollars.
        commission - An optional commission sent from IB.
        """
        
        self.type = 'FILL'
        self.timeindex = timeindex
        self.symbol = symbol
        self.exchange = exchange
        self.quantity = quantity
        self.direction = direction
        self.fill_cost = fill_cost

        # Calculate commission
        if commission is None:
            self.commission = self.calculate_ib_commission()
        else:
            self.commission = commission

    def calculate_ib_commission(self):
        """
        Calculates the fees of trading based on an Interactive
        Brokers fee structure for API, in USD.

        This does not include exchange or ECN fees.

        Based on "US API Directed Orders":
        https://www.interactivebrokers.com/en/index.php?f=commission&p=stocks2
        """
        full_cost = 1.3
        if self.quantity <= 500:
            full_cost = max(1.3, 0.013 * self.quantity)
        else: # Greater than 500
            full_cost = max(1.3, 0.008 * self.quantity)
        full_cost = min(full_cost, 0.5 / 100.0 * self.quantity * self.fill_cost)
        return full_cost
```

In the [next article](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-III.md) of the series we are going to consider how to develop a market ***DataHandler*** class hierarchy that allows both historic backtesting and live trading, via the same class interface.