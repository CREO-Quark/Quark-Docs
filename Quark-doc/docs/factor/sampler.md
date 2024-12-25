# Sampler

When calculating factors, we always need to do sampling. 
The most basic sampling is to sample at the same time interval, 
such as generating candlestick chart; 
considering that the information density is different on timescale, 
we can also sample at the same volume or volatility.  
    
Currently, `Quark` provides `Sampler` on time scale and volume scale,
supporting various aggregation modes by `SamplerMode`.

***

### SamplerMode
`SamplerMode` describes how to calculate sampled data from multiple 
active observations between two snapshots, e.g. price can be `update` and 
volume can be `accumulate`. 

Supported `SamplerMode` are:<br>
- `update`  
- `first`  
- `accumulate`  
- `min`  
- `max`   
- `median`   
- `mean`    
- `variance`    
- `store` : store the raw data, without any calculation    

***

### SamplerAbstractClass

a sampler provides following method:

- Required: `register_sampler(topic, mode)` for topic registration
- Required: `log_obs(...)` to log and process observation \
- Optional: `on_triggered()` a callback function for triggered sampler.
- and some utility function like `get_history`, `get_latest`, `get_active` etc.

check the [demo](factor_demo.md) for how to use the sampler

***

### Temporal Scale Sampler

#### quark.factor.FixedIntervalSampler

```python
class quark.factor.FixedIntervalSampler(sampling_interval: float = 1., sample_size: int = 60)
```
> Class for a fixed temporal interval sampler.  

> **Parameters**:    
> - `sampling_interval` : `float`  
> Time interval between consecutive samples, in seconds. Default is 1 second.    
> - `sample_size` : `int`  
> Number of samples to store in the sampler. Default is 60 samples.    

> **Notes**:    
> - `sampling_interval` must be a positive value; otherwise, a warning is issued.    
> - `sample_size` must be greater than 2 according to Shannon's Theorem.    
> - The sampler is NOT guaranteed to be triggered in a fixed interval, if corresponding market data is missing (e.g. in a melt down event or hit the upper bound limit.)      
> To partially address this issue, subscribe TickData (market snapshot) from exchange and use it to update the sampler.    

***

### Volume Scale Sampler

#### quark.factor.FixedVolumeIntervalSampler

```python
class quark.factor.FixedVolumeIntervalSampler(sampling_interval: float = 100., sample_size: int = 20)
```
> Class for a fixed volume interval sampler.    

> **Parameters**:    
> - `sampling_interval` : `float`    
> Volume interval between consecutive samples. Default is 100.    
> - `sample_size` : `int`     
> Number of samples to store in the sampler. Default is 20.    

> **Methods**:   
```python
accumulate_volume(ticker: str = None, volume: float = 0., market_data: MarketData = None, use_notional: bool = False)
```
> Accumulates volume based on market data or explicit ticker and volume, `use_notional` can be modified.    

> **Notes**:    
> - The `sampling_interval` is in **shares**.     
> - The same fixed volume interval for all ticker.    

***

#### quark.factor.VolumeProfileSampler
```python
class quark.factor.VolumeProfileSampler(sampling_interval: float = 60.0, sample_size: int = 20, profile_type: VolumeProfileType = 'simple_online', use_notional: bool = True, **kwargs)
```
> Class for adaptive volume interval sampler.

> Three cases for `profile_type`  
> 
> 1. `profile_type = 'simple_online'`  
> Starting from market open, use `n_window * sampling_interval` seconds of data 
> to estimate a baseline volume for each ticker within `sampling_interval` seconds,
> and use the baseline volume for sampling.  
> ***This method will over-estimate numbers of sampling.***
> > **Parameters**:  
> > - `sampling_interval` : `float`  
> > Target temporal interval between consecutive samples. Default is 60.    
> > - `sample_size` : `int`  
> > Number of samples to store in the sampler. Default is 20.    
> > - `use_notional` : `bool`  
> > Estimate baseline use notional or volume.    
> > - `n_window` : `int`  
> > Number of samples to estimate baseline<br>   
>   
> 2. `profile_type = 'accumulative_volume'`  
> Before the market opens, a prediction model for the **full-day trading volume**
> is established based on the market data of the past several days. 
> Combined with the current market data, a Bayesian update is performed 
> on the full-day trading volume. The sampling volume is determined 
> based on the current time and `sampling_interval`.
> > **Parameters**:  
> > - `sampling_interval` : `float`  
> > Target temporal interval between consecutive samples. Default is 60.    
> > - `sample_size` : `int`  
> > Number of samples to store in the sampler. Default is 20.    
> > - `use_notional` : `bool`  
> > Estimate baseline use notional or volume.
>   
> 3. `profile_type = 'interval_volume'`  
> Before the market opens, a prediction model for the **sampling interval volume**
> is established based on the market data of the past several days. 
> Combined with the current market data, a Bayesian update is performed 
> on the sampling interval volume, which will be used for sampling.
> > **Parameters**:  
> > - `sampling_interval` : `float`  
> > Target temporal interval between consecutive samples. Default is 60.    
> > - `sample_size` : `int`  
> > Number of samples to store in the sampler. Default is 20.    
> > - `use_notional` : `bool`  
> > Estimate baseline use notional or volume.

