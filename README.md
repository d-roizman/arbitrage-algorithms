# Pairs trading algorithm

This repository contains an simple algorithm that searches for EV/EBITDA multiple disparities among a list of stocks. Following, there is a short explanation of it:

1. Obtain daily EV/EBITDA data for the past 3 months for every stock on your list.
2. Test for cointegration between EV/EBITDA series of each pair of correlated stocks.
3. Check if today's ratio of the EV/EBITDA series of these (correlated) stocks is significantly above (or below) it's 3 months average.

## Usage
1. **Import packages**
```bash
import yfinance as yf
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
import datetime
```

1. **Function to retrieve TTM EV and EBITDA**

Now, we want to calculate daily EV and EBITDA for every stock in our list. Here we use yahoo finance (yfinance) to obtain financial and market data. Despite the fact that financial documents are standardized, sometimes it may not be so. Thats why its always good to write a robust "error-proof" function that gets the correct and complete information. The Enterprise Value (market value + net debt) is calculated (via yfinance) the following way:
- (+) 'Current Debt And Capital Lease Obligation'
- (+) 'Long Term Debt And Capital Lease Obligation'
- (-) 'Cash Cash Equivalents And Short Term Investments'
- (+) market cap

Where market cap is obtained by doing 'free floating shares' times 'adjusted close price' for the chosen period. Since we want daily EV/EBITDA, we need to sum this 4 items daily. This is done using this function:
```bash

def TTM_ev_ebitda (tickers) :
    
    # Get TTM EBITDA data
    data = yf.download(tickers, period="3mo")['Adj Close']
    
    EV_EBITDA_series = {}
    today = datetime.date.today().strftime("%Y-%m-%d")
    
    for tick in tickers:
        try: # tick = "AMZN"
            stock = yf.Ticker(tick)
    
            # Get first and second-last release dates
            second, first = pd.DataFrame(stock.earnings_dates).dropna(how='any').iloc[:2].index
            second, first = first.strftime("%Y-%m-%d"), second.strftime("%Y-%m-%d")
    
            # Get first and second-last LTM EBITDAs (conferir se 'bate' o primeiro e o segundo LTM ebitdas)
            ebitdas = stock.quarterly_income_stmt.loc['EBITDA']
            ebitda_series = pd.DataFrame(data=[sum(ebitdas.iloc[1:5]), sum(ebitdas.iloc[:4]), sum(ebitdas.iloc[:4])], 
                                         index = pd.to_datetime([first, second, today]))
    
            # Get mkt cap and net debt
            balance = stock.quarterly_balance_sheet.iloc[:,:2] # last two quarters
    
                # (short term debt)
            if 'Current Debt And Capital Lease Obligation' in balance.index:
                st_term_debt = balance.loc['Current Debt And Capital Lease Obligation']
            elif 'Current Debt' in balance.index:
                st_term_debt = balance.loc['Current Debt']
            elif 'Capital Lease Obligations' in balance.index: 
                st_term_debt = balance.loc['Capital Lease Obligations']
            
                # (long term debt)
            if 'Long Term Debt And Capital Lease Obligation' in balance.index:
                lg_term_debt = balance.loc['Long Term Debt And Capital Lease Obligation']
            elif 'Long Term Debt' in balance.index:
                lg_term_debt = balance.loc['Long Term Debt']
    
                # (cash and cash equivalents)
            if 'Cash Cash Equivalents And Short Term Investments' in balance.index:
                cash = balance.loc['Cash Cash Equivalents And Short Term Investments']
            elif 'Cash And Cash Equivalents' in balance.index:
                cash = balance.loc['Cash And Cash Equivalents']
            
            net_debt = lg_term_debt + st_term_debt - cash 
            net_debt_series = pd.DataFrame([net_debt[0], net_debt[1], net_debt[1]], index = pd.to_datetime([first, second, today]))
    
            market_cap = (data[tick] * stock.info['sharesOutstanding'])
     
            # Forward fill  EBITDA and net debt values to match the specified daily frequency
            ebitda_series = ebitda_series.resample('D').ffill() #.reindex(data.index, method = 'ffill')
            net_debt_series = net_debt_series.resample('D').ffill() #.reindex(data.index, method = 'ffill')
                    
            # Calculate, EV/EBITDA series only for last 3 months
            EV_EBITDA_series[tick] = ((market_cap + net_debt_series[0]) / ebitda_series[0])[
                data.index[0].strftime("%Y-%m-%d"):
                    data.index[len(data.index)-1].strftime("%Y-%m-%d")].dropna()

        except KeyError:
            tickers.remove(tick)
            continue
    
    return pd.DataFrame(EV_EBITDA_series)
```

This function has some redundancies on the calculation of EV, in order to ensure that there will be no missing data.

2. **Testing for cointegration**

   Cointegration is a statistical property of time series. Cointegration-testing decides whether two non-stationary time series are "integrated", which means that they have a long-term equilibrium relationship. If two stocks are cointegrated, we should expect that, after great deviations, they should converge again. Pairs-Trading is based on that idea. In this code, I used the [Johansen test](https://www.math.ku.dk/bibliotek/arkivet/preprints-fra-ims/1989/preprint_1989_-_no_3_johansen__s_ren_-_estimation_and_hypothesis_testing_of_cointegration---.pdf) for cointegration, which is avaliable whithin the package `statsmodels`.

   Nevertheless, instead of testing for the price series, I tested for the EV/EBITDAs time series. Before performing Johansen's cointegration test, though, I filtered my data a little bit, by choosing only correlated (50% or more) EV/EBITDA series.

```bash
from statsmodels.tsa.stattools import coint

# function to check possible candidates for cointegration test (choosing 50%+ correlated stocks only)
def get_correlated_stocks(data, correlation_threshold):

    correlation_matrix = data.corr(numeric_only=False)
    correlated_stocks = {}
    tickers = list(correlation_matrix.keys())
    while tickers: 
        x = tickers.pop()
        x_correlations = correlation_matrix[x]
        for y in tickers:
            if y != x and x_correlations[y] > correlation_threshold:
                if x in correlated_stocks.keys():
                    correlated_stocks[x].append(y)
                else:
                    correlated_stocks[x] = [y]
    return correlated_stocks

correlated_stocks = get_correlated_stocks(ev_ebitda_data, correlation_threshold = 0.9)
#correlated_stocks

# function to perform Johansen cointegration test for every pair of highly pseudo-correlated stocks
def get_cointegrated_stocks(data): # how does the test work ?
    
    cointegrated_stocks = {}    
    correlated_stocks = get_correlated_stocks(data, 0.5)
    tickers = list(correlated_stocks.keys())
    while tickers:
        x = tickers.pop()
        x_data = data[x]
        for y in correlated_stocks[x]:
            y_data = data[y]
            
            # Perform Cointegration test and discard t_statistics more than 5%
            t_statistic, p_val, critical_p_val = coint(x_data,y_data)
            if t_statistic < critical_p_val[1]:
                if x in cointegrated_stocks.keys():
                    cointegrated_stocks[x].append(y)
                else:
                    cointegrated_stocks[x] = [y]
    
    pairs = []
    keys = list(cointegrated_stocks.keys())
    while keys:
        key = keys.pop()
        pairs += [(key, value) for value in cointegrated_stocks[key] if value not in keys]
    
    return pairs

cointegrated_stocks = get_cointegrated_stocks(ev_ebitda_data)
cointegrated_stocks

```

Now we have a short list of a few cointegrated EV/EBITDA series that could offer arbitrage opportunities. The last thing to do is checking if any of those (cointegrated) series is currently deviated from its "equilibrium".

3. **Buy/Sell Signals**

```bash
   import numpy as np

# Calculate EV/EBITDA ratios
for x, y in cointegrated_stocks:
    ratio = ev_ebitda_data[x]/ev_ebitda_data[y]
    avg = np.average(ratio)
    stdev = np.std(ratio)
    
    if ratio[-1] > avg + 2 * stdev:
        print(f'Short {x}, Buy {y}')

        # plot ev/ebitda ratios
        plt.plot(ratio)
        plt.axhline(avg, color = 'black')
        plt.axhline(avg + 2*stdev, color = 'grey')
        plt.axhline(avg - 2*stdev, color = 'grey')
        plt.title(f' {x}/{y} EV/EBITDA ratio')
        plt.show()

    elif ratio[-1] < avg - 2 * stdev:
        print(f'Short {y}, Buy {x}')

        # plot ev/ebitda ratios
        plt.plot(ratio)
        plt.axhline(avg, color = 'black')
        plt.axhline(avg + 2*stdev, color = 'grey')
        plt.axhline(avg - 2*stdev, color = 'grey')
        plt.title(f' {x}/{y} EV/EBITDA ratio')
        plt.show()

```

If any of those cointegrated EV/EBITDA series is more (or less) than two standard deviations from its 3 months average, we should investigate it, because it is possibly a good trade opportunity!

   
