# Event-Driven Backtesting with Python - Part III

   *Source: [QuantStart](https://www.quantstart.com/articles/Event-Driven-Backtesting-with-Python-Part-III/)*
   
   *中文：[用Python进行事件驱动的回测-第3部分](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/zh/用Python进行事件驱动的回测-第3部分.md)*

In the previous two articles of the series [we discussed what an event-driven backtesting system is](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-I.md) and the [class hierarchy for the Event object](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-II.md). In this article we are going to consider how market data is utilised, both in a historical backtesting context and for live trade execution.

One of our goals with an event-driven trading system is to minimise duplication of code between the backtesting element and the live execution element. Ideally it would be optimal to utilise the same signal generation methodology and portfolio management components for both historical testing and live trading. In order for this to work the ***Strategy*** object which generates the Signals, and the ***Portfolio*** object which provides Orders based on them, must utilise an identical interface to a market feed for both historic and live running.

This motivates the concept of a class hierarchy based on a ***DataHandler*** object, which gives all subclasses an interface for providing market data to the remaining components within the system. In this way any subclass data handler can be "swapped out", without affecting strategy or portfolio calculation.

Specific example subclasses could include ***HistoricCSVDataHandler***, ***QuandlDataHandler***, ***SecuritiesMasterDataHandler***, ***InteractiveBrokersMarketFeedDataHandler*** etc. In this tutorial we are only going to consider the creation of a historic CSV data handler, which will load intraday CSV data for equities in an Open-Low-High-Close-Volume-OpenInterest set of bars. This can then be used to "drip feed" on a bar-by-bar basis the data into the ***Strategy*** and ***Portfolio*** classes on every heartbeat of the system, thus avoiding lookahead bias.

The first task is to import the necessary libraries. Specifically we are going to import pandas and the [abstract base class](http://en.wikipedia.org/wiki/Class_%28computer_programming%29#Abstract_and_concrete) tools. Since the DataHandler generates MarketEvents we also need to import ***event.py*** as described in the [previous tutorial](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-II.md):

```
# data.py

import datetime
import os, os.path
import pandas as pd

from abc import ABCMeta, abstractmethod

from event import MarketEvent
```

The ***DataHandler*** is an abstract base class (ABC), which means that it is impossible to instantiate an instance directly. Only subclasses may be instantiated. The rationale for this is that the ABC provides an interface that all subsequent ***DataHandler*** subclasses must adhere to thereby ensuring compatibility with other classes that communicate with them.

We make use of the ***__metaclass__*** property to let Python know that this is an ABC. In addition we use the ***@abstractmethod*** decorator to let Python know that the method will be overridden in subclasses (this is identical to a [pure virtual method](http://en.wikipedia.org/wiki/Virtual_function#Abstract_classes_and_pure_virtual_functions) in C++).

The two methods of interest are ***get_latest_bars*** and ***update_bars***. The former returns the last ***N*** bars from the current heartbeat timestamp, which is useful for rolling calculations needed in Strategy classes. The latter method provides a "drip feed" mechanism for placing bar information on a new data structure that strictly prohibits lookahead bias. Notice that exceptions will be raised if an attempted instantiation of the class occurs:

```
# data.py

class DataHandler(object):
    """
    DataHandler is an abstract base class providing an interface for
    all subsequent (inherited) data handlers (both live and historic).

    The goal of a (derived) DataHandler object is to output a generated
    set of bars (OLHCVI) for each symbol requested. 

    This will replicate how a live strategy would function as current
    market data would be sent "down the pipe". Thus a historic and live
    system will be treated identically by the rest of the backtesting suite.
    """

    __metaclass__ = ABCMeta

    @abstractmethod
    def get_latest_bars(self, symbol, N=1):
        """
        Returns the last N bars from the latest_symbol list,
        or fewer if less bars are available.
        """
        raise NotImplementedError("Should implement get_latest_bars()")

    @abstractmethod
    def update_bars(self):
        """
        Pushes the latest bar to the latest symbol structure
        for all symbols in the symbol list.
        """
        raise NotImplementedError("Should implement update_bars()")
```

With the ***DataHandler*** ABC specified the next step is to create a handler for historic CSV files. In particular the ***HistoricCSVDataHandler*** will take multiple CSV files, one for each symbol, and convert these into a dictionary of pandas DataFrames.

The data handler requires a few parameters, namely an Event Queue on which to push MarketEvent information to, the absolute path of the CSV files and a list of symbols. Here is the initialisation of the class:

```
# data.py

class HistoricCSVDataHandler(DataHandler):
    """
    HistoricCSVDataHandler is designed to read CSV files for
    each requested symbol from disk and provide an interface
    to obtain the "latest" bar in a manner identical to a live
    trading interface. 
    """

    def __init__(self, events, csv_dir, symbol_list):
        """
        Initialises the historic data handler by requesting
        the location of the CSV files and a list of symbols.

        It will be assumed that all files are of the form
        'symbol.csv', where symbol is a string in the list.

        Parameters:
        events - The Event Queue.
        csv_dir - Absolute directory path to the CSV files.
        symbol_list - A list of symbol strings.
        """
        self.events = events
        self.csv_dir = csv_dir
        self.symbol_list = symbol_list

        self.symbol_data = {}
        self.latest_symbol_data = {}
        self.continue_backtest = True       

        self._open_convert_csv_files()
```

It will implicitly try to open the files with the format of "SYMBOL.csv" where symbol is the ticker symbol. The format of the files matches that provided by the [DTN IQFeed](http://www.iqfeed.net/) vendor, but is easily modified to handle additional data formats. The opening of the files is handled by the ***_open_convert_csv_files*** method below.

One of the benefits of using pandas as a datastore internally within the ***HistoricCSVDataHandler*** is that the indexes of all symbols being tracked can be merged together. This allows missing data points to be padded forward, backward or interpolated within these gaps such that tickers can be compared on a bar-to-bar basis. This is necessary for mean-reverting strategies, for instance. Notice the use of the ***union*** and ***reindex*** methods when combining the indexes for all symbols:

```
# data.py

    def _open_convert_csv_files(self):
        """
        Opens the CSV files from the data directory, converting
        them into pandas DataFrames within a symbol dictionary.

        For this handler it will be assumed that the data is
        taken from DTN IQFeed. Thus its format will be respected.
        """
        comb_index = None
        for s in self.symbol_list:
            # Load the CSV file with no header information, indexed on date
            self.symbol_data[s] = pd.io.parsers.read_csv(
                                      os.path.join(self.csv_dir, '%s.csv' % s),
                                      header=0, index_col=0, 
                                      names=['datetime','open','low','high','close','volume','oi']
                                  )

            # Combine the index to pad forward values
            if comb_index is None:
                comb_index = self.symbol_data[s].index
            else:
                comb_index.union(self.symbol_data[s].index)

            # Set the latest symbol_data to None
            self.latest_symbol_data[s] = []

        # Reindex the dataframes
        for s in self.symbol_list:
            self.symbol_data[s] = self.symbol_data[s].reindex(index=comb_index, method='pad').iterrows()
```

The ***_get_new_bar*** method creates a [generator](https://wiki.python.org/moin/Generators) to provide a formatted version of the bar data. This means that subsequent calls to the method will *yield* a new bar until the end of the symbol data is reached:

```
# data.py

    def _get_new_bar(self, symbol):
        """
        Returns the latest bar from the data feed as a tuple of 
        (sybmbol, datetime, open, low, high, close, volume).
        """
        for b in self.symbol_data[symbol]:
            yield tuple([symbol, datetime.datetime.strptime(b[0], '%Y-%m-%d %H:%M:%S'), 
                        b[1][0], b[1][1], b[1][2], b[1][3], b[1][4]])
```

The first abstract method from ***DataHandler*** to be implemented is ***get_latest_bars***. This method simply provides a list of the last N bars from the ***latest_symbol_data*** structure. Setting N=1 allows the retrieval of the current bar (wrapped in a list):

```
# data.py

    def get_latest_bars(self, symbol, N=1):
        """
        Returns the last N bars from the latest_symbol list,
        or N-k if less available.
        """
        try:
            bars_list = self.latest_symbol_data[symbol]
        except KeyError:
            print "That symbol is not available in the historical data set."
        else:
            return bars_list[-N:] 
```

The final method, ***update_bars***, is the second abstract method from ***DataHandler***. It simply generates a ***MarketEvent*** that gets added to the queue as it appends the latest bars to the ***latest_symbol_data***:

```
# data.py

    def update_bars(self):
        """
        Pushes the latest bar to the latest_symbol_data structure
        for all symbols in the symbol list.
        """
        for s in self.symbol_list:
            try:
                bar = self._get_new_bar(s).next()
            except StopIteration:
                self.continue_backtest = False
            else:
                if bar is not None:
                    self.latest_symbol_data[s].append(bar)
        self.events.put(MarketEvent())
```

Thus we have a ***DataHandler***-derived object, which is used by the remaining components to keep track of market data. The ***Strategy***, ***Portfolio*** and ***ExecutionHandler*** objects all require the current market data thus it makes sense to centralise it to avoid duplication of storage.

In the [next article](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-IV.md) we will consider the ***Strategy*** class hierarchy and describe how a strategy can be designed to handle multiple symbols, thus generating multiple ***SignalEvents*** for the ***Portfolio*** object.