# Event-Driven Backtesting with Python - Part V

   *Source: [QuantStart](https://www.quantstart.com/articles/Event-Driven-Backtesting-with-Python-Part-V/)*
   
   *中文：[用Python进行事件驱动的回测-第5部分](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/zh/用Python进行事件驱动的回测-第5部分.md)*

In the [previous article](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-IV.md) on event-driven backtesting we considered how to construct a Strategy class hierarchy. Strategies, as defined here, are used to generate *signals*, which are used by a portfolio object in order to make decisions on whether to send *orders*. As before it is natural to create a ***Portfolio*** abstract base class (ABC) that all subsequent subclasses inherit from.

This article describes a ***NaivePortfolio*** object that keeps track of the positions within a portfolio and generates orders of a fixed quantity of stock based on signals. Later portfolio objects will include more sophisticated risk management tools and will be the subject of later articles.

## Position Tracking and Order Management

The portfolio order management system is possibly the most complex component of an event-driven backtester. Its role is to keep track of all current market positions as well as the market value of the positions (known as the "holdings"). This is simply an estimate of the liquidation value of the position and is derived in part from the data handling facility of the backtester.

In addition to the positions and holdings management the portfolio must also be aware of risk factors and position sizing techniques in order to optimise orders that are sent to a brokerage or other form of market access.

Continuing in the vein of the ***Event*** class hierarchy a ***Portfolio*** object must be able to handle ***SignalEvent*** objects, generate ***OrderEvent*** objects and interpret ***FillEvent*** objects to update positions. Thus it is no surprise that the ***Portfolio*** objects are often the largest component of event-driven systems, in terms of lines of code (LOC).

## Implementation

We create a new file ***portfolio.py*** and import the necessary libraries. These are the same as most of the other abstract base class implementations. We need to import the ***floor*** function from the ***math*** library in order to generate integer-valued order sizes. We also need the ***FillEvent*** and ***OrderEvent*** objects since the ***Portfolio*** handles both.

```
# portfolio.py

import datetime
import numpy as np
import pandas as pd
import Queue

from abc import ABCMeta, abstractmethod
from math import floor

from event import FillEvent, OrderEvent
```

As before we create an ABC for ***Portfolio*** and have two pure virtual methods ***update_signal*** and ***update_fill***. The former handles new trading signals being grabbed from the events queue and the latter handles fills received from an execution handler object.

```
# portfolio.py

class Portfolio(object):
    """
    The Portfolio class handles the positions and market
    value of all instruments at a resolution of a "bar",
    i.e. secondly, minutely, 5-min, 30-min, 60 min or EOD.
    """

    __metaclass__ = ABCMeta

    @abstractmethod
    def update_signal(self, event):
        """
        Acts on a SignalEvent to generate new orders 
        based on the portfolio logic.
        """
        raise NotImplementedError("Should implement update_signal()")

    @abstractmethod
    def update_fill(self, event):
        """
        Updates the portfolio current positions and holdings 
        from a FillEvent.
        """
        raise NotImplementedError("Should implement update_fill()")
```

The principal subject of this article is the ***NaivePortfolio*** class. It is designed to handle position sizing and current holdings, but will carry out trading orders in a "dumb" manner by simply sending them directly to the brokerage with a predetermined fixed quantity size, irrespective of cash held. These are all unrealistic assumptions, but they help to outline how a portfolio order management system (OMS) functions in an event-driven fashion.

The ***NaivePortfolio*** requires an initial capital value, which I have set to the default of 100,000 USD. It also requires a starting date-time.

The portfolio contains the ***all_positions*** and ***current_positions*** members. The former stores a list of all previous *positions* recorded at the timestamp of a market data event. A position is simply the quantity of the asset. Negative positions mean the asset has been shorted. The latter member stores a dictionary containing the current positions for the last market bar update.

In addition to the positions members the portfolio stores *holdings*, which describe the current market value of the positions held. "Current market value" in this instance means the closing price obtained from the current market bar, which is clearly an approximation, but is reasonable enough for the time being. ***all_holdings*** stores the historical list of all symbol holdings, while ***current_holdings*** stores the most up to date dictionary of all symbol holdings values.

```
# portfolio.py

class NaivePortfolio(Portfolio):
    """
    The NaivePortfolio object is designed to send orders to
    a brokerage object with a constant quantity size blindly,
    i.e. without any risk management or position sizing. It is
    used to test simpler strategies such as BuyAndHoldStrategy.
    """
    
    def __init__(self, bars, events, start_date, initial_capital=100000.0):
        """
        Initialises the portfolio with bars and an event queue. 
        Also includes a starting datetime index and initial capital 
        (USD unless otherwise stated).

        Parameters:
        bars - The DataHandler object with current market data.
        events - The Event Queue object.
        start_date - The start date (bar) of the portfolio.
        initial_capital - The starting capital in USD.
        """
        self.bars = bars
        self.events = events
        self.symbol_list = self.bars.symbol_list
        self.start_date = start_date
        self.initial_capital = initial_capital
        
        self.all_positions = self.construct_all_positions()
        self.current_positions = dict( (k,v) for k, v in [(s, 0) for s in self.symbol_list] )

        self.all_holdings = self.construct_all_holdings()
        self.current_holdings = self.construct_current_holdings()
```

The following method, ***construct_all_positions***, simply creates a dictionary for each symbol, sets the value to zero for each and then adds a datetime key, finally adding it to a list. It uses a [dictionary comprehension](http://stackoverflow.com/questions/1747817/python-create-a-dictionary-with-list-comprehension), which is similar in spirit to a list comprehension:

```
# portfolio.py

    def construct_all_positions(self):
        """
        Constructs the positions list using the start_date
        to determine when the time index will begin.
        """
        d = dict( (k,v) for k, v in [(s, 0) for s in self.symbol_list] )
        d['datetime'] = self.start_date
        return [d]
```

The ***construct_all_holdings*** method is similar to the above but adds extra keys for cash, commission and total, which respectively represent the spare cash in the account after any purchases, the cumulative commission accrued and the total account equity including cash and any open positions. Short positions are treated as negative. The starting cash and total account equity are both set to the initial capital:

```
# portfolio.py

    def construct_all_holdings(self):
        """
        Constructs the holdings list using the start_date
        to determine when the time index will begin.
        """
        d = dict( (k,v) for k, v in [(s, 0.0) for s in self.symbol_list] )
        d['datetime'] = self.start_date
        d['cash'] = self.initial_capital
        d['commission'] = 0.0
        d['total'] = self.initial_capital
        return [d]
```

The following method, ***construct_current_holdings*** is almost identical to the method above except that it doesn't wrap the dictionary in a list:

```
# portfolio.py

    def construct_current_holdings(self):
        """
        This constructs the dictionary which will hold the instantaneous
        value of the portfolio across all symbols.
        """
        d = dict( (k,v) for k, v in [(s, 0.0) for s in self.symbol_list] )
        d['cash'] = self.initial_capital
        d['commission'] = 0.0
        d['total'] = self.initial_capital
        return d
```

On every "heartbeat", that is every time new market data is requested from the ***DataHandler*** object, the portfolio must update the current market value of all the positions held. In a live trading scenario this information can be downloaded and parsed directly from the brokerage, but for a backtesting implementation it is necessary to calculate these values manually.

Unfortunately there is no such as thing as the "current market value" due to bid/ask spreads and liquidity issues. Thus it is necessary to estimate it by multiplying the quantity of the asset held by a "price". The approach I have taken here is to use the closing price of the last bar received. For an intraday strategy this is relatively realistic. For a daily strategy this is less realistic as the opening price can differ substantially from the closing price.

The method ***update_timeindex*** handles the new holdings tracking. It firstly obtains the latest prices from the market data handler and creates a new dictionary of symbols to represent the current positions, by setting the "new" positions equal to the "current" positions. These are only changed when a ***FillEvent*** is obtained, which is handled later on in the portfolio. The method then appends this set of current positions to the ***all_positions*** list. Next, the holdings are updated in a similar manner, with the exception that the market value is recalculated by multiplying the current positions count with the closing price of the latest bar (***self.current_positions[s] * bars[s][0][5]***). Finally the new holdings are appended to ***all_holdings***:

```
# portfolio.py

    def update_timeindex(self, event):
        """
        Adds a new record to the positions matrix for the current 
        market data bar. This reflects the PREVIOUS bar, i.e. all
        current market data at this stage is known (OLHCVI).

        Makes use of a MarketEvent from the events queue.
        """
        bars = {}
        for sym in self.symbol_list:
            bars[sym] = self.bars.get_latest_bars(sym, N=1)

        # Update positions
        dp = dict( (k,v) for k, v in [(s, 0) for s in self.symbol_list] )
        dp['datetime'] = bars[self.symbol_list[0]][0][1]

        for s in self.symbol_list:
            dp[s] = self.current_positions[s]

        # Append the current positions
        self.all_positions.append(dp)

        # Update holdings
        dh = dict( (k,v) for k, v in [(s, 0) for s in self.symbol_list] )
        dh['datetime'] = bars[self.symbol_list[0]][0][1]
        dh['cash'] = self.current_holdings['cash']
        dh['commission'] = self.current_holdings['commission']
        dh['total'] = self.current_holdings['cash']

        for s in self.symbol_list:
            # Approximation to the real value
            market_value = self.current_positions[s] * bars[s][0][5]
            dh[s] = market_value
            dh['total'] += market_value

        # Append the current holdings
        self.all_holdings.append(dh)
```

The method ***update_positions_from_fill*** determines whether a ***FillEvent*** is a Buy or a Sell and then updates the ***current_positions*** dictionary accordingly by adding/subtracting the correct quantity of shares:

```
# portfolio.py

    def update_positions_from_fill(self, fill):
        """
        Takes a FilltEvent object and updates the position matrix
        to reflect the new position.

        Parameters:
        fill - The FillEvent object to update the positions with.
        """
        # Check whether the fill is a buy or sell
        fill_dir = 0
        if fill.direction == 'BUY':
            fill_dir = 1
        if fill.direction == 'SELL':
            fill_dir = -1

        # Update positions list with new quantities
        self.current_positions[fill.symbol] += fill_dir*fill.quantity
```

The corresponding ***update_holdings_from_fill*** is similar to the above method but updates the *holdings* values instead. In order to simulate the cost of a fill, the following method does not use the cost associated from the ***FillEvent***. Why is this? Simply put, in a backtesting environment the fill cost is actually unknown and thus is must be estimated. Thus the fill cost is set to the the "current market price" (the closing price of the last bar). The holdings for a particular symbol are then set to be equal to the fill cost multiplied by the transacted quantity.

Once the fill cost is known the current holdings, cash and total values can all be updated. The cumulative commission is also updated:

```
# portfolio.py

    def update_holdings_from_fill(self, fill):
        """
        Takes a FillEvent object and updates the holdings matrix
        to reflect the holdings value.

        Parameters:
        fill - The FillEvent object to update the holdings with.
        """
        # Check whether the fill is a buy or sell
        fill_dir = 0
        if fill.direction == 'BUY':
            fill_dir = 1
        if fill.direction == 'SELL':
            fill_dir = -1

        # Update holdings list with new quantities
        fill_cost = self.bars.get_latest_bars(fill.symbol)[0][5]  # Close price
        cost = fill_dir * fill_cost * fill.quantity
        self.current_holdings[fill.symbol] += cost
        self.current_holdings['commission'] += fill.commission
        self.current_holdings['cash'] -= (cost + fill.commission)
        self.current_holdings['total'] -= (cost + fill.commission)
```

The pure virtual ***update_fill*** method from the ***Portfolio*** ABC is implemented here. It simply executes the two preceding methods, ***update_positions_from_fill*** and ***update_holdings_from_fill***, which have already been discussed above:

```
# portfolio.py

    def update_fill(self, event):
        """
        Updates the portfolio current positions and holdings 
        from a FillEvent.
        """
        if event.type == 'FILL':
            self.update_positions_from_fill(event)
            self.update_holdings_from_fill(event)
```

While the ***Portfolio*** object must handle ***FillEvents***, it must also take care of generating ***OrderEvents*** upon the receipt of one or more ***SignalEvents***. The ***generate_naive_order*** method simply takes a signal to long or short an asset and then sends an order to do so for 100 shares of such an asset. Clearly 100 is an arbitrary value. In a realistic implementation this value will be determined by a risk management or position sizing overlay. However, this is a ***NaivePortfolio*** and so it "naively" sends all orders directly from the signals, without a risk system.

The method handles longing, shorting and exiting of a position, based on the current quantity and particular symbol. Corresponding ***OrderEvent*** objects are then generated:

```
# portfolio.py

    def generate_naive_order(self, signal):
        """
        Simply transacts an OrderEvent object as a constant quantity
        sizing of the signal object, without risk management or
        position sizing considerations.

        Parameters:
        signal - The SignalEvent signal information.
        """
        order = None

        symbol = signal.symbol
        direction = signal.signal_type
        strength = signal.strength

        mkt_quantity = floor(100 * strength)
        cur_quantity = self.current_positions[symbol]
        order_type = 'MKT'

        if direction == 'LONG' and cur_quantity == 0:
            order = OrderEvent(symbol, order_type, mkt_quantity, 'BUY')
        if direction == 'SHORT' and cur_quantity == 0:
            order = OrderEvent(symbol, order_type, mkt_quantity, 'SELL')   
    
        if direction == 'EXIT' and cur_quantity > 0:
            order = OrderEvent(symbol, order_type, abs(cur_quantity), 'SELL')
        if direction == 'EXIT' and cur_quantity < 0:
            order = OrderEvent(symbol, order_type, abs(cur_quantity), 'BUY')
        return order
```

The ***update_signal*** method simply calls the above method and adds the generated order to the events queue:

```
# portfolio.py

    def update_signal(self, event):
        """
        Acts on a SignalEvent to generate new orders 
        based on the portfolio logic.
        """
        if event.type == 'SIGNAL':
            order_event = self.generate_naive_order(event)
            self.events.put(order_event)
```

The final method in the ***NaivePortfolio*** is the generation of an equity curve. This simply creates a returns stream, useful for performance calculations and then normalises the equity curve to be percentage based. Thus the account initial size is equal to 1.0:

```
# portfolio.py

    def create_equity_curve_dataframe(self):
        """
        Creates a pandas DataFrame from the all_holdings
        list of dictionaries.
        """
        curve = pd.DataFrame(self.all_holdings)
        curve.set_index('datetime', inplace=True)
        curve['returns'] = curve['total'].pct_change()
        curve['equity_curve'] = (1.0+curve['returns']).cumprod()
        self.equity_curve = curve
```

The ***Portfolio*** object is the most complex aspect of the entire event-driven backtest system. The implementation here, while intricate, is relatively elementary in its handling of positions. Later versions will consider risk management and position sizing, which will lead to a much more realistic idea of strategy performance.

In the [next article](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-VI.md) we will consider the final piece of the event-driven backtester, namely an ***ExecutionHandler*** object, which is used to take ***OrderEvent*** objects and create ***FillEvent*** objects from them.