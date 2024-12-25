# Available data fields  

There are four frequently used subclass of `MarketData`.  
- `TickData`  
- `TransactionData` | `TradeData`  
- `OrderData`  
- `OrderBook`<br>

***

### TickData  

Represents tick data for a specific ticker.<br>
<br>
**Properties**:<br>
- `ticker` : `str`  
The ticker symbol for the tick data.      
<br> 
- `timestamp` : `float`  
The timestamp of the tick data    
<br> 
- `bid` : `list[list[float | int]] | None`  
A list of bid prices and volumes. Optional, used to build the order book.  
<br> 
- `ask` : `list[list[float | int]] | None`  
A list of ask prices and volumes. Optional, used to build the order book.    
<br> 
- `level_2` : [`OrderBook`](data_fields.md#orderbook) | `None`  
The level 2 order book created from the bid and ask data.  
<br> 
- `order_book` : [`OrderBook`](data_fields.md#orderbook) | `None`  
Alias for `level_2`.  
<br> 
- `last_price` : `float`  
The last traded price.  
<br> 
- `bid_price` : `float | None`  
The bid1 price.  
<br> 
- `ask_price` : `float | None`  
The ask1 price.  
<br> 
- `bid_volume` : `float | None`  
The bid1 volume.  
<br> 
- `ask_volume` : `float | None`  
The ask1 volume.  
<br> 
- `total_traded_volume` : `float`  
The total traded volume, accumulated from opening.  
<br> 
- `total_traded_notional` : `float`  
The total traded notional value, accumulated from opening.  
<br> 
- `total_trade_count` : `float`  
The total number of trades, accumulated from opening.  
<br> 
- `mid_price` : `float`  
The midpoint price calculated as the average of bid and ask prices.  
<br> 
- `market_price` : `float`  
The last traded price.  

***

### TransactionData | TradeData  

`TransactionData` represents transaction data for a specific ticker.<br>
`TradeData` allows initialization with `trade_price` instead of `price` and 
`trade_volume` instead of `volume`.
It provides additional alias properties for these alternate names. <br>
<br>
***SZ market provide canceling in TradeData with `price`, `notional`, `side.sign`, either `buy_id` or `sell_id` = 0, [`FilterMode.no_cancel`](factor_demo.md#hooks-and-callbacks) is used to filter canceling.***<br>
<br>
**Properties**:<br>
- `ticker` : `str`  
The ticker symbol for the transaction data.  
<br> 
- `timestamp` : `float`  
The timestamp of the transaction data.  
<br> 
- `price` : `float`  
The price at which the transaction occurred.  
<br> 
- `volume` : `float | None`  
The volume of the transaction.   
<br> 
- `side` : `TransactionSide`  
The side of the transaction (buy, sell or cancel, `side.sign` 1, -1, 0).   
<br> 
- `multiplier` : `float`  
The multiplier for the transaction.  
<br> 
- `transaction_id` : `int | str | None`  
The identifier for the transaction.  
<br>
- `buy_id` : `int | str | None`  
The identifier for the buying transaction.  
<br> 
- `sell_id` : `int | str | None`  
The identifier for the selling transaction.  
<br> 
- `notional` : `float`  
The notional value of the transaction.  
<br> 
- `market_price` : `float`  
Alias for `price`.  
<br> 
- `flow` : `float`  
The flow of the transaction, calculated as `side.sign` * `volume`.  

***

### OrderData  
Represents order data for a specific ticker.<br>
<br>
***SH market provide canceling in OrderData with `order_type='D'`, [`FilterMode.no_cancel`](factor_demo.md#hooks-and-callbacks) is used to filter canceling.***<br>
<br>
**Properties**:<br>
- `ticker` : `str`  
The ticker symbol for the order data.    
<br> 
- `timestamp` : `float`  
The timestamp of the order data.  
<br> 
- `price` : `float`  
The price at which the order placed.  
<br> 
- `volume` : `float | None`  
The volume of the order.   
<br> 
- `side` : `TransactionSide`  
The side of the order (buy or sell, `side.order_sign` 1 or -1).  
<br> 
- `OrderType` : `OrderType` ## check here  
<br> 
- `order_id` : `int | str`  
Order id, associated with `buy_id` or `sell_id` in `TransactionData`.  

***

### OrderBook
Represents order book for a specific ticker.<br>
<br>
**Properties**:<br>
- `ticker` : `str`  
The ticker symbol for the order book.    
<br> 
- `timestamp` : `float`  
The timestamp of the order book.  
<br> 
- `mid_price` : `float`  
(ask1 price + bid1 price)/2  
<br> 
- `spread` : `float`  
ask1 price - bid1 price  
<br> 
- `spread_pct` : `float`  
`spread`/`mid_price`  
<br> 
- `bid` : `Book`  
- `ask` : `Book`  
`Book` have property `price` and `volume`, which are `list[float]`.  
<br> 
- `best_bid_price` : `float`  
- `best_bid_volume` : `float`  
bid1 price and volume.  
<br> 
- `best_ask_price` : `float`  
- `best_ask_volume` : `float`  
ask1 price and volume.