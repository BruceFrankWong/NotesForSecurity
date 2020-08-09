# Event-Driven Backtesting with Python - Part I

   *Source: [QuantStart](https://www.quantstart.com/articles/Event-Driven-Backtesting-with-Python-Part-I/)*
   
   *中文：[用Python进行事件驱动的回测-第1部分](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/zh/Event-Driven-Backtesting-with-Python-Part-I_zh.md)*

We've spent the last couple of months on QuantStart backtesting various trading strategies utilising Python and [pandas](http://pandas.pydata.org/). The vectorised nature of pandas ensures that certain operations on large datasets are extremely rapid. However the forms of vectorised backtester that we have studied to date suffer from some drawbacks in the way that trade execution is simulated. In this series of articles we are going to discuss a more realistic approach to historical strategy simulation by constructing an **event-driven** backtesting environment using Python.

## Event-Driven Software

Before we delve into development of such a backtester we need to understand the concept of event-driven systems. Video games provide a natural use case for event-driven software and provide a straightforward example to explore. A video game has multiple components that interact with each other in a real-time setting at high framerates. This is handled by running the entire set of calculations within an "infinite" loop known as the **event-loop** or **game-loop**.

At each tick of the game-loop a function is called to receive the latest **event**, which will have been generated by some corresponding prior action within the game. Depending upon the nature of the event, which could include a key-press or a mouse click, some subsequent action is taken, which will either terminate the loop or generate some additional events. The process will then continue. Here is some example pseudo-code:

```
while True:  # Run the loop forever
    new_event = get_new_event()   # Get the latest event

    # Based on the event type, perform an action
    if new_event.type == "LEFT_MOUSE_CLICK":
        open_menu()
    elif new_event.type == "ESCAPE_KEY_PRESS":
        quit_game()
    elif new_event.type == "UP_KEY_PRESS":
        move_player_north()
    # ... and many more events

    redraw_screen()   # Update the screen to provide animation
    tick(50)   # Wait 50 milliseconds
```

The code is continually checking for new events and then performing actions based on these events. In particular it allows the illusion of real-time response handling because the code is continually being looped and events checked for. As will become clear this is precisely what we need in order to carry out high frequency trading simulation.

## Why An Event-Driven Backtester?

Event-driven systems provide many advantages over a vectorised approach:

+ **Code Reuse** - An event-driven backtester, by design, can be used for both historical backtesting and live trading with minimal switch-out of components. This is not true of vectorised backtesters where all data must be available at once to carry out statistical analysis.

+ **Lookahead Bias** - With an event-driven backtester there is no lookahead bias as market data receipt is treated as an "event" that must be acted upon. Thus it is possible to "drip feed" an event-driven backtester with market data, replicating how an order management and portfolio system would behave.

+ **Realism** - Event-driven backtesters allow significant customisation over how orders are executed and transaction costs are incurred. It is straightforward to handle basic market and limit orders, as well as market-on-open (MOO) and market-on-close (MOC), since a custom exchange handler can be constructed.

Although event-driven systems come with many benefits they suffer from two major disadvantages over simpler vectorised systems. Firstly they are significantly more complex to implement and test. There are more "moving parts" leading to a greater chance of introducing bugs. To mitigate this proper software testing methodology such as [test-driven development](http://en.wikipedia.org/wiki/Test-driven_development) can be employed.

Secondly they are slower to execute compared to a vectorised system. Optimal vectorised operations are unable to be utilised when carrying out mathematical calculations. We will discuss ways to overcome these limitations in later articles.

## Event-Driven Backtester Overview

To apply an event-driven approach to a backtesting system it is necessary to define our components (or objects) that will handle specific tasks:

+ **Event** - The Event is the fundamental class unit of the event-driven system. It contains a type (such as "MARKET", "SIGNAL", "ORDER" or "FILL") that determines how it will be handled within the event-loop.

+ **Event Queue** - The ***Event*** Queue is an in-memory Python Queue object that stores all of the Event sub-class objects that are generated by the rest of the software.

+ **DataHandler** - The ***DataHandler*** is an [abstract base class](http://en.wikipedia.org/wiki/Class_%28computer_programming%29#Abstract_and_concrete) (ABC) that presents an interface for handling both historical or live market data. This provides significant flexibility as the Strategy and Portfolio modules can thus be reused between both approaches. The DataHandler generates a new ***MarketEvent*** upon every heartbeat of the system (see below).

+ **Strategy** - The ***Strategy*** is also an ABC that presents an interface for taking market data and generating corresponding SignalEvents, which are ultimately utilised by the Portfolio object. A SignalEvent contains a ticker symbol, a direction (LONG or SHORT) and a timestamp.

+ **Portfolio** - This is an ABC which handles the order management associated with current and subsequent positions for a strategy. It also carries out risk management across the portfolio, including sector exposure and position sizing. In a more sophisticated implementation this could be delegated to a RiskManagement class. The ***Portfolio*** takes SignalEvents from the Queue and generates OrderEvents that get added to the Queue.

+ **ExecutionHandler** - The ***ExecutionHandler*** simulates a connection to a brokerage. The job of the handler is to take OrderEvents from the Queue and execute them, either via a simulated approach or an actual connection to a liver brokerage. Once orders are executed the handler creates FillEvents, which describe what was actually transacted, including fees, commission and slippage (if modelled).

+ **The Loop** - All of these components are wrapped in an event-loop that correctly handles all Event types, routing them to the appropriate component.

This is quite a basic model of a trading engine. There is significant scope for expansion, particularly in regard to how the ***Portfolio*** is used. In addition differing transaction cost models might also be abstracted into their own class hierarchy. At this stage it introduces needless complexity within this series of articles so we will not currently discuss it further. In later tutorials we will likely expand the system to include additional realism.

Here is a snippet of Python code that demonstrates how the backtester works in practice. There are two loops occuring in the code. The outer loop is used to give the backtester a **heartbeat**. For live trading this is the frequency at which new market data is polled. For backtesting strategies this is not strictly necessary since the backtester uses the market data provided in drip-feed form (see the ***bars.update_bars()*** line).

The inner loop actually handles the Events from the ***events*** Queue object. Specific events are delegated to the respective component and subsequently new events are added to the queue. When the events Queue is empty, the heartbeat loop continues:

```
# Declare the components with respective parameters
bars = DataHandler(..)
strategy = Strategy(..)
port = Portfolio(..)
broker = ExecutionHandler(..)

while True:
    # Update the bars (specific backtest code, as opposed to live trading)
    if bars.continue_backtest == True:
        bars.update_bars()
    else:
        break
    
    # Handle the events
    while True:
        try:
            event = events.get(False)
        except Queue.Empty:
            break
        else:
            if event is not None:
                if event.type == 'MARKET':
                    strategy.calculate_signals(event)
                    port.update_timeindex(event)

                elif event.type == 'SIGNAL':
                    port.update_signal(event)

                elif event.type == 'ORDER':
                    broker.execute_order(event)

                elif event.type == 'FILL':
                    port.update_fill(event)

    # 10-Minute heartbeat
    time.sleep(10*60)
```

This is the basic outline of a how an event-driven backtester is designed. In the [next](https://github.com/BruceFrankWong/NotesForSecurity/tree/master/quantstart/en/Event-Driven-Backtesting-with-Python-Part-II.md) article we will discuss the Event class hierarchy.