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

1. **Function to retrie TTM EV and EBITDA**

Now, we want to calculate daily EV and EBITDA for every stock in our list. Here we use yahoo finance (yfinance) to obtain financial and market data. Despite the fact that financial documents are standardized, sometimes it may not be so. Thats why its always good to write a robust "error-proof" function that gets the correct and complete information. The Enterprise Value (market value + net debt) is calculated (via yfinance) the following way:
- (+) 'Current Debt And Capital Lease Obligation'
- (+) 'Long Term Debt And Capital Lease Obligation'
- (-) 'Cash Cash Equivalents And Short Term Investments'
- (+) market cap
Where the market cap is obtained by doing 'free floating shares' times 'adjusted close price' for the chosen period. Since we want daily EV/EBITDA, we need to sum this 4 items daily. This is done using this function:
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

````

This function has some redundancies on the calculation of EV, in order to ensure that there will be no missing data.
