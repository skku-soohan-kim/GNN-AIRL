### 성균관대학교 소프트웨어학과 졸업작품 (2017310734 김수한)

## Commands

1. Getting Data
    
    `python3 get_data.py`

2. Preprocess

    `python3 preprocess.py -s_date={s_date} -e_date={e_date} -m_trd={m_trd} -thres={thres} -num_d={num_d} -use_log={use_log}`

3. Train & Test

    `python3 main.py`
- Change the settings in configs.py to conduct different experiments
- For training, trains with same configs for NASDAQ, NYSE and industry, wiki relations (total 4 rounds)


## Data Source

1. Stock Tickers
- NYSE tickers: https://github.com/fulifeng/Temporal_Relational_Stock_Ranking/blob/master/data/NYSE_tickers_qualify_dr-0.98_min-5_smooth.csv
- NASDAQ tickers: https://github.com/fulifeng/Temporal_Relational_Stock_Ranking/blob/master/data/NASDAQ_tickers_qualify_dr-0.98_min-5_smooth.csv

2. Relation
- nasd_wiki_q.csv: https://github.com/fulifeng/Temporal_Relational_Stock_Ranking/blob/master/data/NASDAQ_wiki.csv
- nyse_wiki_q.csv: https://github.com/fulifeng/Temporal_Relational_Stock_Ranking/blob/master/data/NYSE_wiki.csv
- other json files: https://github.com/fulifeng/Temporal_Relational_Stock_Ranking/blob/master/data/relation.tar.gz


## Methods

1. Fractional Differencing
- Preprocess OHLC features based on Fixed-Width Window Fractional Differencing for each stock

2. Temporal Graph Convolution (TGC) Network 
- input: (N, D, F) [N: num stocks, D: num past days, F: num features]
- output: (N, 1)
- Aims to rank the scores of each stock based on their future sharpe ratio outlook

3. RL Environment (no additional gym.Env class; executed inside AIRL class)
- state: (N + 1, 1) [output of TGC + current portfolio profit]
- action: (M,) [M: number of selected top stocks based on TGC output scores; values denote weights]
- M < R required to calculate expert actions, i.e. Markowitz Max-Sharpe Portfolio Weights [R: rebalancing period]
- reward: calculated w/ updated reward function per iteration
- Configured as to collect single pairs (s, a, s')
- Utilizes TGC & Actor Network inside this class
- Log prob of action given state also calculated

4. Actor Network
- input: (N, D, F)
- flow: input -> TGC -> 0-mask non_selected stocks -> FC -> output
- output: (M, 1) [mean vector for top M stock weights]
- Output and predetermined covariance matrix used to construct Multivariate Gaussian Distribution
- Sample is drawn from distribution and goes through softmax (long-only, sum of weights is 1)

5. Adversarial Inverse Reinforcement Learning
- Actor Network serves the role of Generator
- Discriminator Network: D(s, a, s') = exp(r(s, a, s')) / {exp(r(s, a, s')) + pi(a|s)}
- r(s, a, s') = g(s, a_e) + gamma * h(s') - h(s) [a_e: action vector extended by filling 0's on positions of not selected stocks]
- g, h: fully-connected layers

6. Objective Loss & Optimization
- Discriminator Loss: min{- E[log(D_expert(s, a, s'))] - E[1 - log(D_policy(s, a, s'))]}
- Rank Loss: sum_{i, j}(max(0, -(s_true_i - s_true_j)(s_i - s_j))) [s_i, s_j components of state s (stock scores)]
- Actor Loss: max{log(D_policy(s, a, s')) - log(1 - D_policy(s, a, s')) - alpha * Rank Loss}
- Rank Loss aims to serve similar role as that of critic loss from shared network (actor)
- Rank Loss true values are true sharpe ratios for each stock on trading period (transition period from s to s', i.e. where weights are fixed as a)


## Models

- COMMON DATE SPLIT: TRAIN (20020101 - 20171231), TEST(20180101 - 20211231)
- Following 4 rounds are trained per model configuration as default

1. NASDAQ, Industry Relation

2. NASDAQ, Wiki Relation

3. NYSE, Industry Relation

4. NYSE, Wiki Relation

- samp_model: selected 30 stocks w/ epsilon 1e-8 
=> discriminator was not able to train because of too 
much zero-weights generated by max-sharpe expert portfolio

- sel_10_eps_5e-3: selected 10 stocks w/ epsilon 5e-3

- sel_5_eps_5e-3: selected 5 stocks w/ epsilon 5e-3

- sel_3_eps_5e-3: selected 3 stocks w/ epsilon 5e-3

- sel_3_eps_5e-3_batch_128: increased batch from 50 to 128

- sel_3_eps_5e-3_batch_128_lr_0.1: increased init lr from 0.001 to 0.1

- sel_5_eps_5e-3_batch_256_lr_0.1: increased selected stocks back to 5 & batch from 128 to 256

- sel_3_eps_5e-3_batch_256_lr_0.1: decreased selected stocks back to 3

- actor_loss_entropy: changed actor loss to maximizing log(D) - log(1-D) [[equiv. to entropy-regularized reward, to ensure sufficient exploration]] // configurations are same as sel_3_eps_5e-3_batch_256_lr_0.1


## References

1. https://github.com/fulifeng/Temporal_Relational_Stock_Ranking

2. https://arxiv.org/abs/1809.09441

3. Lopez de Prado, Marcos; Advances in Financial Machine Learning, 75-88

4. https://towardsdatascience.com/infer-your-reward-by-observing-an-expert-140b685fd5b5

5. https://arxiv.org/abs/1710.11248
