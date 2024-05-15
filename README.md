# GG20 Donations made through the [IDriss Browser Extension](https://chromewebstore.google.com/detail/idriss/fghhpjoffbgecjikiipbkpdakfmkbmig)
Post GG20 report on donations made through the IDriss browser extension

#### Imports


```python
import pandas as pd
import requests
from web3 import Web3
from tabulate import tabulate
from eth_abi import decode_single, decode_abi
import plotly.graph_objects as go
import matplotlib.pyplot as plt
```

#### Constants and Functions


```python
ACROSS_REQUEST_URL = "https://api.across.to/deposits/details"


def to_eth(value):
    return value / 10**18

```

### Cross-Chain Donations


```python
def decode_data(hex_str):
    data_bytes = bytes.fromhex(hex_str[2:])
    return decode_abi(
        ['address', 'address', 'uint256', 'address', 'uint256', 'address'],
        data_bytes
    )
```


```python
def fetch_deposit_details(tx_hash, origin_chain_id):
    params = {
        'depositTxHash': tx_hash,
        'originChainId': origin_chain_id
    }
    
    response = requests.get(ACROSS_REQUEST_URL, params=params)
    
    result = {
        'status': None,
        'message': None,
        'fillTxhash': None,
        'destination_chain': None
    }
    
    if response.status_code == 200:
        data = response.json()
        result['status'] = data.get('status')
        result['message'] = data.get('message')
        result['destination_chain'] = data.get('destinationChainId')
        
        fill_txs = data.get('fillTxs', [])
        if fill_txs:
            result['fillTxhash'] = fill_txs[0].get('hash')
    
    return result
```

#### Importing .csv files
The attestation files are downloaded .csv files from EAS' indexer as described here: https://github.com/idriss-crypto/browser-extensions/blob/master/CONTRACTS.md

The wrapper .csv files are downloaded transactions from our wrapper contracts. We used the download functionality on the respective block explorer and filtered for the date of the GG20 round before importing the files here. The contract addresses (and links to the block explorer) can be found in the `Gitcoin GG20 Donations` section here: https://github.com/idriss-crypto/browser-extensions/tree/master


```python
arbitrum_attestations = 'attestations_arbitrum.csv'
optimism_attestations = 'attestations_optimism.csv'
arbitrum_wrapper = 'arbitrum_wrapper.csv'
optimism_wrapper = 'optimism_wrapper.csv'
ethereum_wrapper = 'ethereum_wrapper.csv'
base_wrapper = 'base_wrapper.csv'
linea_wrapper = 'linea_wrapper.csv'
zksync_wrapper = 'zksync_wrapper.csv'
```


```python
relevant_columns_attestattion = ['attester', 'data', 'recipient', 'txid', 'id']
df_attestations_arb = pd.read_csv(arbitrum_attestations, index_col=False)[relevant_columns_attestattion]
df_attestations_op = pd.read_csv(optimism_attestations, index_col=False)[relevant_columns_attestattion]

relevant_columns_wrapper = ['Txhash', 'From', 'To', 'Method']
df_wrapper_arb = pd.read_csv(arbitrum_wrapper, index_col=False)[relevant_columns_wrapper]
df_wrapper_op = pd.read_csv(optimism_wrapper, index_col=False)[relevant_columns_wrapper]
df_wrapper_eth = pd.read_csv(ethereum_wrapper, index_col=False)[relevant_columns_wrapper]
df_wrapper_base = pd.read_csv(base_wrapper, index_col=False)[relevant_columns_wrapper]
df_wrapper_linea = pd.read_csv(linea_wrapper, index_col=False)[relevant_columns_wrapper]
df_wrapper_zksync = pd.read_csv(zksync_wrapper, index_col=False)[relevant_columns_wrapper]
```


```python
method_hex = '0x6fde4731'
method_name = 'Call Deposit V3'
df_wrapper_arb['Method'] = df_wrapper_arb['Method'].replace(method_hex, method_name)
df_wrapper_op['Method'] = df_wrapper_op['Method'].replace(method_hex, method_name)
df_wrapper_eth['Method'] = df_wrapper_eth['Method'].replace(method_hex, method_name)
df_wrapper_base['Method'] = df_wrapper_base['Method'].replace(method_hex, method_name)
df_wrapper_linea['Method'] = df_wrapper_linea['Method'].replace(method_hex, method_name)
df_wrapper_zksync['Method'] = df_wrapper_zksync['Method'].replace(method_hex, method_name)
```


```python
df_wrapper_arb = df_wrapper_arb.loc[df_wrapper_arb['Method'] == method_name]
df_wrapper_op = df_wrapper_op.loc[df_wrapper_op['Method'] == method_name]
df_wrapper_eth = df_wrapper_eth.loc[df_wrapper_eth['Method'] == method_name]
df_wrapper_base = df_wrapper_base.loc[df_wrapper_base['Method'] == method_name]
df_wrapper_linea = df_wrapper_linea.loc[df_wrapper_linea['Method'] == method_name]
df_wrapper_zksync = df_wrapper_zksync.loc[df_wrapper_zksync['Method'] == method_name]
```


```python
df_attestations_arb['destination_chain'] = '42161'
df_attestations_op['destination_chain'] = '10'
df_wrapper_arb['origin_chain'] = '42161'
df_wrapper_op['origin_chain'] = '10'
df_wrapper_eth['origin_chain'] = '1'
df_wrapper_base['origin_chain'] = '8453'
df_wrapper_linea['origin_chain'] = '59144'
df_wrapper_zksync['origin_chain'] = '324'
df_attestations = pd.concat([df_attestations_arb, df_attestations_op], ignore_index=True)
df_wrapper = pd.concat([df_wrapper_arb, df_wrapper_op, df_wrapper_eth, df_wrapper_base, df_wrapper_linea, df_wrapper_zksync], ignore_index=True)

df_attestations = df_attestations.rename(columns={'recipient':'attestation_recipient', 'txid':'Txhash', 'id':'uid'})
```


```python
df_wrapper['status'] = None
df_wrapper['message'] = None
df_wrapper['fillTxhash'] = None
df_wrapper['destination_chain'] = None

# Iterate through each row and update the new columns based on API data
for index, row in df_wrapper.iterrows():
    deposit_tx_hash = row['Txhash']
    origin_chain_id = row['origin_chain']
    
    # Fetch details and extract relevant fields
    details = fetch_deposit_details(deposit_tx_hash, origin_chain_id)
    
    # Assign the extracted values to the appropriate columns
    df_wrapper.at[index, 'status'] = details['status']
    df_wrapper.at[index, 'message'] = details['message']
    df_wrapper.at[index, 'fillTxhash'] = details['fillTxhash']
    df_wrapper.at[index, 'destination_chain'] = str(details['destination_chain'])
```


```python
# Confirmed error in the Across API:
# Txhash from attestation/donation: 0x7798d6ceb4f3f18f377c15980a2e19ac37f1c75d6e0f7d5f4a0ce8908337d76a
# Has a true corresponding deposit from Optimism at 0x9d83208add1a5517dd53e6ba392c66e36ec876317a1b39861c7eb9980fbf420a
correction = {
    'status': 'filled',
    'fillTxhash': '0x7798d6ceb4f3f18f377c15980a2e19ac37f1c75d6e0f7d5f4a0ce8908337d76a',
    'destination_chain': 42161,
    'message': '0x0000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000022000000000000000000000000000000000000000000000000000000000000001c0000000000000000000000000000000000000000000000000000000000000001d0000000000000000000000004a3755eb99ae8b22aafb8f16f0c51cf68eb60b850000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000014000000000000000000000000088e5e09a58292ec59ff229130c1f83b37b61e07300000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000060000000000000000000000000eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee0000000000000000000000000000000000000000000000000001269e991cf5fc0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004117016dcf9195e94de823a66dea9fa5db24018e2ab9325203989c3170b0c5574119b79bcdd72f220e61a9f2f21e8d40eb7933fbacaa78b563c22788c780c5ee181b00000000000000000000000000000000000000000000000000000000000000'
}
index_to_update = df_wrapper.loc[
    (df_wrapper['Txhash'] == '0x9d83208add1a5517dd53e6ba392c66e36ec876317a1b39861c7eb9980fbf420a') & 
    (df_wrapper['origin_chain'] == '10')
].index
if not index_to_update.empty:
    df_wrapper.at[index_to_update[0], 'status'] = correction['status']
    df_wrapper.at[index_to_update[0], 'message'] = correction['message']
    df_wrapper.at[index_to_update[0], 'fillTxhash'] = correction['fillTxhash']
    df_wrapper.at[index_to_update[0], 'destination_chain'] = str(correction['destination_chain'])
```


```python
gg20_rounds = {
    '42161': [23, 24, 25, 26, 27, 28, 29, 31],
    '10': [9]
}
df_attestations[['donor', 'recipient_id', 'round_id', 'token_sent', 'amount', 'origin']] = df_attestations['data'].apply(
    lambda x: pd.Series(decode_data(x))
)
def is_gg20_round(row):
    chain = row['destination_chain']
    round_id = row['round_id']
    return chain in gg20_rounds and round_id in gg20_rounds[chain]
df_attestations['gg20_round'] = df_attestations.apply(is_gg20_round, axis=1)
df_attestations = df_attestations[df_attestations['gg20_round']]
df_attestations.drop(columns='gg20_round', inplace=True)
attestations_final_df = pd.merge(df_attestations, df_wrapper, left_on='Txhash', right_on='fillTxhash', how='left', indicator=True)
attestations_final_df.drop(columns=['_merge', 'Method', 'token_sent', 'status', 'destination_chain_y'], inplace=True)
attestations_final_df = attestations_final_df.rename(columns={'destination_chain_x':'destination_chain', 'Txhash_x':'Txhash_destination', 'Txhash_y':'Txhash_origin'})
```


```python
print(attestations_final_df.head())
```

                                         attester  \
    0  0xaA098E5c9B002F815d7c9756BCfce0fC18B3F362   
    1  0xaA098E5c9B002F815d7c9756BCfce0fC18B3F362   
    2  0xaA098E5c9B002F815d7c9756BCfce0fC18B3F362   
    3  0xaA098E5c9B002F815d7c9756BCfce0fC18B3F362   
    4  0xaA098E5c9B002F815d7c9756BCfce0fC18B3F362   
    
                                                    data  \
    0  0x000000000000000000000000eebf48360c333407076f...   
    1  0x000000000000000000000000cce9a28b570946123f39...   
    2  0x000000000000000000000000cce9a28b570946123f39...   
    3  0x00000000000000000000000096babe8595ed944b0149...   
    4  0x0000000000000000000000005abca791c22e7f99237f...   
    
                            attestation_recipient  \
    0  0xeebF48360C333407076F4a8191017Bd30a88af56   
    1  0xcCE9A28b570946123f392Cf1DbfA6D2D5e636a1f   
    2  0xcCE9A28b570946123f392Cf1DbfA6D2D5e636a1f   
    3  0x96BAbe8595ed944B01499200B7Cfd230f8db3558   
    4  0x5ABca791C22E7f99237fCC04639E094Ffa0cCce9   
    
                                      Txhash_destination  \
    0  0x26b6b7d89b7ddc81e0885a37fd52301b89f434dd3f62...   
    1  0xe5ec4c7546c99dda5c1f2bf78f4329211f0ab60c87be...   
    2  0x0fccff1a4ffb8ca040715ca14524054cefe784a3752b...   
    3  0x8fcb12a27a908857b98e0a8d5ccd5275b20ecd1cd8c7...   
    4  0xf74f39f16aacf92136d44af6afafa5186d51ccba51de...   
    
                                                     uid destination_chain  \
    0  0x9e6fbf0ef8af3034a20cbbe978e1be5bc473a11b86a1...             42161   
    1  0x00dc9f867631830a08670ebbacc42d818eb2b1c83312...             42161   
    2  0xe2f066dad314b32d7389874d3ac7db79c10741adf2d8...             42161   
    3  0xeb97e574b8bc16b1ef1d719f60c6392879c9c6e6c29c...             42161   
    4  0xff92688b5033980549721d99db7058ebb4be1f656c63...             42161   
    
                                            donor  \
    0  0xeebf48360c333407076f4a8191017bd30a88af56   
    1  0xcce9a28b570946123f392cf1dbfa6d2d5e636a1f   
    2  0xcce9a28b570946123f392cf1dbfa6d2d5e636a1f   
    3  0x96babe8595ed944b01499200b7cfd230f8db3558   
    4  0x5abca791c22e7f99237fcc04639e094ffa0ccce9   
    
                                     recipient_id  round_id           amount  \
    0  0x8c8c259579e596b7bdb2bbc3e17d1b8d71e7a56d        25  326139414383250   
    1  0xb71335d629d165ebd081fa3879dd8a54c58b5193        26  326139414383250   
    2  0xe9da9d6b646c693b0651c7926e4666c924c4d01a        27  326139414383250   
    3  0x8c8c259579e596b7bdb2bbc3e17d1b8d71e7a56d        25  469576596957970   
    4  0x8c8c259579e596b7bdb2bbc3e17d1b8d71e7a56d        25  312771529639145   
    
                                           origin  \
    0  0x41ee28ee05341e7fdddc8d433ba66054cd302ca1   
    1  0x41ee28ee05341e7fdddc8d433ba66054cd302ca1   
    2  0x07ae8551be970cb1cca11dd7a11f47ae82e70e67   
    3  0xcad97616f91872c02ba3553db315db4015cbe850   
    4  0x41ee28ee05341e7fdddc8d433ba66054cd302ca1   
    
                                           Txhash_origin  \
    0  0xb43aa30fb16e58a67d47020f610fb4543edb2d548c62...   
    1  0x31990101bcbdb24d2ec7059990a23711426166ecb9d2...   
    2  0x5036d0d333573752c7711398f479c026d71c28527864...   
    3  0xb2dfee6dec15b9a3e4a947b08d40acdcad0162f1dc75...   
    4  0x7be3e2bc0dc385ebdbf0348bcc1542679a659c3c7aae...   
    
                                             From  \
    0  0xeebF48360C333407076F4a8191017Bd30a88af56   
    1  0xcce9a28b570946123f392cf1dbfa6d2d5e636a1f   
    2  0xcce9a28b570946123f392cf1dbfa6d2d5e636a1f   
    3  0x96babe8595ed944b01499200b7cfd230f8db3558   
    4  0x5abca791c22e7f99237fcc04639e094ffa0ccce9   
    
                                               To origin_chain  \
    0  0x8F5fc20f5a3e69B7DCc5AC477dCC4484C64897dA          324   
    1  0x51c2ddc09b67ab9152acfb6a9a5e7a8db1485ae8         8453   
    2  0x51c2ddc09b67ab9152acfb6a9a5e7a8db1485ae8         8453   
    3  0xd82bdb8391109f8bad393ff2cda9e7cd56f8239c           10   
    4  0x51c2ddc09b67ab9152acfb6a9a5e7a8db1485ae8         8453   
    
                                                 message  \
    0  0x00000000000000000000000000000000000000000000...   
    1  0x00000000000000000000000000000000000000000000...   
    2  0x00000000000000000000000000000000000000000000...   
    3  0x00000000000000000000000000000000000000000000...   
    4  0x00000000000000000000000000000000000000000000...   
    
                                              fillTxhash  
    0  0x26b6b7d89b7ddc81e0885a37fd52301b89f434dd3f62...  
    1  0xe5ec4c7546c99dda5c1f2bf78f4329211f0ab60c87be...  
    2  0x0fccff1a4ffb8ca040715ca14524054cefe784a3752b...  
    3  0x8fcb12a27a908857b98e0a8d5ccd5275b20ecd1cd8c7...  
    4  0xf74f39f16aacf92136d44af6afafa5186d51ccba51de...  
    


```python
num_unique_donors = attestations_final_df['donor'].nunique()
num_unique_recipients = attestations_final_df['recipient_id'].nunique()
avg_amount = attestations_final_df['amount'].mean()
median_amount = attestations_final_df['amount'].median()
total_amount = attestations_final_df['amount'].sum()
top_donor = attestations_final_df['donor'].value_counts().idxmax()
top_recipient = attestations_final_df['recipient_id'].value_counts().idxmax()
top_donor_donations = attestations_final_df[attestations_final_df['donor'] == top_donor].shape[0]
top_recipient_donations = attestations_final_df[attestations_final_df['recipient_id'] == top_recipient].shape[0]
avg_donations_by_donor = attestations_final_df.groupby('donor')['amount'].mean()
median_donations_by_donor = attestations_final_df.groupby('donor')['amount'].median()
avg_donations_by_recipient = attestations_final_df.groupby('recipient_id')['amount'].mean()
median_donations_by_recipient = attestations_final_df.groupby('recipient_id')['amount'].median()
round_id_counts = attestations_final_df['round_id'].value_counts()
total_donations = attestations_final_df.shape[0]


statistics = {
    'Total Donations': total_donations,
    'Number of Unique Donors': num_unique_donors,
    'Number of Unique Recipients': num_unique_recipients,
    'Average Amount': to_eth(avg_amount),
    'Median Amount': to_eth(median_amount),
    'Total Amount': to_eth(total_amount),
    'Top Donor': top_donor,
    'Top Recipient': top_recipient,
    'Number of Donations by Top Donor': top_donor_donations,
    'Number of Donations Received by Top Recipient': top_recipient_donations,
    'Average Donations by Donor': to_eth(avg_donations_by_donor.mean()),
    'Median Donations by Donor': to_eth(median_donations_by_donor.median()),
    'Average Donations by Recipient': to_eth(avg_donations_by_recipient.mean()),
    'Median Donations by Recipient': to_eth(median_donations_by_recipient.median())
}


statistics_df = pd.DataFrame.from_dict(statistics, orient='index', columns=['Value'])

print("\nStatistics for Cross-Chain Donations:")
print(statistics_df)

```

    
    Statistics for Cross-Chain Donations:
                                                                                        Value
    Total Donations                                                                       230
    Number of Unique Donors                                                                55
    Number of Unique Recipients                                                           108
    Average Amount                                                                0.000451459
    Median Amount                                                                 0.000323257
    Total Amount                                                                     0.103835
    Top Donor                                      0x5abca791c22e7f99237fcc04639e094ffa0ccce9
    Top Recipient                                  0x8c8c259579e596b7bdb2bbc3e17d1b8d71e7a56d
    Number of Donations by Top Donor                                                       45
    Number of Donations Received by Top Recipient                                          45
    Average Donations by Donor                                                    0.000526271
    Median Donations by Donor                                                     0.000327877
    Average Donations by Recipient                                                0.000391558
    Median Donations by Recipient                                                 0.000323147
    


```python
transactions_by_origin = attestations_final_df.groupby('origin_chain').agg(
    transaction_count=('amount', 'count'),
    total_amount=('amount', 'sum'),
    average_amount=('amount', 'mean'),
    median_amount=('amount', 'median')
)

transactions_by_origin.reset_index(inplace=True)

transactions_by_origin[['total_amount', 'average_amount', 'median_amount']] = \
    transactions_by_origin[['total_amount', 'average_amount', 'median_amount']].applymap(to_eth).round(6)

print("\nTransactions by Origin Chain:")
print(tabulate(transactions_by_origin, headers='keys', tablefmt='pretty', showindex=False, colalign=('right', 'center', 'right', 'right', 'right')))

```

    
    Transactions by Origin Chain:
    +--------------+-------------------+--------------+----------------+---------------+
    | origin_chain | transaction_count | total_amount | average_amount | median_amount |
    +--------------+-------------------+--------------+----------------+---------------+
    |            1 |         3         |     0.002321 |       0.000774 |      0.000322 |
    |           10 |        37         |     0.015958 |       0.000431 |      0.000323 |
    |          324 |        86         |     0.040651 |       0.000473 |      0.000327 |
    |        42161 |         1         |     0.000323 |       0.000323 |      0.000323 |
    |        59144 |        21         |     0.010353 |       0.000493 |       0.00032 |
    |         8453 |        82         |     0.034229 |       0.000417 |      0.000319 |
    +--------------+-------------------+--------------+----------------+---------------+
    


```python
round_id_counts = attestations_final_df['round_id'].value_counts().reset_index()
round_id_counts.columns = ['round_id', 'count']  # Rename columns for clarity

round_id_counts.sort_values(by='round_id', inplace=True)
total_donations = round_id_counts['count'].sum()
round_id_counts['percentage'] = round_id_counts['count'] / total_donations * 100
round_id_counts['percentage'] = round_id_counts['percentage'].apply(lambda x: f"{x:.2f}%")

print("\nRound ID Counts Table for Cross-Chain Donations:")
print(tabulate(round_id_counts, headers='keys', tablefmt='pretty', showindex=False, colalign=('right', 'right', 'right')))
```

    
    Round ID Counts Table for Cross-Chain Donations:
    +----------+-------+------------+
    | round_id | count | percentage |
    +----------+-------+------------+
    |        9 |     2 |      0.87% |
    |       23 |    12 |      5.22% |
    |       24 |    17 |      7.39% |
    |       25 |   115 |     50.00% |
    |       26 |    41 |     17.83% |
    |       27 |     8 |      3.48% |
    |       28 |     9 |      3.91% |
    |       29 |    24 |     10.43% |
    |       31 |     2 |      0.87% |
    +----------+-------+------------+
    


```python
origin_chain_counts = attestations_final_df['origin_chain'].value_counts()
plt.figure(figsize=(8, 8))
plt.pie(origin_chain_counts, labels=origin_chain_counts.index, autopct='%1.1f%%', startangle=140)
plt.title('Distribution of Origin Chains')
plt.axis('equal')
plt.show()

destination_chain_counts = attestations_final_df['destination_chain'].value_counts()
plt.figure(figsize=(8, 8))
plt.pie(destination_chain_counts, labels=destination_chain_counts.index, autopct='%1.1f%%', startangle=140)
plt.title('Distribution of Destination Chains')
plt.axis('equal')
plt.show()

round_id_counts = attestations_final_df['round_id'].value_counts()

plt.figure(figsize=(8, 8))
plt.pie(
    round_id_counts, 
    labels=round_id_counts.index, 
    autopct='%1.1f%%',  
    startangle=140      
)
plt.title('Distribution of Round ID Counts')
plt.axis('equal')
plt.show()

top_donors = attestations_final_df['donor'].value_counts().head(10)
plt.figure(figsize=(10, 6))
plt.bar(top_donors.index, top_donors.values)
plt.xlabel('Top Donors')
plt.ylabel('Count')
plt.title('Top 10 Donors')
plt.xticks(rotation=45, ha='right')
plt.grid(axis='y')
plt.show()

top_recipients = attestations_final_df['recipient_id'].value_counts().head(10)
plt.figure(figsize=(10, 6))
plt.bar(top_recipients.index, top_recipients.values)
plt.xlabel('Top Recipients')
plt.ylabel('Count')
plt.title('Top 10 Recipients')
plt.xticks(rotation=45, ha='right')
plt.grid(axis='y')
plt.show()



amounts = attestations_final_df['amount'] / 10**18  
plt.figure(figsize=(10, 6))
plt.hist(amounts, bins=60, edgecolor='black')
plt.xlabel('Amount (ETH)')
plt.ylabel('Frequency')
plt.title('Distribution of Amounts')
plt.grid(axis='y')
```


![png](/notebookImg/output_21_0.png)



![png](/notebookImg/output_21_1.png)



![png](/notebookImg/output_21_2.png)



![png](/notebookImg/output_21_3.png)



![png](/notebookImg/output_21_4.png)



![png](/notebookImg/output_21_5.png)


### Same Chain Donations 


```python
provider_arb = Web3.HTTPProvider("https://arb1.arbitrum.io/rpc")
provider_op = Web3.HTTPProvider("https://mainnet.optimism.io")
w3_arb = Web3(provider_arb)
w3_op = Web3(provider_op)
```

The `{chain}_allo.csv` files are downloaded transactions from the Arbitrum and Optimism block explorer, filtered by the dates of GG20. We are not aware of another direct contract integration and assume that all direct `allocate()` calls were made through our extension. The traditional checkout uses a multi-checkout contract, which makes it so the allocations show up as internal transactions (and therefore not in the following data frames).


```python
arbitrum_allo = 'arbitrum_allo.csv'
optimism_allo = 'optimism_allo.csv'

relevant_columns_attestattion = ['Txhash', 'From', 'Method', 'Value_IN(ETH)', 'Status']
df_allo_arb = pd.read_csv(arbitrum_allo, index_col=False)[relevant_columns_attestattion]
df_allo_op = pd.read_csv(optimism_allo, index_col=False)[relevant_columns_attestattion]
df_allo_arb['origin_chain'] = 42161
df_allo_op['origin_chain'] = 10
df_allo_arb['destination_chain'] = 42161
df_allo_op['destination_chain'] = 10
df_allo = pd.concat([df_allo_arb, df_allo_op], ignore_index=True)
df_allo = df_allo.loc[df_allo['Method'] == 'Allocate']
df_allo = df_allo.loc[df_allo['Status'] != 'Error(0)']
df_allo.drop(columns='Status', inplace=True)
```


```python
allocate_event_signature = '0xdc9d40760308557d1377c2fe7c984ace9eb02d23b60a5f6f26be62c52431bc38'
allocated_event_abi = [
    {"indexed": True, "name": "recipientId", "type": "address"},
    {"indexed": False, "name": "amount", "type": "uint256"},
    {"indexed": False, "name": "token", "type": "address"},
    {"indexed": False, "name": "sender", "type": "address"},
    {"indexed": False, "name": "origin", "type": "address"}
]
gg20_rounds = {
    '42161': [23, 24, 25, 26, 27, 28, 29, 31],
    '10': [9]
}
def decode_tx_data_and_event(row):
    tx_hash = row['Txhash']
    origin_chain = str(row['origin_chain'])
    if origin_chain == '42161':
        w3 = w3_arb
    elif origin_chain == '10':
        w3 = w3_op
    else:
        return None  

    try:
        tx = w3.eth.get_transaction(tx_hash)
        tx_receipt = w3.eth.get_transaction_receipt(tx_hash)
    except Exception as e:
        print(f"Error fetching receipt for {tx_hash}: {e}")
        return None 
    
    try:
        input_data = tx.input[10:]
        decoded_input = decode_abi(['uint256', 'bytes'], bytes.fromhex(input_data))
        round_id = decoded_input[0]
    except Exception as e:
        print(f"Error decoding input data for {tx_hash}: {e}")
        round_id = None
        
    allocated_event_log = None
    for log in tx_receipt.logs:
        if log['topics'][0].hex() == allocate_event_signature:
            allocated_event_log = log
            break
            
    if allocated_event_log:
        recipient_id_raw = allocated_event_log['topics'][1].hex()
        recipient_id = w3.toChecksumAddress('0x' + recipient_id_raw[-40:])

        non_indexed_data = decode_abi(
            ['uint256', 'address', 'address', 'address'],
            bytes.fromhex(allocated_event_log['data'][2:])
        )
        amount, token, sender, origin = non_indexed_data
        is_gg20_round = round_id in gg20_rounds.get(origin_chain, [])

        return {
            'round_id': round_id,
            'recipient_id': recipient_id,
            'amount': amount,
            'token': token,
            'donor': sender,
            'origin': origin,
            'is_gg20_round': is_gg20_round,
        }
    else:
        return {
            'round_id': round_id,
            'recipient_id': None,
            'amount': None,
            'token': None,
            'donor': None,
            'origin': None,
            'is_gg20_round': False,
        }

```


```python
new_columns = df_allo.apply(lambda row: pd.Series(decode_tx_data_and_event(row)), axis=1)

df_allo = pd.concat([df_allo, new_columns], axis=1)
```


```python
df_allo = df_allo[df_allo['is_gg20_round']]
df_allo.drop(columns='is_gg20_round', inplace=True)
```


```python
columns_to_check = ['round_id', 'recipient_id', 'amount', 'token', 'donor', 'origin']

missing_values_df = df_allo[df_allo[columns_to_check].isnull().any(axis=1)]

print("\nTransactions with None or missing values:")
print(missing_values_df)
```

    
    Transactions with None or missing values:
    Empty DataFrame
    Columns: [Txhash, From, Method, Value_IN(ETH), origin_chain, destination_chain, round_id, recipient_id, amount, token, donor, origin]
    Index: []
    


```python
num_unique_donors_same_chain = df_allo['donor'].nunique()
num_unique_recipients_same_chain = df_allo['recipient_id'].nunique()
avg_amount_same_chain = df_allo['amount'].mean()
median_amount_same_chain = df_allo['amount'].median()
total_amount_same_chain = df_allo['amount'].sum()
top_donor_same_chain = df_allo['donor'].value_counts().idxmax()
top_recipient_same_chain = df_allo['recipient_id'].value_counts().idxmax()
top_donor_donations_same_chain = df_allo[df_allo['donor'] == top_donor].shape[0]
top_recipient_donations_same_chain = df_allo[df_allo['recipient_id'] == top_recipient_same_chain].shape[0]
avg_donations_by_donor_same_chain = df_allo.groupby('donor')['amount'].mean()
median_donations_by_donor_same_chain = df_allo.groupby('donor')['amount'].median()
avg_donations_by_recipient_same_chain = df_allo.groupby('recipient_id')['amount'].mean()
median_donations_by_recipient_same_chain = df_allo.groupby('recipient_id')['amount'].median()
round_id_counts_same_chain = df_allo['round_id'].value_counts()
total_donations_same_chain = df_allo.shape[0]


statistics_same_chain = {
    'Total Donations': total_donations_same_chain,
    'Number of Unique Donors': num_unique_donors_same_chain,
    'Number of Unique Recipients': num_unique_recipients_same_chain,
    'Average Amount': to_eth(avg_amount_same_chain),
    'Median Amount': to_eth(median_amount_same_chain),
    'Total Amount': to_eth(total_amount_same_chain),
    'Top Donor': top_donor_same_chain,
    'Top Recipient': top_recipient_same_chain,
    'Number of Donations by Top Donor': top_donor_donations_same_chain,
    'Number of Donations Received by Top Recipient': top_recipient_donations_same_chain,
    'Average Donations by Donor': to_eth(avg_donations_by_donor_same_chain.mean()),
    'Median Donations by Donor': to_eth(median_donations_by_donor_same_chain.median()),
    'Average Donations by Recipient': to_eth(avg_donations_by_recipient_same_chain.mean()),
    'Median Donations by Recipient': to_eth(median_donations_by_recipient_same_chain.median())
}


statistics_df_same_chain = pd.DataFrame.from_dict(statistics_same_chain, orient='index', columns=['Value'])

print("\nStatistics for Same Chain Donations:")
print(statistics_df_same_chain)

```

    
    Statistics for Same Chain Donations:
                                                                                        Value
    Total Donations                                                                       185
    Number of Unique Donors                                                                48
    Number of Unique Recipients                                                            62
    Average Amount                                                                0.000394502
    Median Amount                                                                 0.000327142
    Total Amount                                                                    0.0729828
    Top Donor                                      0x4a3755eb99ae8b22aafb8f16f0c51cf68eb60b85
    Top Recipient                                  0x8c8c259579e596b7BdB2BbC3E17D1b8d71E7a56d
    Number of Donations by Top Donor                                                        5
    Number of Donations Received by Top Recipient                                          33
    Average Donations by Donor                                                    0.000537891
    Median Donations by Donor                                                     0.000326992
    Average Donations by Recipient                                                0.000383868
    Median Donations by Recipient                                                 0.000327091
    


```python
round_id_counts_same_chain = df_allo['round_id'].value_counts().reset_index()
round_id_counts_same_chain.columns = ['round_id', 'count']  # Rename columns for clarity

round_id_counts_same_chain.sort_values(by='round_id', inplace=True)
total_donations = round_id_counts_same_chain['count'].sum()
round_id_counts_same_chain['percentage'] = round_id_counts_same_chain['count'] / total_donations * 100
round_id_counts_same_chain['percentage'] = round_id_counts_same_chain['percentage'].apply(lambda x: f"{x:.2f}%")

print("\nRound ID Counts Table:")
print(tabulate(round_id_counts_same_chain, headers='keys', tablefmt='pretty', showindex=False, colalign=('right', 'right', 'right')))
```

    
    Round ID Counts Table:
    +----------+-------+------------+
    | round_id | count | percentage |
    +----------+-------+------------+
    |        9 |     3 |      1.62% |
    |       23 |     2 |      1.08% |
    |       24 |     5 |      2.70% |
    |       25 |    98 |     52.97% |
    |       26 |    37 |     20.00% |
    |       27 |    25 |     13.51% |
    |       28 |     6 |      3.24% |
    |       29 |     9 |      4.86% |
    +----------+-------+------------+
    


```python
origin_chain_counts_same_chain = df_allo['origin_chain'].value_counts()
plt.figure(figsize=(8, 8))
plt.pie(origin_chain_counts_same_chain, labels=origin_chain_counts_same_chain.index, autopct='%1.1f%%', startangle=140)
plt.title('Distribution of Origin Chains')
plt.axis('equal')
plt.show()

round_id_counts_same_chain = df_allo['round_id'].value_counts()

plt.figure(figsize=(8, 8))
plt.pie(
    round_id_counts_same_chain, 
    labels=round_id_counts_same_chain.index, 
    autopct='%1.1f%%',  
    startangle=140     
)
plt.title('Distribution of Round ID Counts')
plt.axis('equal')  
plt.show()

top_donors_same_chain = df_allo['donor'].value_counts().head(10)
plt.figure(figsize=(10, 6))
plt.bar(top_donors_same_chain.index, top_donors_same_chain.values)
plt.xlabel('Top Donors')
plt.ylabel('Count')
plt.title('Top 10 Donors')
plt.xticks(rotation=45, ha='right')
plt.grid(axis='y')
plt.show()

top_recipients_same_chain = df_allo['recipient_id'].value_counts().head(10)
plt.figure(figsize=(10, 6))
plt.bar(top_recipients_same_chain.index, top_recipients_same_chain.values)
plt.xlabel('Top Recipients')
plt.ylabel('Count')
plt.title('Top 10 Recipients')
plt.xticks(rotation=45, ha='right')
plt.grid(axis='y')
plt.show()



amounts_same_chain = df_allo['amount'] / 10**18  
plt.figure(figsize=(10, 6))
plt.hist(amounts_same_chain, bins=60, edgecolor='black')
plt.xlabel('Amount (ETH)')
plt.ylabel('Frequency')
plt.title('Distribution of Amounts')
plt.grid(axis='y')
```


![png](/notebookImg/output_32_0.png)



![png](/notebookImg/output_32_1.png)



![png](/notebookImg/output_32_2.png)



![png](/notebookImg/output_32_3.png)



![png](/notebookImg/output_32_4.png)


## Combined


```python
df_combined = pd.concat([df_allo, attestations_final_df], axis=0, ignore_index=True)
print(df_combined.head())
```

                                                  Txhash  \
    0  0xf7cb5d4bf0cc9bdcdfc2119ea44d5c22c27a788ef787...   
    1  0xfe60d4af445afe6a05c56f760cb9d8acdc852cb439c0...   
    2  0x78f525a45ac22fe8df1c69b94702827a4ba4857d3f32...   
    3  0xadaaf2379b4d5d2021e53caeea2ca2c2ac2bb480992c...   
    4  0x0faa98cb9f8c63a68bac919b0811e922db84eb639bcb...   
    
                                             From    Method  Value_IN(ETH)  \
    0  0x96babe8595ed944b01499200b7cfd230f8db3558  Allocate       0.000094   
    1  0xf106e0311b00e2b5e2a27dfef45a44add2df4b5b  Allocate       0.000308   
    2  0x9a03f172d94ec286a0d72ee8f389ddfd105f0a36  Allocate       0.000309   
    3  0x87b53f4d4cb88a808504f2d4c13b93bfaf7510b6  Allocate       0.001544   
    4  0x9a03f172d94ec286a0d72ee8f389ddfd105f0a36  Allocate       0.000031   
    
      origin_chain destination_chain  round_id  \
    0        42161             42161        25   
    1        42161             42161        25   
    2        42161             42161        25   
    3        42161             42161        25   
    4        42161             42161        25   
    
                                     recipient_id        amount  \
    0  0x8c8c259579e596b7BdB2BbC3E17D1b8d71E7a56d  9.394450e+13   
    1  0x8c8c259579e596b7BdB2BbC3E17D1b8d71E7a56d  3.078306e+14   
    2  0x8c8c259579e596b7BdB2BbC3E17D1b8d71E7a56d  3.085045e+14   
    3  0x8c8c259579e596b7BdB2BbC3E17D1b8d71E7a56d  1.544402e+15   
    4  0x8c8c259579e596b7BdB2BbC3E17D1b8d71E7a56d  3.094624e+13   
    
                                            token  ...  \
    0  0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee  ...   
    1  0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee  ...   
    2  0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee  ...   
    3  0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee  ...   
    4  0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee  ...   
    
                                           origin attester data  \
    0  0x96babe8595ed944b01499200b7cfd230f8db3558      NaN  NaN   
    1  0xf106e0311b00e2b5e2a27dfef45a44add2df4b5b      NaN  NaN   
    2  0x9a03f172d94ec286a0d72ee8f389ddfd105f0a36      NaN  NaN   
    3  0x87b53f4d4cb88a808504f2d4c13b93bfaf7510b6      NaN  NaN   
    4  0x9a03f172d94ec286a0d72ee8f389ddfd105f0a36      NaN  NaN   
    
      attestation_recipient Txhash_destination  uid Txhash_origin   To message  \
    0                   NaN                NaN  NaN           NaN  NaN     NaN   
    1                   NaN                NaN  NaN           NaN  NaN     NaN   
    2                   NaN                NaN  NaN           NaN  NaN     NaN   
    3                   NaN                NaN  NaN           NaN  NaN     NaN   
    4                   NaN                NaN  NaN           NaN  NaN     NaN   
    
      fillTxhash  
    0        NaN  
    1        NaN  
    2        NaN  
    3        NaN  
    4        NaN  
    
    [5 rows x 21 columns]
    


```python
num_unique_donors_combined = df_combined['donor'].nunique()
num_unique_recipients_combined = df_combined['recipient_id'].nunique()
avg_amount_combined = df_combined['amount'].mean()
median_amount_combined = df_combined['amount'].median()
total_amount_combined = df_combined['amount'].sum()
top_donor_combined = df_combined['donor'].value_counts().idxmax()
top_recipient_combined = df_combined['recipient_id'].value_counts().idxmax()
top_donor_donations_combined = df_combined[df_combined['donor'] == top_donor_combined].shape[0]
top_recipient_donations_combined = df_combined[df_combined['recipient_id'] == top_recipient_combined].shape[0]
avg_donations_by_donor_combined = df_combined.groupby('donor')['amount'].mean()
median_donations_by_donor_combined = df_combined.groupby('donor')['amount'].median()
avg_donations_by_recipient_combined = df_combined.groupby('recipient_id')['amount'].mean()
median_donations_by_recipient_combined = df_combined.groupby('recipient_id')['amount'].median()
round_id_counts_combined = df_combined['round_id'].value_counts()
total_donations_combined = df_combined.shape[0]


statistics_combined = {
    'Total Donations': total_donations_combined,
    'Number of Unique Donors': num_unique_donors_combined,
    'Number of Unique Recipients': num_unique_recipients_combined,
    'Average Amount': to_eth(avg_amount_combined),
    'Median Amount': to_eth(median_amount_combined),
    'Total Amount': to_eth(total_amount_combined),
    'Top Donor': top_donor_combined,
    'Top Recipient': top_recipient_combined,
    'Number of Donations by Top Donor': top_donor_donations_combined,
    'Number of Donations Received by Top Recipient': top_recipient_donations_combined,
    'Average Donations by Donor': to_eth(avg_donations_by_donor_combined.mean()),
    'Median Donations by Donor': to_eth(median_donations_by_donor_combined.median()),
    'Average Donations by Recipient': to_eth(avg_donations_by_recipient_combined.mean()),
    'Median Donations by Recipient': to_eth(median_donations_by_recipient_combined.median())
}


statistics_df_combined = pd.DataFrame.from_dict(statistics_combined, orient='index', columns=['Value'])

print("\nCombined Statistics:")
print(statistics_df_combined)

```

    
    Combined Statistics:
                                                                                        Value
    Total Donations                                                                       415
    Number of Unique Donors                                                                90
    Number of Unique Recipients                                                           170
    Average Amount                                                                0.000426068
    Median Amount                                                                 0.000326835
    Total Amount                                                                     0.176818
    Top Donor                                      0x4a3755eb99ae8b22aafb8f16f0c51cf68eb60b85
    Top Recipient                                  0x8c8c259579e596b7bdb2bbc3e17d1b8d71e7a56d
    Number of Donations by Top Donor                                                       59
    Number of Donations Received by Top Recipient                                          45
    Average Donations by Donor                                                    0.000546921
    Median Donations by Donor                                                     0.000327078
    Average Donations by Recipient                                                0.000388754
    Median Donations by Recipient                                                 0.000324168
    


```python
transactions_by_origin_combined = df_combined.groupby('origin_chain').agg(
    transaction_count=('amount', 'count'),
    total_amount=('amount', 'sum'),
    average_amount=('amount', 'mean'),
    median_amount=('amount', 'median')
)

# Convert amounts to Ethereum
transactions_by_origin_combined[['total_amount', 'average_amount', 'median_amount']] = \
    transactions_by_origin_combined[['total_amount', 'average_amount', 'median_amount']].applymap(to_eth).round(6)

# Calculate totals for percentage calculations
total_transactions = transactions_by_origin_combined['transaction_count'].sum()
total_amount_sent = transactions_by_origin_combined['total_amount'].sum()

# Add percentage columns
transactions_by_origin_combined['transactions_pct'] = (transactions_by_origin_combined['transaction_count'] / total_transactions * 100).round(2).astype(str) + '%'
transactions_by_origin_combined['amount_pct'] = (transactions_by_origin_combined['total_amount'] / total_amount_sent * 100).round(2).astype(str) + '%'

# Reset index to make 'origin_chain' a column for display
transactions_by_origin_combined.reset_index(inplace=True)

# Print the table
print("\nCombined Transactions by Origin Chain:")
print(tabulate(transactions_by_origin_combined, headers='keys', tablefmt='pretty', showindex=False, colalign=('right', 'right', 'right', 'right', 'right', 'right', 'right')))
```

    
    Combined Transactions by Origin Chain:
    +--------------+-------------------+--------------+----------------+---------------+------------------+------------+
    | origin_chain | transaction_count | total_amount | average_amount | median_amount | transactions_pct | amount_pct |
    +--------------+-------------------+--------------+----------------+---------------+------------------+------------+
    |            1 |                 3 |     0.002321 |       0.000774 |      0.000322 |            0.72% |      1.31% |
    |           10 |                40 |     0.018229 |       0.000456 |      0.000323 |            9.64% |     10.31% |
    |          324 |                86 |     0.040651 |       0.000473 |      0.000327 |           20.72% |     22.99% |
    |        42161 |               183 |     0.071035 |       0.000388 |      0.000327 |            44.1% |     40.17% |
    |        59144 |                21 |     0.010353 |       0.000493 |       0.00032 |            5.06% |      5.86% |
    |         8453 |                82 |     0.034229 |       0.000417 |      0.000319 |           19.76% |     19.36% |
    +--------------+-------------------+--------------+----------------+---------------+------------------+------------+
    



```python
round_id_counts_combined = df_combined['round_id'].value_counts().reset_index()
round_id_counts_combined.columns = ['round_id', 'count']  # Rename columns for clarity
total_donations = round_id_counts_combined['count'].sum()
round_id_counts_combined['percentage'] = round_id_counts_combined['count'] / total_donations * 100
round_id_counts_combined['percentage'] = round_id_counts_combined['percentage'].apply(lambda x: f"{x:.2f}%")
round_id_counts_combined.sort_values(by='round_id', inplace=True)

print("\nCombined Round ID Counts Table:")
print(tabulate(round_id_counts_combined, headers='keys', tablefmt='pretty', showindex=False, colalign=('right', 'right', 'right')))
```

    
    Combined Round ID Counts Table:
    +----------+-------+------------+
    | round_id | count | percentage |
    +----------+-------+------------+
    |        9 |     5 |      1.20% |
    |       23 |    14 |      3.37% |
    |       24 |    22 |      5.30% |
    |       25 |   213 |     51.33% |
    |       26 |    78 |     18.80% |
    |       27 |    33 |      7.95% |
    |       28 |    15 |      3.61% |
    |       29 |    33 |      7.95% |
    |       31 |     2 |      0.48% |
    +----------+-------+------------+
    


```python
origin_chain_counts_combined = df_combined['origin_chain'].value_counts()
plt.figure(figsize=(8, 8))
plt.pie(origin_chain_counts_combined, labels=origin_chain_counts_combined.index, autopct='%1.1f%%', startangle=140)
plt.title('Distribution of Origin Chains')
plt.axis('equal')
plt.show()

destination_chain_counts_combined = df_combined['destination_chain'].value_counts()
plt.figure(figsize=(8, 8))
plt.pie(destination_chain_counts_combined, labels=destination_chain_counts_combined.index, autopct='%1.1f%%', startangle=140)
plt.title('Distribution of Destination Chains')
plt.axis('equal')
plt.show()

round_id_counts_combined = df_combined['round_id'].value_counts()

plt.figure(figsize=(8, 8))
plt.pie(
    round_id_counts_combined, 
    labels=round_id_counts_combined.index, 
    autopct='%1.1f%%', 
    startangle=140     
)
plt.title('Distribution of Round ID Counts')
plt.axis('equal')  
plt.show()

top_donors_combined = df_combined['donor'].value_counts().head(10)
plt.figure(figsize=(10, 6))
plt.bar(top_donors_combined.index, top_donors_combined.values)
plt.xlabel('Top Donors')
plt.ylabel('Count')
plt.title('Top 10 Donors')
plt.xticks(rotation=45, ha='right')
plt.grid(axis='y')
plt.show()

top_recipients_combined = df_combined['recipient_id'].value_counts().head(10)
plt.figure(figsize=(10, 6))
plt.bar(top_recipients_combined.index, top_recipients_combined.values)
plt.xlabel('Top Recipients')
plt.ylabel('Count')
plt.title('Top 10 Recipients')
plt.xticks(rotation=45, ha='right')
plt.grid(axis='y')
plt.show()


amounts_combined = df_combined['amount'] / 10**18  # Adjust this if amounts are already in ETH
plt.figure(figsize=(10, 6))
plt.hist(amounts_combined, bins=60, edgecolor='black')  # Adjust the number of bins as needed
plt.xlabel('Amount (ETH)')
plt.ylabel('Frequency')
plt.title('Distribution of Amounts')
plt.grid(axis='y')
```


![png](/notebookImg/output_38_0.png)



![png](/notebookImg/output_38_1.png)



![png](/notebookImg/output_38_2.png)



![png](/notebookImg/output_38_3.png)



![png](/notebookImg/output_38_4.png)



![png](/notebookImg/output_38_5.png)



```python
all_chains_combined = pd.concat([df_combined['origin_chain'], df_combined['destination_chain']]).unique()
chain_to_idx_combined = {chain: idx for idx, chain in enumerate(all_chains_combined)}

df_combined['origin_chain'] = df_combined['origin_chain'].astype(str)
df_combined['destination_chain'] = df_combined['destination_chain'].astype(str)

chain_names = {
    '42161': 'Arbitrum',
    '324': 'zkSync Era',
    '8453': 'Base',
    '59144': 'Linea',
    '10': 'Optimism',
    '1': 'Ethereum'
}

chain_colors = {
    '42161': '#0033ad', 
    '324': '#76e0f7',   
    '8453': '#4d88ff', 
    '59144': '#505050',
    '10': '#ff6961',  
    '1': '#627eea'  
}

df_combined['origin_chain_name'] = df_combined['origin_chain'].map(chain_names)
df_combined['destination_chain_name'] = df_combined['destination_chain'].map(chain_names)
df_combined['origin_color'] = df_combined['origin_chain'].map(chain_colors)
df_combined['destination_color'] = df_combined['destination_chain'].map(chain_colors)

unique_chains = pd.concat([df_combined['origin_chain_name'], df_combined['destination_chain_name']]).unique()
chain_to_idx_origin = {chain: idx for idx, chain in enumerate(unique_chains)}
chain_to_idx_destination = {chain: idx + len(unique_chains) for idx, chain in enumerate(unique_chains)}

sources = df_combined['origin_chain_name'].map(chain_to_idx_origin)
targets = df_combined['destination_chain_name'].map(chain_to_idx_destination)
values = [1] * len(df_combined)  
link_colors = df_combined['origin_color'].tolist()
node_colors = [chain_colors[next(key for key, value in chain_names.items() if value == name)] for name in unique_chains]
labels = [name for name in unique_chains] + [name for name in unique_chains]

fig = go.Figure(go.Sankey(
    node=dict(
        pad=15,
        thickness=20,
        line=dict(color="black", width=0.5),
        label=labels,
        color=node_colors 
    ),
    link=dict(
        source=sources,
        target=targets,
        value=values,
        color=link_colors  
    )
))


fig.update_layout(
    title_text="Flow of Transactions",
    font_size=12,
    annotations=[
        dict(
            text="Origin",
            x=0.05, y=1.1,
            showarrow=False,
            font=dict(size=16, color='black')
        ),
        dict(
            text="Destination",
            x=0.95, y=1.1,
            showarrow=False,
            font=dict(size=16, color='black')
        )
    ]
)

fig.show()
```
![png](/notebookImg/output_38_6.png)

```python
df_allo.to_csv('df_allo.csv', index=False)
attestations_final_df.to_csv('attestations_final_df.csv', index=False)
df_combined.to_csv('df_combined.csv', index=False)
```
