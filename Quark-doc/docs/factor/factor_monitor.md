# Design a Factor Monitor

These steps should be noted when designing a `FactorMonitor` for customized purpose.

***

### inheritance, multi-inheritance

A factor must inherit from abstract class `FactorMonitor` in `quark.factor.utils`.  
Could inherit from certain samplers class in `quark.factor.sampler` e.g. `FixedIntervalSampler`;  
Could inherit from `EMA` and `Synthetic` in `quark.factor.utils` to composite and transform stock factor to index factor.

***

### signature

No kwargs allowed.  
No advanced data type allowed, only the basic (int/float/str/etc.) and serializable ones (datetime.date is not serializable!)

***

### market data filtering

Use `filter_mode` parameter as mentioned in the [demo](factor_demo.md#hooks-and-callbacks).

***

### requesting external data

Add 'external' field in the `_meta` attribute or `__meta__` section.  
External data will be loaded on BoD process, into `self.contexts['external']`.  

***

### factor value caching

Register yor return value in `factor_name` method.  
Do not hook the calculation process into `.value` property.  Do not assume the `.value` property even be called.  
Calculate the indicator / factor value in `on_triggered` method, if possible. And properly cache the results.    
`.composite()` from `Synthetic` is, however, advised to be called in `.value`  

***

### advanced feature: communicate with other factor

Use shared memory to communicate with other factor.

```python
from quark.base.memory_core import SyncMemoryCore

memory_manager = SyncMemoryCore()

# to set a value into the shared memory
named_vector = memory_manager.register(dtype='NamedVector', name='some_name')
named_vector['a'] = 123

# to access the value is just like the same
a = named_vector['a']
```

Each `NamedVector` can be accessed, from different monitor, as long as they have the same name.
there are several available dtypes for the shared memory access.

- Vector = 'Vector'  
- NamedVector = 'NamedVector'  
- Deque = 'Deque'  
- IntValue = 'IntValue'  
- FloatValue = 'FloatValue'  

***

### advanced feature: serializing and deserializing

Use `to_json` and `update_from_json`

Note that do not override `from_json` method.

***

### advanced feature: logging and profiling

Use `quark.base.telemetrics` module.
