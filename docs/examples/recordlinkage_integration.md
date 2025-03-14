# Integration with recordlinkage package

In this example we aim to show how users can utilize blocking results achieved with BlockingPy and use them with the [recordlinkage](https://github.com/J535D165/recordlinkage) package. The [recordlinkage](https://github.com/J535D165/recordlinkage) allows for both blocking and one-to-one record linkage and deduplication. However, it is possible to transfer blocking results from BlockingPy and incorporate them in the full entity resolution pipeline.

This example will show deduplication of febrl1 dataset which comes buillt-in with [recordlinkage](https://github.com/J535D165/recordlinkage).

We aim to follow the [Data deduplication](https://recordlinkage.readthedocs.io/en/latest/guides/data_deduplication.html#Introduction) example available on the recordlinkage documentation website and substitute the blocking procedure with our own.

## Setup

Firstly, we need to install `BlockingPy` and `recordlinkage`:

```bash
pip install blockingpy recordlinkage
```

Import necessary components:

```python
import recordlinkage
from recordlinkage.datasets import load_febrl1
from blockingpy import Blocker
import pandas as pd
from itertools import combinations 
```

## Data preparation

`febrl1` dataset contains 1000 records of which 500 are original and 500 are duplicates. It containts fictitious personal information e.g. name, surname, adress.

```python
df = load_febrl1()
print(df.head())

#               given_name	 surnam     street_number   address_1         address_2	suburb	    postcode	state	date_of_birth	soc_sec_id
# rec_id										
# rec-223-org	NaN	         waller	    6	            tullaroop street  willaroo	st james    4011        wa	    19081209	    6988048
# rec-122-org	lachlan	         berry	    69	            giblin street     killarney	bittern	    4814        qld	    19990219	    7364009

```

Prepare data in a suitable format for blockingpy. For this we need to fill missing values and concat fields to the `txt` column:

```python
df = df.fillna('')
df['txt'] = df['given_name'] + df['surname'] + \
            df['street_number'] + df['address_1'] + \
            df['address_2'] + df['suburb'] + \
            df['postcode'] + df['state'] + \
            df['date_of_birth'] + df['soc_sec_id']
```

## Blocking

Now we can obtain blocks from `BlockingPy`:

```python
blocker = Blocker()
blocking_result = blocker.block(x=df['txt'])

print(res)
# ========================================================
# Blocking based on the faiss method.
# Number of blocks: 500
# Number of columns used for blocking: 1023
# Reduction ratio: 0.998999
# ========================================================
# Distribution of the size of the blocks:
# Block Size | Number of Blocks
#          2 | 500  
print(res.result.head())
#        x    y  block      dist
# 0    474    0      0  0.048376
# 1    330    1      1  0.038961
# 2    351    2      2  0.086690
# 3    290    3      3  0.024617
# 4    333    4      4  0.105662
```

## Integration

To integrate our results, we can add a `block` column to the original dataframe, which we can do by melting the blocking result and merging it with the original dataframe.

```python
result_df = blocking_result.result

mapping_df = (
    result_df
    .melt(id_vars=['block'], value_vars=['x', 'y']  value_name='record_id')
    .drop_duplicates(subset=['record_id'])
)

record_to_block = dict(zip(mapping_df['record_id'], mapping_df['block']))
    
new_data = original_data.copy()

new_data['block'] = [record_to_block.get(i) for i in range(len(original_data))]

print(new_data[['given_name', 'block']].head(5))
# 	        given_name block
# rec_id		
# rec-223-org		   0
# rec-122-org	lachlan	   1
# rec-373-org	deakin	   2
# rec-10-dup-0	kayla	   3
# rec-227-org	luke	   4

```

Now we can use the `Index` object from `recordlinkage` with the `block` column to integrate `BlockingPy` results with `recordlinkage`:

```python
indexer = recordlinkage.Index()
indexer.block('block')
pairs = indexer.index(new_data)
print(pairs)
# MultiIndex([('rec-344-dup-0',   'rec-344-org'),
#             (  'rec-251-org', 'rec-251-dup-0'),
#             ('rec-335-dup-0',   'rec-335-org'),
#             ( 'rec-23-dup-0',    'rec-23-org'),
#             (  'rec-382-org', 'rec-382-dup-0'),
#               ....
```

***NOTE*** : This is the example for deduplication. Keep in mind that for record linkage this step needs to be modified.

Finally, we can use the execute one-to-one record linkage with the `recordlinkage` package. We will use the same comparison rules as in the original example:

```python
dfA = load_febrl1() # load original dataset once again for clean data
compare_cl = recordlinkage.Compare()

compare_cl.exact("given_name", "given_name", label="given_name")
compare_cl.string(
    "surname", "surname", method="jarowinkler", threshold=0.85, label="surname"
)
compare_cl.exact("date_of_birth", "date_of_birth", label="date_of_birth")
compare_cl.exact("suburb", "suburb", label="suburb")
compare_cl.exact("state", "state", label="state")
compare_cl.string("address_1", "address_1", threshold=0.85, label="address_1")

features = compare_cl.compute(pairs, dfA)

matches = features[features.sum(axis=1) > 3]
print(len(matches))
# 458 
# vs. 317 when blocking traditionally on 'given_name'
```
Comparison rules were adopted from the [orignal example](https://recordlinkage.readthedocs.io/en/latest/guides/data_deduplication.html#Introduction). 