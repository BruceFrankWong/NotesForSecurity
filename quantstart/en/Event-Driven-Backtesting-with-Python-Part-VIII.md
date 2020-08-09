# Event-Driven Backtesting with Python - Part VIII

   *Source: [QuantStart](https://www.quantstart.com/articles/Event-Driven-Backtesting-with-Python-Part-VIII/)*
   
   *中文：[用Python进行事件驱动的回测-第8部分](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/zh/用Python进行事件驱动的回测-第8部分.md)*

It's been a while since we've considered the event-driven backtester, which we began discussing in [this article](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-I.md). In [Part VI](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-VI.md) I described how to code a stand-in ***ExecutionHandler*** model that worked for a historical backtesting situation. In this article we are going to code the corresponding Interactive Brokers API handler in order to move towards a live trading system.

I've previously discussed how to [download Trader Workstation and create an Interactive Brokers demo account](http://www.quantstart.com/articles/Interactive-Brokers-Demo-Account-Signup-Tutorial) as well as how to [create a basic interface to the IB API using IbPy](http://www.quantstart.com/articles/Using-Python-IBPy-and-the-Interactive-Brokers-API-to-Automate-Trades). This article will wrap up the basic IbPy interface into the event-driven system, so that when it is paired with a live market feed, it will form the basis for an automated execution system.

The essential idea of the ***IBExecutionHandler*** class (see below) is to receive ***OrderEvent*** instances from the events queue and then to execute them directly against the Interactive Brokers order API using the IbPy library. The class will also handle the "Server Response" messages sent back via the API. At this stage, the only action taken will be to create corresponding ***FillEvent*** instances that will then be sent back to the events queue.

The class itself could feasibly become rather complex, with execution optimisation logic as well as sophisticated error handling. However, I have opted to keep it relatively simple so that you can see the main ideas and extend it in the direction that suits your particular trading style.

## Python Implementation

As always, the first task is to create the Python file and import the necessary libraries. The file is called ***ib_execution.py*** and lives in the same directory as the other event-driven files.

We import the necessary date/time handling libraries, the IbPy objects and the specific Event objects that are handled by ***IBExecutionHandler***:

```
# ib_execution.py

import datetime
import time

from ib.ext.Contract import Contract
from ib.ext.Order import Order
from ib.opt import ibConnection, message

from event import FillEvent, OrderEvent
from execution import ExecutionHandler
```

We now define the ***IBExecutionHandler*** class. The ***__init__*** constructor firstly requires knowledge of the ***events*** queue. It also requires specification of ***order_routing***, which I've defaulted to "SMART". If you have specific exchange requirements, you can specify them here. The default ***currency*** has also been set to US Dollars.

Within the method we create a ***fill_dict*** dictionary, needed later for usage in generating ***FillEvent*** instances. We also create a ***tws_conn*** connection object to store our connection information to the Interactive Brokers API. We also have to create an initial default ***order_id***, which keeps track of all subsequent orders to avoid duplicates. Finally we register the message handlers (which we'll define in more detail below):

```
# ib_execution.py

class IBExecutionHandler(ExecutionHandler):
    """
    Handles order execution via the Interactive Brokers
    API, for use against accounts when trading live
    directly.
    """

    def __init__(self, events, 
                 order_routing="SMART", 
                 currency="USD"):
        """
        Initialises the IBExecutionHandler instance.
        """
        self.events = events
        self.order_routing = order_routing
        self.currency = currency
        self.fill_dict = {}

        self.tws_conn = self.create_tws_connection()
        self.order_id = self.create_initial_order_id()
        self.register_handlers()
```

The IB API utilises a message-based event system that allows our class to respond in particular ways to certain messages, in a similar manner to the event-driven backtester itself. I've not included any real error handling (for the purposes of brevity), beyond output to the terminal, via the ***_error_handler*** method.

The ***_reply_handler*** method, on the other hand, is used to determine if a ***FillEvent*** instance needs to be created. The method asks if an "openOrder" message has been received and checks whether an entry in our ***fill_dict*** for this particular orderId has already been set. If not then one is created.

If it sees an "orderStatus" message and that particular message states than an order has been filled, then it calls ***create_fill*** to create a ***FillEvent***. It also outputs the message to the terminal for logging/debug purposes:

```
# ib_execution.py
    
    def _error_handler(self, msg):
        """
        Handles the capturing of error messages
        """
        # Currently no error handling.
        print "Server Error: %s" % msg

    def _reply_handler(self, msg):
        """
        Handles of server replies
        """
        # Handle open order orderId processing
        if msg.typeName == "openOrder" and \
            msg.orderId == self.order_id and \
            not self.fill_dict.has_key(msg.orderId):
            self.create_fill_dict_entry(msg)
        # Handle Fills
        if msg.typeName == "orderStatus" and \
            msg.status == "Filled" and \
            self.fill_dict[msg.orderId]["filled"] == False:
            self.create_fill(msg)      
        print "Server Response: %s, %s\n" % (msg.typeName, msg)
```

The following method, ***create_tws_connection***, creates a connection to the IB API using the IbPy ***ibConnection*** object. It uses a default port of 7496 and a default clientId of 10. Once the object is created, the ***connect*** method is called to perform the connection:

```
# ib_execution.py
    
    def create_tws_connection(self):
        """
        Connect to the Trader Workstation (TWS) running on the
        usual port of 7496, with a clientId of 10.
        The clientId is chosen by us and we will need 
        separate IDs for both the execution connection and
        market data connection, if the latter is used elsewhere.
        """
        tws_conn = ibConnection()
        tws_conn.connect()
        return tws_conn
```

To keep track of separate orders (for the purposes of tracking fills) the following method ***create_initial_order_id*** is used. I've defaulted it to "1", but a more sophisticated approach would be th query IB for the latest available ID and use that. You can always reset the current API order ID via the Trader Workstation > Global Configuration > API Settings panel:

```
# ib_execution.py
    
    def create_initial_order_id(self):
        """
        Creates the initial order ID used for Interactive
        Brokers to keep track of submitted orders.
        """
        # There is scope for more logic here, but we
        # will use "1" as the default for now.
        return 1
```

The following method, ***register_handlers***, simply registers the error and reply handler methods defined above with the TWS connection:

```
# ib_execution.py
    
    def register_handlers(self):
        """
        Register the error and server reply 
        message handling functions.
        """
        # Assign the error handling function defined above
        # to the TWS connection
        self.tws_conn.register(self._error_handler, 'Error')

        # Assign all of the server reply messages to the
        # reply_handler function defined above
        self.tws_conn.registerAll(self._reply_handler)
```

As with the [previous tutorial](http://www.quantstart.com/articles/Using-Python-IBPy-and-the-Interactive-Brokers-API-to-Automate-Trades) on using IbPy we need to create a ***Contract*** instance and then pair it with an ***Order*** instance, which will be sent to the IB API. The following method, ***create_contract***, generates the first component of this pair. It expects a ticker symbol, a security type (e.g. stock or future), an exchange/primary exchange and a currency. It returns the ***Contract*** instance:

```
# ib_execution.py
    
    def create_contract(self, symbol, sec_type, exch, prim_exch, curr):
        """
        Create a Contract object defining what will
        be purchased, at which exchange and in which currency.

        symbol - The ticker symbol for the contract
        sec_type - The security type for the contract ('STK' is 'stock')
        exch - The exchange to carry out the contract on
        prim_exch - The primary exchange to carry out the contract on
        curr - The currency in which to purchase the contract
        """
        contract = Contract()
        contract.m_symbol = symbol
        contract.m_secType = sec_type
        contract.m_exchange = exch
        contract.m_primaryExch = prim_exch
        contract.m_currency = curr
        return contract
```

The following method, ***create_order***, generates the second component of the pair, namely the ***Order*** instance. It expects an order type (e.g. market or limit), a quantity of the asset to trade and an "action" (buy or sell). It returns the ***Order*** instance:

```
# ib_execution.py
    
    def create_order(self, order_type, quantity, action):
        """
        Create an Order object (Market/Limit) to go long/short.

        order_type - 'MKT', 'LMT' for Market or Limit orders
        quantity - Integral number of assets to order
        action - 'BUY' or 'SELL'
        """
        order = Order()
        order.m_orderType = order_type
        order.m_totalQuantity = quantity
        order.m_action = action
        return order
```

In order to avoid duplicating ***FillEvent*** instances for a particular order ID, we utilise a dictionary called the ***fill_dict*** to store keys that match particular order IDs. When a fill has been generated the "filled" key of an entry for a particular order ID is set to ***True***. If a subsequent "Server Response" message is received from IB stating that an order has been filled (and is a duplicate message) it will not lead to a new fill. The following method ***create_fill_dict_entry*** carries this out:

```
# ib_execution.py
    
    def create_fill_dict_entry(self, msg):
        """
        Creates an entry in the Fill Dictionary that lists 
        orderIds and provides security information. This is
        needed for the event-driven behaviour of the IB
        server message behaviour.
        """
        self.fill_dict[msg.orderId] = {
            "symbol": msg.contract.m_symbol,
            "exchange": msg.contract.m_exchange,
            "direction": msg.order.m_action,
            "filled": False
        }
```

The following method, ***create_fill***, actually creates the ***FillEvent*** instance and places it onto the events queue:

```
# ib_execution.py
    
    def create_fill(self, msg):
        """
        Handles the creation of the FillEvent that will be
        placed onto the events queue subsequent to an order
        being filled.
        """
        fd = self.fill_dict[msg.orderId]

        # Prepare the fill data
        symbol = fd["symbol"]
        exchange = fd["exchange"]
        filled = msg.filled
        direction = fd["direction"]
        fill_cost = msg.avgFillPrice

        # Create a fill event object
        fill = FillEvent(
            datetime.datetime.utcnow(), symbol, 
            exchange, filled, direction, fill_cost
        )

        # Make sure that multiple messages don't create
        # additional fills.
        self.fill_dict[msg.orderId]["filled"] = True

        # Place the fill event onto the event queue
        self.events.put(fill_event)
```

Now that all of the preceeding methods having been implemented it remains to override the ***execute_order*** method from the ***ExecutionHandler*** abstract base class. This method actually carries out the order placement with the IB API.

We first check that the event being received to this method is actually an ***OrderEvent*** and then prepare the ***Contract*** and ***Order*** objects with their respective parameters. Once both are created the IbPy method ***placeOrder*** of the connection object is called with an associated ***order_id***.

It is extremely important to call the ***time.sleep(1)*** method to ensure the order actually goes through to IB. Removal of this line leads to inconsistent behaviour of the API, at least on my system!

Finally, we increment the order ID to ensure we don't duplicate orders:

```
# ib_execution.py
    
    def execute_order(self, event):
        """
        Creates the necessary InteractiveBrokers order object
        and submits it to IB via their API.

        The results are then queried in order to generate a
        corresponding Fill object, which is placed back on
        the event queue.

        Parameters:
        event - Contains an Event object with order information.
        """
        if event.type == 'ORDER':
            # Prepare the parameters for the asset order
            asset = event.symbol
            asset_type = "STK"
            order_type = event.order_type
            quantity = event.quantity
            direction = event.direction

            # Create the Interactive Brokers contract via the 
            # passed Order event
            ib_contract = self.create_contract(
                asset, asset_type, self.order_routing,
                self.order_routing, self.currency
            )

            # Create the Interactive Brokers order via the 
            # passed Order event
            ib_order = self.create_order(
                order_type, quantity, direction
            )

            # Use the connection to the send the order to IB
            self.tws_conn.placeOrder(
                self.order_id, ib_contract, ib_order
            )

            # NOTE: This following line is crucial.
            # It ensures the order goes through!
            time.sleep(1)

            # Increment the order ID for this session
            self.order_id += 1
```

This class forms the basis of an Interactive Brokers execution handler and can be used in place of the simulated execution handler, which is only suitable for backtesting. Before the IB handler can be utilised, however, it is necessary to create a live market feed handler to replace the historical data feed handler of the backtester system. This will be the subject of a future article.

In this way we are reusing as much as possible from the backtest and live systems to ensure that code "swap out" is minimised and thus behaviour across both is similar, if not identical.