# How to validate factors

Factor validation task is defined in `quark_validator/factor/validation.py`.

A simple validation script is like:
```python
import os
import datetime
import time
from quark.factor import IndexWeight
from quark.profile import profile_cn_override
from quark_validator.factor import validation
from factor_pool.sharpe import IntensityAdaptiveIndexMonitor
from factor_pool.auxiliary import TrendIndexAdaptiveAuxiliaryMonitor

profile_cn_override() # cn trading time etc.

os.environ['TZ'] = 'Asia/Shanghai'
time.tzset()

task = validation.ValidationTask(
    start_date=datetime.date(2024,1,1),
    end_date=datetime.date(2024,2,1),
    index_name='000016.SH',
    index_weights=IndexWeight(index_name='000016.SH'),
    dtype=['TickData', 'TransactionData', 'OrderData'], # depend on which types of data are required in factor calculation 
    override_cache=True, # true to re-calculate factor value, otherwise load from database
    dev_pool=True, # store into dev_pool or prod_pool, must be true
    validator=validation.InterTemporalValidation, # 2 types, details below
    factor=[
        IntensityAdaptiveIndexMonitor(sampling_interval=10, sample_size=20, weights=validation.INDEX_WEIGHTS, baseline_window=100, aligned_interval=False, name='Monitor.Intensity.Adaptive.Index.1'),
        IntensityAdaptiveIndexMonitor(sampling_interval=15, sample_size=20, weights=validation.INDEX_WEIGHTS, baseline_window=100, aligned_interval=False, name='Monitor.Intensity.Adaptive.Index.2'),
    ], # factor monitor instances
    pred_var=[
        'pct_change_900','state_3',
        'up_actual_3','down_actual_3','target_actual_3',
        'up_smoothed_3', 'down_smoothed_3','target_smoothed_3'
    ], # pred_var, details below
    pred_target='IH_MAIN',
    sampling_interval=10, # interval in seconds for synthetic index
    poly_degree=2, # degree of polynomial features for regression analysis
    training_days=5, # using prior n-day data for model training
    auxiliary_factor=[
        TrendIndexAdaptiveAuxiliaryMonitor(sampling_interval=5,sample_size=20,alpha=0.0739,baseline_window=100,aligned_interval=False,name='Monitor.Aux.Adaptive.Index.0')
    ] # only used when validator=validation.FactorParamsOptimizer, details below
)

task.run()
validation.safe_exit(0)
```

Detail about certain params is as below.  

***


### validator
There are 2 options for validator, defined in `quark_validator/factor/validation.py`:  

* InterTemporalValidation:  
    Model is trained with ***X*** = all factor monitors defined 
    in `factors` with their `poly_degree` interaction terms. The
    model uses `traning_day` days' data prior calibration date.<br>
    <br>
* FactorParamsOptimizer:  
    For all factors defined in `factors`, calculate metrics using
    cross validation, and select the best parameter set which will
    be used in the next day.  
    Parameter set is selected based on an ema(alpha=0.5) of previous
    **average** metrics.  
    New `auxiliary_factor` param is used to evaluate factors' incremental 
    effects on metrics beyond aux factors. `TrendIndexAdaptiveAuxiliaryMonitor`
    is a momentum factor for fitting factors having no predictive power on trend.

***

### pred_var

There are 8 basic topics and many extended topics, which are defined in `quark/calibration/future.py`:

* `pct_change_900`: 900s/60=15min ret
* `state_3`: recursive decoding state of level3
* `up_actual_3`: up ret of level3
* `down_actual_3`: down ret of level3
* `target_actual_3`: up & down ret of level3
* `up_smoothed_3`: up ret of level3 considered potential loss
* `down_smoothed_3`: down ret of level3 considered potential loss
* `target_smoothed_3`: up & down ret of level3 considered potential loss

Extended topics include `pct_change_[300|900|1800|3600]`,`state_[2|3|4]` and `up_actual_[2|3|4]`...

***

### config.ini

There are also some validation config can be tuned in sections named 
after `[Datalore]` in `runtime/config.ini`. Different params are allowed
for different `pred_var`, which can be tuned in sub sections.
    
Here are some worth mentioning:     

- `l1` & `l2`: for regression regularization.
- `optimizer`: methods in [`scipy.optimize.minimize`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.minimize.html#minimize)
and solvers in [`CVXPY`](https://www.cvxpy.org/tutorial/solvers/index.html), and `Adam`.
- `balanced_label`: whether to adjust weight for y_pos and y_neg, available for `pct_change_900`, `state_3`, `target_actual_3` and `target_smoothed_3`.
- `avoid_quantile_crossing`: true to avoid fitted quantile crossover
- `tol`: tolerance for scipy and cvxpy optimizer  
  <br>
  below are config for `Adam`:  

- `enable_lr_scheduler`
- `enable_diagnose`
- `lr`
- `validation_split`
- `patience`