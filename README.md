# Pairs trading algorithm
This repository contains an algorithm that searches for EV-EBITDA multiple disparities among a list of stocks. Following, there is a short explanation of it:

1. Obtain daily EV-EBITDA data for the past 3 months for every stock on your list.
2. Test for cointegration between EV-EBITDA series of each pair of correlated stocks.
3. Check if today's ratio of the EV-EBITDA series of these (correlated) stocks is significantly above (or below) it's 3 months average.
