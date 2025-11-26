下面是 **pandas 分组聚合（GroupBy + Aggregation）** 的最全详解，涵盖从基础到高级的所有常用技巧，帮助你真正掌握这一核心操作。



# Pandas



### 1. 基本概念：Split-Apply-Combine



text

```
Split     → 按某个（或多个）键把 DataFrame 分成若干组
Apply     → 对每组分别执行聚合、转换、过滤等操作
Combine   → 把结果合并成新的 DataFrame/Series
```

### 2. 最基础用法

Python

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    '店铺': ['北京1', '北京1', '北京2', '上海1', '上海1', '上海2', '广州1'],
    '日期': ['2025-01-01', '2025-01-02', '2025-01-01', '2025-01-01', 
             '2025-01-03', '2025-01-02', '2025-01-01'],
    '品类': ['手机', '电脑', '手机', '配件', '手机', '电脑', '配件'],
    '销售额': [5000, 8000, 6000, 2000, 7000, 9000, 3000],
    '销量': [5, 3, 6, 20, 7, 4, 30]
})

# 按单个列分组求和
df.groupby('店铺')['销售额'].sum()
# 或者等价写法
df.groupby('店铺').销售额.sum()
```

### 3. 常用聚合函数一览

| 函数               | 说明                | 支持多列 |
| ------------------ | ------------------- | -------- |
| .sum()             | 求和                | 是       |
| .mean()            | 平均值              | 是       |
| .median()          | 中位数              | 是       |
| .min() / .max()    | 最小/最大值         | 是       |
| .std() / .var()    | 标准差/方差         | 是       |
| .count()           | 非 NA 个数          | 是       |
| .size()            | 每组行数（包含 NA） | 否       |
| .first() / .last() | 第一条/最后一条记录 | 是       |
| .nunique()         | 去重后的个数        | 是       |
| .sem()             | 标准误差            | 是       |

### 4. 同时对多个列做相同聚合

Python

```
# 方法1：最常用
df.groupby('店铺')[['销售额', '销量']].sum()

# 方法2：agg()（推荐，功能最强）
df.groupby('店铺').agg({'销售额': 'sum', '销量': 'sum'})

# 方法3：aggregate() 是 agg 的别名
df.groupby('店铺').aggregate({'销售额': 'sum', '销量': 'mean'})
```

### 5. 对不同列使用不同聚合函数（最常用场景）

Python

```
df.groupby('店铺').agg({
    '销售额': ['sum', 'mean', 'max'],
    '销量':   ['sum', 'count', 'nunique']
})
# 结果会自动产生多层列索引（MultiIndex）
```

### 6. 给聚合列自定义名字（推荐！）

Python

```
df.groupby('店铺').agg(
    总销售额=('销售额', 'sum'),
    平均销售额=('销售额', 'mean'),
    总销量=('销量', 'sum'),
    订单数=('销量', 'count')      # count 其实是订单数
).round(2)   # 保留两位小数
```

### 7. 多列分组（超级常用）

Python

```
# 按店铺+品类分组
df.groupby(['店铺', '品类']).agg({'销售额': 'sum'})

# 或者
df.groupby(['店铺', '品类'], as_index=False).agg({'销售额': 'sum'})
# as_index=False 让分组列变成普通列，而不是索引
```

### 8. namedagg 方式（pandas 1.0+ 推荐写法）

Python

```
df.groupby('店铺', as_index=False).agg(
    总销售额 = pd.NamedAgg(column='销售额', aggfunc='sum'),
    平均单价 = pd.NamedAgg(column='销售额', aggfunc=lambda x: x.sum()/df.loc[x.index, '销量'].sum())
)
```

### 9. 自定义聚合函数

Python

```
# 1. 普通函数
def range_func(x):
    return x.max() - x.min()

df.groupby('店铺').agg({
    '销售额': ['sum', range_func, lambda x: x.max()-x.min()]
})

# 2. 极差占比（最大值是最小值的几倍）
def ratio_max_min(x):
    return x.max() / x.min() if x.min() != 0 else np.nan

df.groupby('店铺').agg(极差倍数=('销售额', ratio_max_min))
```

### 10. transform：分组后广播回原数据（不改变形状）

Python

```
# 每行加上自己店铺的平均销售额
df['店铺平均销售额'] = df.groupby('店铺')['销售额'].transform('mean')

# 占比：每行销售额占店铺总销售额的百分比
df['店铺占比'] = df['销售额'] / df.groupby('店铺')['销售额'].transform('sum')
```

### 11. filter：过滤掉不满足条件的组

Python

```
# 只保留总销售额 > 10000 的店铺
df.groupby('店铺').filter(lambda x: x['销售额'].sum() > 10000)
```

### 12. apply：最灵活但最慢（能干任何事）

Python

```
# 返回每组的前两名销售额记录
df.groupby('店铺').apply(lambda g: g.nlargest(2, '销售额'))

# 返回自定义 DataFrame
def my_stat(g):
    return pd.DataFrame({
        '总销售额': [g['销售额'].sum()],
        '平均单价': [g['销售额'].sum() / g['销量'].sum()]
    })

df.groupby('店铺').apply(my_stat)
```

### 13. 实战中最常用的 5 种写法总结

Python

```
# 1. 简单汇总
df.groupby('店铺').销售额.sum()

# 2. 多列多函数 + 自定义列名（推荐）
df.groupby('店铺').agg(
    总销售额=('销售额', 'sum'),
    总销量=('销量', 'sum'),
    订单数=('销量', 'count'),
    平均客单价=('销售额', lambda x: x.sum()/x.count())
).round(0)

# 3. 多层分组 + 排序
(df.groupby(['店铺', '品类'], as_index=False)
   .agg(销售额=('销售额', 'sum'))
   .sort_values('销售额', ascending=False))

# 4. 添加占比列
temp = df.groupby('店铺').agg(店铺总销售额=('销售额','sum')).reset_index()
df = df.merge(temp, on='店铺')
df['店铺占比'] = df['销售额'] / df['店铺总销售额']

# 5. transform 一行搞定占比
df['店铺占比'] = df['销售额'] / df.groupby('店铺')['销售额'].transform('sum')
```
