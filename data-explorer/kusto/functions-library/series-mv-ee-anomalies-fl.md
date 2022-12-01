---
title: series_mv_ee_anomalies_fl() - Azure Data Explorer
description: Learn how to use the series_mv_ee_anomalies_fl() user-defined function in Azure Data Explorer.
ms.reviewer: adieldar
ms.topic: reference
ms.date: 11/20/2022
---
# series_mv_ee_anomalies_fl()

The function `series_mv_ee_anomalies_fl()` detects multivariate anomalies in series by applying [elliptic envelope model from scikit-learn](https://scikit-learn.org/stable/modules/generated/sklearn.covariance.EllipticEnvelope.html). This model assumes that the source of the multivariate data is multi-dimensional normal distribution. The function accepts a set of series as numerical dynamic arrays, the names of the features columns and the expected percentage of anomalies out of the whole series. The function builds a multi-dimensional elliptical envelope for each series and marks the points that fall outside this normal envelope as anomalies.

> [!NOTE]
>
> * `series_mv_ee_anomalies_fl()` is a [user-defined function](../query/functions/user-defined-functions.md). For more information, see [usage](#usage).
> * This function contains inline Python and requires [enabling the python() plugin](../query/pythonplugin.md#enable-the-plugin) on the cluster.

## Syntax

`T | invoke series_mv_ee_anomalies_fl(`*features_cols*`,` *anomaly_col*`,` *anomalies_pct*`)`
  
## Arguments

| Name | Type | Required | Description |
|--|--|--|--|
| *features_cols* | dynamic | &check; | Dynamic array containing the names of the columns that are used for the multivariate anomaly detection model |
| *anomaly_col* | string | &check; | The name of the column to store the detected anomalies. |
| *anomalies_pct* | real | | A real number in the range [0-50] specifying the expected percentage of anomalies in the data. Default value: 4% |

## Usage

`series_mv_ee_anomalies_fl()` is a user-defined function [tabular function](../query/functions/user-defined-functions.md#tabular-function), to be applied using the [invoke operator](../query/invokeoperator.md). You can either embed its code in your query, or install it in your database. There are two usage options: ad hoc and persistent usage. See the below tabs for examples.

### [Ad hoc](#tab/adhoc)

For ad hoc usage, embed its code using [let statement](../query/letstatement.md). No permission is required.

<!-- csl: https://help.kusto.windows.net/Samples -->
```kusto
// Define function
let series_mv_ee_anomalies_fl=(tbl:(*), features_cols:dynamic, anomaly_col:string, anomalies_pct:real=4.0)
{
    let kwargs = bag_pack('features_cols', features_cols, 'anomaly_col', anomaly_col, 'anomalies_pct', anomalies_pct);
    let code = ```if 1:
        from sklearn.covariance import EllipticEnvelope
        features_cols = kargs['features_cols']
        anomaly_col = kargs['anomaly_col']
        anomalies_pct = kargs['anomalies_pct']
        dff = df[features_cols]
        ellipsoid = EllipticEnvelope(contamination=anomalies_pct/100.0)
        for i in range(len(dff)):
            dffi = dff.iloc[[i], :]
            dffe = dffi.explode(features_cols)
            ellipsoid.fit(dffe)
            df.loc[i, anomaly_col] = (ellipsoid.predict(dffe) < 0).astype(int).tolist()
        result = df
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
}
;
// Usage
normal_2d_with_anomalies
| extend anomalies=dynamic(null)
| invoke series_mv_ee_anomalies_fl(pack_array('x', 'y'), 'anomalies')
| extend anomalies=series_multiply(80, anomalies)
| render timechart
```

### [Persistent](#tab/persistent)

For persistent usage, use [`.create function`](../management/create-function.md).  Creating a function requires [database user permission](../management/access-control/role-based-authorization.md).

### One time installation

<!-- csl: https://help.kusto.windows.net/Samples -->
~~~kusto
.create-or-alter function with (folder = "Packages\\Series", docstring = "Anomaly Detection for multi dimensional normally distributed data using elliptical envelope model")
series_mv_ee_anomalies_fl(tbl:(*), features_cols:dynamic, anomaly_col:string, anomalies_pct:real=4.0)
{
    let kwargs = bag_pack('features_cols', features_cols, 'anomaly_col', anomaly_col, 'anomalies_pct', anomalies_pct);
    let code = ```if 1:
        from sklearn.covariance import EllipticEnvelope
        features_cols = kargs['features_cols']
        anomaly_col = kargs['anomaly_col']
        anomalies_pct = kargs['anomalies_pct']
        dff = df[features_cols]
        ellipsoid = EllipticEnvelope(contamination=anomalies_pct/100.0)
        for i in range(len(dff)):
            dffi = dff.iloc[[i], :]
            dffe = dffi.explode(features_cols)
            ellipsoid.fit(dffe)
            df.loc[i, anomaly_col] = (ellipsoid.predict(dffe) < 0).astype(int).tolist()
        result = df
    ```;
    tbl
    | evaluate python(typeof(*), code, kwargs)
}
~~~

### Usage

<!-- csl: https://help.kusto.windows.net/Samples -->
```kusto
normal_2d_with_anomalies
| extend anomalies=dynamic(null)
| invoke series_mv_ee_anomalies_fl(pack_array('x', 'y'), 'anomalies')
| extend anomalies=series_multiply(80, anomalies)
| render timechart
```

---

The table normal_2d_with_anomalies contains a set of 3 time series. Each time series has two-dimensional normal distribution with daily anomalies added at midnight, 8am, and 4pm respectively. You can create this sample data set using [an example query](series-mv-ee-anomalies-fl.md#create-a-sample-data-set).

![Graph showing multivariate anomalies on a time chart.](images/series-mv-ee-anomalies-fl/mv-ee-anomalies-time-chart.png)

To view the data as a scatter chart, replace the usage code with the following:

```kusto
normal_2d_with_anomalies
| extend anomalies=dynamic(null)
| invoke series_mv_ee_anomalies_fl(pack_array('x', 'y'), 'anomalies')
| where name == 'TS1'
| project x, y, anomalies
| mv-expand x to typeof(real), y to typeof(real), anomalies to typeof(string)
| render scatterchart with(series=anomalies)
```

![Graph showing multivariate anomalies on a scatter chart.](images/series-mv-ee-anomalies-fl/mv-ee-anomalies-scatter-chart.png)

You can see that on TS1 most of the midnight anomalies were detected using this multivariate model.

### Create a sample data set

<!-- csl: https://help.kusto.windows.net/Samples -->
```kusto
.set normal_2d_with_anomalies <|
//
let window=14d;
let dt=1h;
let n=toint(window/dt);
let rand_normal_fl=(avg:real=0.0, stdv:real=1.0)
{
    let x =rand()+rand()+rand()+rand()+rand()+rand()+rand()+rand()+rand()+rand()+rand()+rand();
    (x - 6)*stdv + avg
};
union
(range s from 0 to n step 1
| project t=startofday(now())-s*dt
| extend x=rand_normal_fl(10, 5)
| extend y=iff(hourofday(t) == 0, 2*(10-x)+7+rand_normal_fl(0, 3), 2*x+7+rand_normal_fl(0, 3))  //  anomalies every midnight
| extend name='TS1'),
(range s from 0 to n step 1
| project t=startofday(now())-s*dt
| extend x=rand_normal_fl(15, 3)
| extend y=iff(hourofday(t) == 8, (15-x)+10+rand_normal_fl(0, 2), x-7+rand_normal_fl(0, 1)) //  anomalies every 8am
| extend name='TS2'),
(range s from 0 to n step 1
| project t=startofday(now())-s*dt
| extend x=rand_normal_fl(8, 6)
| extend y=iff(hourofday(t) == 16, x+5+rand_normal_fl(0, 4), (12-x)+rand_normal_fl(0, 4)) //  anomalies every 4pm
| extend name='TS3')
| summarize t=make_list(t), x=make_list(x), y=make_list(y) by name
```

![Scatter chart of the sample data set.](images/series-mv-ee-anomalies-fl/mv-ee-anomalies-sample-data.png)