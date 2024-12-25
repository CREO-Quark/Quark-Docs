# A factor demo

A factor demo is presented in `quark-fp/factor_pool/sharpe.py`.  
A typical factor file should include 4 parts:

1. `__meta__`: `name` & `params` are the most importance two.
2. `xxxMonitor` & `xxxAdaptiveMonitor`: class inherited from `FactorMonitor` and certain type of sampler, 
   which do sampling according to timestamp or volume and implement factor calculation process for stocks.
3. `xxxAdaptiveIndexMonitor`: class inherited from `Synthetic`, which composite stock factor to index factor.
4. `main`: optional, added to run in terminal.

We will discuss them in detail.

***

### __meta\__

__meta\__ will be used in scripts in `quark-fp\evaluation\*`. The `name` and
`params` are used to generate list of `FactorMonitor` with different params set.
`comments` should be documented to illustrate how the factor is calculated.

```python
__meta__ = {
    'ver': '0.1.2.alpha3',
    'name': 'IntensityAdaptiveIndexMonitor',
    'params': [
        dict(
            sampling_interval=5,
            sample_size=20,
            name='Monitor.Intensity.Adaptive.Index.0'
        ),
        dict(
            sampling_interval=10,
            sample_size=20,
            name='Monitor.Intensity.Adaptive.Index.1'
        )
    ],
    'family': 'low-pass',
    'market': 'cn',
    'requirements': [
        'quark @ git+https://github.com/BolunHan/Quark.git#egg=Quark',  # this requirement is not necessary, only put there as a demo
        'numpy',  # not necessary too, since the env installed with numpy by default
        'PyAlgoEngine',  # also not necessary
    ],
    'external_data': [],  # topic for required external data
    'external_lib': [],  # other library, like c compiled lib file.
    'factor_type': 'basic',
    'activated_date': None,
    'deactivated_date': None,
    'dependencies': [],
    'comments': """
    This is a demo for how to properly define a factor.
    """
}
```

***

### ArbitrarySampler

To use a sampler, a `FactorMonitor` class must have the following parts

#### inherit and config the sampler

Several samplers are provided in `quark.factor.sampler`, e.g. `FixedIntervalSampler`, `FixedVolumeSampler`, `VolumeProfileSampler`, which are subclass of abstract class `MarketDataSampler`.  
To use a certain type of sampler, a factor should inherit from both `FactorMonitor` and the SamplerClass with needed signature.  
Additional information about sampler is [here](sampler.md).

```python
class IntensityMonitor(FactorMonitor, FixedIntervalSampler):
    def __init__(self, sampling_interval: float, sample_size: int, name: str = 'Monitor.Intensity', monitor_id: str = None):
        super().__init__(name=name, monitor_id=monitor_id, filter_mode=FilterMode.no_cancel|FilterMode.no_auction)
        FixedIntervalSampler.__init__(self=self, sampling_interval=sampling_interval, sample_size=sample_size)
```
```python
class IntensityAdaptiveMonitor(IntensityMonitor, VolumeProfileSampler):
    def __init__(self, sampling_interval: float, sample_size: int = 20, name: str = 'Monitor.Intensity.Adaptive', monitor_id: str = None):
        super().__init__(
            sampling_interval=sampling_interval,
            sample_size=sample_size,
            name=name,
            monitor_id=monitor_id
        )
        VolumeProfileSampler.__init__(
            self=self,
            sampling_interval=sampling_interval,
            sample_size=sample_size,
            profile_type=VolumeProfileType.interval_volume
        )
```

#### register & clear topic

When using sampler, some specific topics must be registered with `register_sampler` method before use in `__init__` method.  
```python
class IntensityMonitor(FactorMonitor, FixedIntervalSampler):
    def __init__(self, sampling_interval: float, sample_size: int, name: str = 'Monitor.Intensity', monitor_id: str = None):
        
        ...
       
        self.register_sampler(topic='price', mode=SamplerMode.update)
        self.register_sampler(topic='notional', mode=SamplerMode.accumulate)
        self.register_sampler(topic='buy_price', mode=SamplerMode.update)
        self.register_sampler(topic='buy_notional', mode=SamplerMode.accumulate)
        self.register_sampler(topic='sell_price', mode=SamplerMode.update)
        self.register_sampler(topic='sell_notional', mode=SamplerMode.accumulate)
```

`clear` method should be overridden to re-register topics of the monitor.
```python
class IntensityMonitor(FactorMonitor, FixedIntervalSampler):
    def clear(self):
        super().clear()
   
        # re-register topics
        self.register_sampler(topic='price', mode=SamplerMode.update)
        self.register_sampler(topic='notional', mode=SamplerMode.accumulate)
        self.register_sampler(topic='buy_price', mode=SamplerMode.update)
        self.register_sampler(topic='buy_notional', mode=SamplerMode.accumulate)
        self.register_sampler(topic='sell_price', mode=SamplerMode.update)
        self.register_sampler(topic='sell_notional', mode=SamplerMode.accumulate)
```

#### hooks and callbacks

The process of factor calculation is:  
1. `MarketData` comes  
2. filter `MarketData` if needed with `FilterMode`  
3. `accumulate_volume` if using volume scale sampler  
4. check if timestamp or volume reach next obs in `log_obs`  
5. if reached, enroll the obs in `history` and calculate factor with `on_triggered`  

- `FilterMode`  
   Basic filter for market data. There are 5 `enum.auto()` types,
   including `no_cancal`, `no_auction`, `no_order`, `no_trade`, `no_tick`.
   Filters can be added using `FilterMode.no_cancal|FilterMode.no_auction` in `__init__`.

```python
class IntensityMonitor(FactorMonitor, FixedIntervalSampler):
   def __init__(self, sampling_interval: float, sample_size: int, name: str = 'Monitor.Intensity', monitor_id: str = None):
       super().__init__(name=name, monitor_id=monitor_id, filter_mode=FilterMode.no_cancel | FilterMode.no_auction)
```
- `accumulate_volume`  
   When using volume scale sampler, i.e. `FixedVolumeSampler` and `VolumeProfileSampler`, `accumulate_volume` should be 
   hooked in `on_tick_data` or `on_trade_data`, and it doesn't need to be associated with factor calculation.  
   **`on_trade_data` will be more precise than `on_tick_data`**.  
   NOT hook `accumulate_volume` in `on_market_data`, volume from tick data will override volume from trade data.

```python
class IntensityAdaptiveMonitor(IntensityMonitor, VolumeProfileSampler):
    def on_trade_data(self, trade_data: TradeData | TransactionData, **kwargs):
       self.accumulate_volume(market_data=trade_data)
```

- `log_obs`  
   When certain type of `MarketData` comes, send all registered observations at the same time to the sampler is advised.  
   `log_obs` can be hooked in either `on_xxx_data` or `on_market_data` with filter.  
```python
class IntensityMonitor(FactorMonitor, FixedIntervalSampler):
    def on_trade_data(self, trade_data: TradeData | TransactionData, **kwargs):
       ticker = trade_data.ticker
       timestamp = trade_data.timestamp
       price = trade_data.price
       side = trade_data.side
       volume = trade_data.volume

       if side.value == 1:
           self.log_obs(ticker=ticker, timestamp=timestamp, price=price, notional=price * volume, buy_price=price, buy_notional=price * volume)
       elif side.value == -1:
           self.log_obs(ticker=ticker, timestamp=timestamp, price=price, notional=price * volume, sell_price=price, sell_notional=price * volume)
       else:
           pass
```
   or
```python
class IntensityMonitor(FactorMonitor, FixedIntervalSampler):
    def on_market_data(self, market_data: MarketData, **kwargs):
       if not isinstance(market_data, TradeData | TransactionData):
          return
       ticker = market_data.ticker
       timestamp = market_data.timestamp
       price = market_data.price
       side = market_data.side
       volume = market_data.volume

       if side.value == 1:
           self.log_obs(ticker=ticker, timestamp=timestamp, price=price, notional=price * volume, buy_price=price, buy_notional=price * volume)
       elif side.value == -1:
           self.log_obs(ticker=ticker, timestamp=timestamp, price=price, notional=price * volume, sell_price=price, sell_notional=price * volume)
       else:
           pass
```
  
- `on_triggered`  
   Calculate factor in `on_triggered` and cache the results.  
   All registered topics will call `on_triggered`, it's advised to filter needed topics in calculation.  
   If other registered topics is needed, can use `self.get_history(topic,ticker)` to get the data.
```python
class IntensityMonitor(FactorMonitor, FixedIntervalSampler):
   def on_triggered(self, ticker, topic, sampler, **kwargs):
       if topic == 'buy_price':
           self._intensity_buy[ticker] = self.calculate_intensity(price=sampler.history.get(ticker))
       elif topic == 'sell_price':
           self._intensity_sell[ticker] = self.calculate_intensity(price=sampler.history.get(ticker))
       elif topic == 'price':
           self._intensity_all[ticker] = self.calculate_intensity(price=sampler.history.get(ticker))
```
  

#### sampler readiness
   `VolumeProfileSampler` need fitting for volume prediction model. `is_ready` property should be overridden with correct readiness flag if using this sampler.
```python
class IntensityAdaptiveMonitor(IntensityMonitor, VolumeProfileSampler):
   @property
   def is_ready(self) -> bool:
       return self.profile_ready
```

***

### Synthetic

Build the class `xxxAdaptiveIndexMonitor` to composite stock factor to index factor.  
<br>
This class inherit from the previous class which calculate stock factor and `Synthetic` from `quark.factor.utils`.  
```python
class IntensityAdaptiveIndexMonitor(IntensityAdaptiveMonitor, Synthetic):
    def __init__(self, sampling_interval: float, sample_size: int, weights: dict[str, float] = None, name: str = 'Monitor.Intensity.Adaptive.Index', monitor_id: str = None):
        super().__init__(
            sampling_interval=sampling_interval,
            sample_size=sample_size,
            name=name,
            monitor_id=monitor_id
        )

        Synthetic.__init__(self=self, weights=weights)
```
This class override `__call__` to filter MarketData with tick in weights.  
```python
class IntensityAdaptiveIndexMonitor(IntensityAdaptiveMonitor, Synthetic):
    def __call__(self, market_data: MarketData, **kwargs):
        ticker = market_data.ticker

        if self.weights and ticker not in self.weights:
            return

        super().__call__(market_data=market_data, **kwargs)
```
This class must override `factor_names` to cache factor value, and `value` with method `composite` to calculate index factor based on weighted stock factor.  
Note that `.value` can be multi-dimension. There are 3 dimension `intensity.buy`, `intensity.sell`, `intensity.all` in the demo.
```python
class IntensityAdaptiveIndexMonitor(IntensityAdaptiveMonitor, Synthetic):
    def factor_names(self, subscription: list[str]) -> list[str]:
        return [
            f'{self.name.removeprefix("Monitor.")}.intensity.buy',
            f'{self.name.removeprefix("Monitor.")}.intensity.sell',
            f'{self.name.removeprefix("Monitor.")}.intensity.all'
        ]

    @property
    def value(self) -> dict[str, float]:
        intensity_b = self._intensity_buy
        intensity_s = self._intensity_sell
        intensity_a = self._intensity_all

        return {
            'intensity.buy': self.composite(values=intensity_b),
            'intensity.sell': self.composite(values=intensity_s),
            'intensity.all': self.composite(values=intensity_a),
        }
```

***

**There are other things should be noted when designing a `FactorMonitor` for customized purpose. See [here](factor_monitor.md).**

***

### main

The `main` can be used to validate the factor.  
The `nodes` param can be set to validate one factor or multi factors together.  
The `config_overrides` param will override `config.ini` in CWD.

***

### Full demo

```python
import numpy as np
from algo_engine.base import MarketData, TradeData, TransactionData, TickData
from quark.base import LOGGER
from quark.factor.utils import FilterMode
from quark.factor import SamplerMode, Synthetic, FactorMonitor, VolumeProfileSampler, FixedIntervalSampler, VolumeProfileType
__meta__ = {
    'ver': '0.1.2.alpha3',
    'name': 'IntensityAdaptiveIndexMonitor',
    'params': [
        dict(
            sampling_interval=5,
            sample_size=20,
            name='Monitor.Intensity.Adaptive.Index.0'
        ),
        dict(
            sampling_interval=10,
            sample_size=20,
            name='Monitor.Intensity.Adaptive.Index.1'
        )
    ],
    'family': 'low-pass',
    'market': 'cn',
    'requirements': [
        'quark @ git+https://github.com/BolunHan/Quark.git#egg=Quark',  # this requirement is not necessary, only put there as a demo
        'numpy',  # not necessary too, since the env installed with numpy by default
        'PyAlgoEngine',  # also not necessary
    ],
    'external_data': [],  # topic for required external data
    'external_lib': [],  # other library, like c compiled lib file.
    'factor_type': 'basic',
    'activated_date': None,
    'deactivated_date': None,
    'dependencies': [],
    'comments': """
    This is a demo for how to properly define a factor.
    """
}


class IntensityMonitor(FactorMonitor, FixedIntervalSampler):
    def __init__(self, sampling_interval: float, sample_size: int, name: str = 'Monitor.Intensity', monitor_id: str = None):
        super().__init__(name=name, monitor_id=monitor_id, filter_mode=FilterMode.no_cancel|FilterMode.no_auction)
        FixedIntervalSampler.__init__(self=self, sampling_interval=sampling_interval, sample_size=sample_size)

        self._intensity_buy: dict[str, float] = {}
        self._intensity_sell: dict[str, float] = {}
        self._intensity_all: dict[str, float] = {}

        self.register_sampler(topic='price', mode=SamplerMode.update)
        self.register_sampler(topic='notional', mode=SamplerMode.accumulate)
        self.register_sampler(topic='buy_price', mode=SamplerMode.update)
        self.register_sampler(topic='buy_notional', mode=SamplerMode.accumulate)
        self.register_sampler(topic='sell_price', mode=SamplerMode.update)
        self.register_sampler(topic='sell_notional', mode=SamplerMode.accumulate)

    def on_tick_data(self, tick_data: TickData, **kwargs):
        x = tick_data.total_traded_notional

    def on_trade_data(self, trade_data: TradeData | TransactionData, **kwargs):
        ticker = trade_data.ticker
        timestamp = trade_data.timestamp
        price = trade_data.price
        side = trade_data.side
        volume = trade_data.volume

        if side.value == 1:
            self.log_obs(ticker=ticker, timestamp=timestamp, price=price, notional=price * volume, buy_price=price, buy_notional=price * volume)
        elif side.value == -1:
            self.log_obs(ticker=ticker, timestamp=timestamp, price=price, notional=price * volume, sell_price=price, sell_notional=price * volume)
        else:
            pass

    def calculate_intensity(self, price, epsilon: float=1e-8) -> float:
        price_vector = np.array(price)

        if len(price_vector) < 3:
            return np.nan

        price_vector = price_vector[:-1]
        price_pct_vector = np.diff(price_vector) / price_vector[:-1]
        mean = np.mean(price_pct_vector)
        std = np.std(price_pct_vector)
        return np.divide(mean, std + epsilon)

    def on_triggered(self, ticker, topic, sampler, **kwargs):
        if topic == 'buy_price':
            self._intensity_buy[ticker] = self.calculate_intensity(price=sampler.history.get(ticker))
        elif topic == 'sell_price':
            self._intensity_sell[ticker] = self.calculate_intensity(price=sampler.history.get(ticker))
        elif topic == 'price':
            self._intensity_all[ticker] = self.calculate_intensity(price=sampler.history.get(ticker))

    def factor_names(self, subscription: list[str]) -> list[str]:
        return [
            f'{self.name.removeprefix("Monitor.")}.{ticker}' for ticker in subscription
        ]

    def clear(self):
        super().clear()

        # re-register topics
        self.register_sampler(topic='price', mode=SamplerMode.update)
        self.register_sampler(topic='notional', mode=SamplerMode.accumulate)
        self.register_sampler(topic='buy_price', mode=SamplerMode.update)
        self.register_sampler(topic='buy_notional', mode=SamplerMode.accumulate)
        self.register_sampler(topic='sell_price', mode=SamplerMode.update)
        self.register_sampler(topic='sell_notional', mode=SamplerMode.accumulate)

    @property
    def value(self) -> dict[str, float] | float:
        return self._intensity_all


class IntensityAdaptiveMonitor(IntensityMonitor, VolumeProfileSampler):
    def __init__(self, sampling_interval: float, sample_size: int = 20, name: str = 'Monitor.Intensity.Adaptive', monitor_id: str = None):
        super().__init__(
            sampling_interval=sampling_interval,
            sample_size=sample_size,
            name=name,
            monitor_id=monitor_id
        )
        # for profile_type = VolumeProfileType.interval_volume or VolumeProfileType.accumulative_volume
        VolumeProfileSampler.__init__(
            self=self,
            sampling_interval=sampling_interval,
            sample_size=sample_size,
            profile_type=VolumeProfileType.interval_volume
        )
        # for profile_type = VolumeProfileType.simple_online
        # VolumeProfileSampler.__init__(
        #     self=self,
        #     sampling_interval=sampling_interval,
        #     sample_size=sample_size,
        #     n_window = 100,
        #     profile_type=VolumeProfileType.simple_online
        # )

    def on_trade_data(self, trade_data: TradeData | TransactionData, **kwargs):
        self.accumulate_volume(market_data=trade_data)
        super().on_trade_data(trade_data=trade_data, **kwargs)

    @property
    def is_ready(self) -> bool:
        return self.profile_ready


class IntensityAdaptiveIndexMonitor(IntensityAdaptiveMonitor, Synthetic):
    def __init__(self, sampling_interval: float, sample_size: int, weights: dict[str, float] = None, name: str = 'Monitor.Intensity.Adaptive.Index', monitor_id: str = None):
        super().__init__(
            sampling_interval=sampling_interval,
            sample_size=sample_size,
            name=name,
            monitor_id=monitor_id
        )

        Synthetic.__init__(self=self, weights=weights)

    def __call__(self, market_data: MarketData, **kwargs):
        ticker = market_data.ticker

        if self.weights and ticker not in self.weights:
            return

        super().__call__(market_data=market_data, **kwargs)

    def factor_names(self, subscription: list[str]) -> list[str]:
        return [
            f'{self.name.removeprefix("Monitor.")}.intensity.buy',
            f'{self.name.removeprefix("Monitor.")}.intensity.sell',
            f'{self.name.removeprefix("Monitor.")}.intensity.all'
        ]

    @property
    def value(self) -> dict[str, float]:
        intensity_b = self._intensity_buy
        intensity_s = self._intensity_sell
        intensity_a = self._intensity_all

        return {
            'intensity.buy': self.composite(values=intensity_b),
            'intensity.sell': self.composite(values=intensity_s),
            'intensity.all': self.composite(values=intensity_a),
        }


def main():
    from quark.base import safe_exit
    from quark_validator.factor.evaluation import FactorTree, FactorNode, eval_tree

    factor_tree = FactorTree(
        nodes=[
            FactorNode(name='Monitor.Intensity.Adaptive.Index.0', file=__file__),
            FactorNode(name='Monitor.Intensity.Adaptive.Index.1', file=__file__)
        ],
        config_overrides={
            'Datalore': {
                "FEE_RATE": 0.00032,
                "ALPHA": 0.1,
                "POLY_DEGREE": 2,
                "Calibration": {
                    "pct_change_900": {
                        "optimizer": "SLSQP",
                        "tol": 1e-10,
                        "l1": 0.01,
                        "l2": 0.01,
                        "balanced_label": True
                    },
                    "state_3": {
                        "optimizer": "SLSQP",
                        "tol": 1e-07,
                        "l1": 0.001,
                        "l2": 0.001,
                        "balanced_label": True
                    },
                    "up_actual_3": {
                        "optimizer": "SLSQP",
                        "tol": 1e-06
                    },
                    "down_actual_3": {
                        "optimizer": "SLSQP",
                        "tol": 1e-06
                    },
                    "target_actual_3": {
                        "optimizer": "SLSQP",
                        "tol": 1e-10,
                        "l1": 0.0001,
                        "l2": 0.0001,
                        "balanced_label": True
                    },
                    "up_smoothed_3": {
                        "optimizer": "SLSQP",
                        "tol": 1e-06
                    },
                    "down_smoothed_3": {
                        "optimizer": "SLSQP",
                        "tol": 1e-06
                    },
                    "target_smoothed_3": {
                        "optimizer": "SLSQP",
                        "tol": 1e-10,
                        "l1": 0.0001,
                        "l2": 0.0001,
                        "balanced_label": True
                    }
                }
            }
        }
    )

    eval_tree(factor_tree)
    safe_exit()


if __name__ == '__main__':
    main()

```
