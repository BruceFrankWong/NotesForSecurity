# Research Backtesting Environments in Python with pandas

 *source： [QuantStart](https://www.quantstart.com/articles/Research-Backtesting-Environments-in-Python-with-pandas/)*

 *中文: [在Python和pandas中的研究回测环境](https://github.com/BruceFrankWong/NotesForSecurity/blob/master/quantstart/Research-Backtesting-Environments-in-Python-with-pandas_zh.md)*

Backtesting is the research process of applying a trading strategy idea to historical data in order to ascertain past performance. In particular, a backtester makes no guarantee about the future performance of the strategy. They are however an essential component of the strategy pipeline research process, allowing strategies to be filtered out before being placed into production.

In this article (and those that follow it) a basic object-oriented backtesting system written in Python will be outlined. This early system will primarily be a "teaching aid", used to demonstrate the different components of a backtesting system. As we progress through the articles, more sophisticated functionality will be added.

## Backtesting Overview
The process of designing a robust backtesting system is extremely difficult. Effectively simulating all of the components that affect the performance of an algorithmic trading system is challenging. Poor data granularity, opaqueness of order routing at a broker, order latency and a myriad of other factors conspire to alter the "true" performance of a strategy versus the backtested performance.

When developing a backtesting system it is tempting to want to constantly "rewrite it from scratch" as more factors are found to be crucial in assessing performance. No backtesting system is ever finished and a judgement must be made at a point during development that enough factors have been captured by the system.

With these concerns in mind the backtester presented in here will be somewhat simplistic. As we explore further issues (portfolio optimisation, risk management, transaction cost handling) the backtester will become more robust.

## Types of Backtesting Systems
There are generally two types of backtesting system that will be of interest. The first is research-based, used primarily in the early stages, where many strategies will be tested in order to select those for more serious assessment. These research backtesting systems are often written in Python, R or MatLab as speed of development is more important than speed of execution in this phase.

The second type of backtesting system is event-based. That is, it carries out the backtesting process in an execution loop similar (if not identical) to the trading execution system itself. It will realistically model market data and the order execution process in order to provide a more rigourous assessment of a strategy.

The latter systems are often written in a high-performance language such as C++ or Java, where speed of execution is essential. For lower frequency strategies (although still intraday), Python is more than sufficient to be used in this context.

## Object-Oriented Research Backtester in Python
The design and implementation of an object-oriented research-based backtesting environment will now be discussed. Object orientation has been chosen as the software design paradigm for the following reasons:

+ The interfaces of each component can be specified upfront, while the internals of each component can be modified (or replaced) as the project progresses

+ By specifying the interfaces upfront it is possible to effectively test how each component behaves (via unit testing)

+ When extending the system new components can be constructed upon or in addition to others, either by inheritance or composition

At this stage the backtester is designed for ease of implementation and a reasonable degree of flexibility, at the expense of true market accuracy. In particular, this backtester will only be able to handle strategies acting on a single instrument. Later the backtester will modified to handle sets of instruments. For the initial backtester, the following components are required:

+ Strategy - A Strategy class receives a Pandas DataFrame of bars, i.e. a list of Open-High-Low-Close-Volume (OHLCV) data points at a particular frequency. The Strategy will produce a list of signals, which consist of a timestamp and an element from the set  indicating a long, hold or short signal respectively.

+ Portfolio - The majority of the backtesting work will occur in the Portfolio class. It will receive a set of signals (as described above) and create a series of positions, allocated against a cash component. The job of the Portfolio object is to produce an equity curve, incorporate basic transaction costs and keep track of trades.

+ Performance - The Performance object takes a portfolio and produces a set of statistics about its performance. In particular it will output risk/return characteristics (Sharpe, Sortino and Information Ratios), trade/profit metrics and drawdown information.

### What's Missing?
As can be seen this backtester does not include any reference to portfolio/risk management, execution handling (i.e. no limit orders) nor will it provide sophisticated modelling of transaction costs. This isn't much of a problem at this stage. It allows us to gain familiarity with the process of creating an object-oriented backtester and the Pandas/NumPy libraries. In time it will be improved.

## Implementation
We will now proceed to outline the implementations for each object.

### Strategy
The Strategy object must be quite generic at this stage, since it will be handling forecasting, mean-reversion, momentum and volatility strategies. The strategies being considered here will always be time series based, i.e. "price driven". An early requirement for this backtester is that derived Strategy classes will accept a list of bars (OHLCV) as input, rather than ticks (trade-by-trade prices) or order-book data. Thus the finest granularity being considered here will be 1-second bars.

The Strategy class will also always produce signal recommendations. This means that it will advise a Portfolio instance in the sense of going long/short or holding a position. This flexibility will allow us to create multiple Strategy "advisors" that provide a set of signals, which a more advanced Portfolio class can accept in order to determine the actual positions being entered.

The interface of the classes will be enforced by utilising an abstract base class methodology. An abstract base class is an object that cannot be instantiated and thus only derived classes can be created. The Python code is given below in a file called backtest.py. The Strategy class requires that any subclass implement the generate_signals method.

In order to prevent the Strategy class from being instantiated directly (since it is abstract!) it is necessary to use the ABCMeta and abstractmethod objects from the abc module. We set a property of the class, called __metaclass__ to be equal to ABCMeta and then decorate the generate_signals method with the abstractmethod decorator.

```
# backtest.py

from abc import ABCMeta, abstractmethod

class Strategy(object):
    """Strategy is an abstract base class providing an interface for
    all subsequent (inherited) trading strategies.

    The goal of a (derived) Strategy object is to output a list of signals,
    which has the form of a time series indexed pandas DataFrame.

    In this instance only a single symbol/instrument is supported."""

    __metaclass__ = ABCMeta

    @abstractmethod
    def generate_signals(self):
        """An implementation is required to return the DataFrame of symbols 
        containing the signals to go long, short or hold (1, -1 or 0)."""
        raise NotImplementedError("Should implement generate_signals()!")
```

While the above interface is straightforward it will become more complicated when this class is inherited for each specific type of strategy. Ultimately the goal of the Strategy class in this setting is to provide a list of long/short/hold signals for each instrument to be sent to a Portfolio.

### Portfolio
The Portfolio class is where the majority of the trading logic will reside. For this research backtester the Portfolio is in charge of determining position sizing, risk analysis, transaction cost management and execution handling (i.e. market-on-open, market-on-close orders). At a later stage these tasks will be broken down into separate components. Right now they will be rolled in to one class.

This class makes ample use of pandas and provides a great example of where the library can save a huge amount of time, particularly in regards to "boilerplate" data wrangling. As an aside, the main trick with pandas and NumPy is to avoid iterating over any dataset using the for d in ... syntax. This is because NumPy (which underlies pandas) optimises looping by vectorised operations. Thus you will see few (if any!) direct iterations when utilising pandas.

The goal of the Portfolio class is to ultimately produce a sequence of trades and an equity curve, which will be analysed by the Performance class. In order to achieve this it must be provided with a list of trading recommendations from a Strategy object. Later on, this will be a group of Strategy objects.

The Portfolio class will need to be told how capital is to be deployed for a particular set of trading signals, how to handle transaction costs and which forms of orders will be utilised. The Strategy object is operating on bars of data and thus assumptions must be made in regard to prices achieved at execution of an order. Since the high/low price of any bar is unknown a priori it is only possible to use the open and close prices for trading. In reality it is impossible to guarantee that an order will be filled at one of these particular prices when using a market order, so it will be, at best, an approximation.

In addition to assumptions about orders being filled, this backtester will ignore all concepts of margin/brokerage constraints and will assume that it is possible to go long and short in any instrument freely without any liquidity constraints. This is clearly a very unrealistic assumption, but is one that can be relaxed later.

The following listing continues backtest.py:

```
# backtest.py

class Portfolio(object):
    """An abstract base class representing a portfolio of 
    positions (including both instruments and cash), determined
    on the basis of a set of signals provided by a Strategy."""

    __metaclass__ = ABCMeta

    @abstractmethod
    def generate_positions(self):
        """Provides the logic to determine how the portfolio 
        positions are allocated on the basis of forecasting
        signals and available cash."""
        raise NotImplementedError("Should implement generate_positions()!")

    @abstractmethod
    def backtest_portfolio(self):
        """Provides the logic to generate the trading orders
        and subsequent equity curve (i.e. growth of total equity),
        as a sum of holdings and cash, and the bar-period returns
        associated with this curve based on the 'positions' DataFrame.

        Produces a portfolio object that can be examined by 
        other classes/functions."""
        raise NotImplementedError("Should implement backtest_portfolio()!")
```

At this stage the Strategy and Portfolio abstract base classes have been introduced. We are now in a position to generate some concrete derived implementations of these classes, in order to produce a working "toy strategy".

We will begin by generating a subclass of Strategy called RandomForecastStrategy, the sole task of which is to produce randomly chosen long/short signals! While this is clearly a nonsensical trading strategy, it will serve our needs by demonstrating the object oriented backtesting framework. Thus we will begin a new file called random_forecast.py, with the listing for the random forecaster as follows:

```
# random_forecast.py

import numpy as np
import pandas as pd
import Quandl   # Necessary for obtaining financial data easily

from backtest import Strategy, Portfolio

class RandomForecastingStrategy(Strategy):
    """Derives from Strategy to produce a set of signals that
    are randomly generated long/shorts. Clearly a nonsensical
    strategy, but perfectly acceptable for demonstrating the
    backtesting infrastructure!"""    
    
    def __init__(self, symbol, bars):
        """Requires the symbol ticker and the pandas DataFrame of bars"""
        self.symbol = symbol
        self.bars = bars

    def generate_signals(self):
        """Creates a pandas DataFrame of random signals."""
        signals = pd.DataFrame(index=self.bars.index)
        signals['signal'] = np.sign(np.random.randn(len(signals)))

        # The first five elements are set to zero in order to minimise
        # upstream NaN errors in the forecaster.
        signals['signal'][0:5] = 0.0
        return signals
```

Now that we have a "concrete" forecasting system, we must create an implementation of a Portfolio object. This object will encompass the majority of the backtesting code. It is designed to create two separate DataFrames, the first of which is a positions frame, used to store the quantity of each instrument held at any particular bar. The second, portfolio, actually contains the market price of all holdings for each bar, as well as a tally of the cash, assuming an initial capital. This ultimately provides an equity curve on which to assess strategy performance.

The Portfolio object, while extremely flexible in its interface, requires specific choices when regarding how to handle transaction costs, market orders etc. In this basic example I have considered that it will be possible to go long/short an instrument easily with no restrictions or margin, buy or sell directly at the open price of the bar, zero transaction costs (encompassing slippage, fees and market impact) and have specified the quantity of stock directly to purchase for each trade.

Here is the continuation of the random_forecast.py listing:

```
# random_forecast.py

class MarketOnOpenPortfolio(Portfolio):
    """Inherits Portfolio to create a system that purchases 100 units of 
    a particular symbol upon a long/short signal, assuming the market 
    open price of a bar.

    In addition, there are zero transaction costs and cash can be immediately 
    borrowed for shorting (no margin posting or interest requirements). 

    Requires:
    symbol - A stock symbol which forms the basis of the portfolio.
    bars - A DataFrame of bars for a symbol set.
    signals - A pandas DataFrame of signals (1, 0, -1) for each symbol.
    initial_capital - The amount in cash at the start of the portfolio."""

    def __init__(self, symbol, bars, signals, initial_capital=100000.0):
        self.symbol = symbol        
        self.bars = bars
        self.signals = signals
        self.initial_capital = float(initial_capital)
        self.positions = self.generate_positions()
        
    def generate_positions(self):
        """Creates a 'positions' DataFrame that simply longs or shorts
        100 of the particular symbol based on the forecast signals of
        {1, 0, -1} from the signals DataFrame."""
        positions = pd.DataFrame(index=signals.index).fillna(0.0)
        positions[self.symbol] = 100*signals['signal']
        return positions
                    
    def backtest_portfolio(self):
        """Constructs a portfolio from the positions DataFrame by 
        assuming the ability to trade at the precise market open price
        of each bar (an unrealistic assumption!). 

        Calculates the total of cash and the holdings (market price of
        each position per bar), in order to generate an equity curve
        ('total') and a set of bar-based returns ('returns').

        Returns the portfolio object to be used elsewhere."""

        # Construct the portfolio DataFrame to use the same index
        # as 'positions' and with a set of 'trading orders' in the
        # 'pos_diff' object, assuming market open prices.
        portfolio = self.positions*self.bars['Open']
        pos_diff = self.positions.diff()

        # Create the 'holdings' and 'cash' series by running through
        # the trades and adding/subtracting the relevant quantity from
        # each column
        portfolio['holdings'] = (self.positions*self.bars['Open']).sum(axis=1)
        portfolio['cash'] = self.initial_capital - (pos_diff*self.bars['Open']).sum(axis=1).cumsum()

        # Finalise the total and bar-based returns based on the 'cash'
        # and 'holdings' figures for the portfolio
        portfolio['total'] = portfolio['cash'] + portfolio['holdings']
        portfolio['returns'] = portfolio['total'].pct_change()
        return portfolio
```

This gives us everything we need to generate an equity curve based on such a system. The final step is to tie it all together with a __main__ function:

```
if __name__ == "__main__":
    # Obtain daily bars of SPY (ETF that generally 
    # follows the S&P500) from Quandl (requires 'pip install Quandl'
    # on the command line)
    symbol = 'SPY'
    bars = Quandl.get("GOOG/NYSE_%s" % symbol, collapse="daily")

    # Create a set of random forecasting signals for SPY
    rfs = RandomForecastingStrategy(symbol, bars)
    signals = rfs.generate_signals()

    # Create a portfolio of SPY
    portfolio = MarketOnOpenPortfolio(symbol, bars, signals, initial_capital=100000.0)
    returns = portfolio.backtest_portfolio()

    print returns.tail(10)
```

The output of the program is as follows. Yours will differ from the output below depending upon the date range you select and the random seed used:

```
              SPY  holdings    cash  total   returns
Date                                                
2014-01-02 -18398    -18398  111486  93088  0.000097
2014-01-03  18321     18321   74844  93165  0.000827
2014-01-06  18347     18347   74844  93191  0.000279
2014-01-07  18309     18309   74844  93153 -0.000408
2014-01-08 -18345    -18345  111534  93189  0.000386
2014-01-09 -18410    -18410  111534  93124 -0.000698
2014-01-10 -18395    -18395  111534  93139  0.000161
2014-01-13 -18371    -18371  111534  93163  0.000258
2014-01-14 -18228    -18228  111534  93306  0.001535
2014-01-15  18410     18410   74714  93124 -0.001951
None
```

In this instance the strategy lost money, which is unsurprising given the stochastic nature of the forecaster! The next steps are to create a Performance object that accepts a Portfolio instance and provides a list of performance metrics upon which to base a decision to filter the strategy out or not.

We can also improve the Portfolio object to have a more realistic handling of transaction costs (such as Interactive Brokers commissions and slippage). We can also straightforwardly include a forecasting engine into a Strategy object, which will (hopefully) produce better results. In the following articles we will explore these concepts in more depth.
