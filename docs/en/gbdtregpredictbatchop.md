## Description
Gradient Boosting(often abbreviated to GBDT or GBM) is a popular supervised learning model.
 It is the best off-the-shelf supervised learning model for a wide range of problems,
 especially problems with medium to large data size.
 
 This implementation use histogram-based algorithm.
 See:
 "Mcrank: Learning to rank using multiple classification and gradient boosting", Ping Li et al., NIPS 2007,
 for detail and experiments on histogram-based algorithm.
 
 This implementation use layer-wise tree growing strategy,
 rather than leaf-wise tree growing strategy
 (like the one in "Lightgbm: A highly efficient gradient boosting decision tree", Guolin Ke et al., NIPS 2017),
 because we found the former being faster in flink-based distributed computing environment.
 
 This implementation use data-parallel algorithm.
 See:
 "A communication-efficient parallel algorithm for decision tree", Qi Meng et al., NIPS 2016
 for an introduction on data-parallel, feature-parallel, etc., algorithms to construct decision forests.

## Parameters
| Name | Description | Type | Required？ | Default Value |
| --- | --- | --- | --- | --- |
| predictionCol | Column name of prediction. | String | ✓ |  |
| predictionDetailCol | Column name of prediction result, it will include detailed info. | String |  |  |
| reservedCols | Names of the columns to be retained in the output table | String[] |  | null |


## Script Example
#### Script
```python
import numpy as np
import pandas as pd
from pyalink.alink import *


def exampleData():
    return np.array([
        [1.0, "A", 0, 0, 0],
        [2.0, "B", 1, 1, 0],
        [3.0, "C", 2, 2, 1],
        [4.0, "D", 3, 3, 1]
    ])


def sourceFrame():
    data = exampleData()
    return pd.DataFrame({
        "f0": data[:, 0],
        "f1": data[:, 1],
        "f2": data[:, 2],
        "f3": data[:, 3],
        "label": data[:, 4]
    })


def batchSource():
    return dataframeToOperator(
        sourceFrame(),
        schemaStr='''
    f0 double, 
    f1 string, 
    f2 int, 
    f3 int, 
    label int
    ''',
        op_type='batch'
    )


def streamSource():
    return dataframeToOperator(
        sourceFrame(),
        schemaStr='''
    f0 double, 
    f1 string, 
    f2 int, 
    f3 int, 
    label int
    ''',
        op_type='stream'
    )


trainOp = (
    GbdtRegTrainBatchOp()
    .setLearningRate(1.0)
    .setNumTrees(3)
    .setMinSamplesPerLeaf(1)
    .setLabelCol('label')
    .setFeatureCols(['f0', 'f1', 'f2', 'f3'])
)

predictBatchOp = (
    GbdtRegPredictBatchOp()
    .setPredictionCol('pred')
)

(
    predictBatchOp
    .linkFrom(
        batchSource().link(trainOp),
        batchSource()
    )
    .print()
)

predictStreamOp = (
    GbdtRegPredictStreamOp(
        batchSource().link(trainOp)
    )
    .setPredictionCol('pred')
)

(
    predictStreamOp
    .linkFrom(
        streamSource()
    )
    .print()
)

StreamOperator.execute()
```
#### Result
Batch prediction
```
    f0 f1  f2  f3  label  pred
0  1.0  A   0   0      0   0.0
1  2.0  B   1   1      0   0.0
2  3.0  C   2   2      1   1.0
3  4.0  D   3   3      1   1.0
```

