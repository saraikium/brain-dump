---
created: 2024-04-10T18:27
updated: 2024-04-10T19:09
---


## Mean

Average of the values. Sum the values and divide by the total number
$$
 \mu =  {\sum_{i=1}^{N} x_i  \over N }
$$

## Median
Median is the middle value if the total number of values is odd. If the number of values if even it's the average of the middle two values.

## Mode
Most frequent values in the dataset. If all values occur once then there's no mode. There can be more than one modes in a dataset.

## Python implementation
```python
from typing import Sequence
from collections import Counter

data = [1, 2, 55, 11, 3, 5, 8, 11, 23, 32]


def mean(data: Sequence[int | float]) -> int | float:
    return sum(data) / len(data)


def median(data: Sequence[int | float]) -> float:
    data = sorted(data)
    n = len(data)
    middle = n // 2

    if n % 2 == 0:
        return sum(data[middle : middle + 2]) / 2

    return data[middle]


def mode(data: Sequence[int | float]) -> list[int | float]:
    counts = Counter(data)
    max_count = max(counts.values())
    print(counts)
    modes = [num for num, count in counts.items() if count == max_count]

    if len(modes) == len(data):
        return []
    return modes


```

## PostgreSQL implementation

Let's say we have a table of health data. With the columns `user_id`, `measure`, `measure_value`, We can calculate the mean, median and mode of weight measurements with the following query.

```sql
select
  AVG(measure_value) as avg_weight,
  
  PERCENTILE_CONT(0.5) WITHIN GROUP (
    ORDER BY
      measure_value
  ) as median_weight,
  
  MODE() WITHIN GROUP (
    ORDER BY
      measure_value
  ) as mode_weight
from
  health.user_logs
where
  measure = 'weight';
```