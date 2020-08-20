---
layout: post
title: "Buy and Sell Stock (LeetCode 121, 122, 123, 188, 309, 714)"
date: 2020-08-20
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## 1. 简介

本文对LeetCode中"Best Time to Buy and Sell Stock"系列进行总结，题目包括：
- [[LeetCode 121](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)] Best Time to Buy and Sell Stock
- [[LeetCode 122](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)] Best Time to Buy and Sell Stock II
- [[LeetCode 123](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/)] Best Time to Buy and Sell Stock III
- [[LeetCode 188](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iv/)] Best Time to Buy and Sell Stock IV
- [[LeetCode 309](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/)] Best Time to Buy and Sell Stock with Cooldown
- [[LeetCode 714](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-transaction-fee/)] Best Time to Buy and Sell Stock with Transaction Fee

通过链接能查阅得到题目描述，在此不赘述。


## 2. Buy and Sell

从最基本的情形出发：给定一维数组表示股票每天的价格，寻找利润最多的一段时间，并返回最大金额。

思路：
1. 动态规划
2. 贪心

### 2.1 动态规划

> $$dp[i]$$表示从第0天～第$$i$$天最多能挣多少钱  \\
> 状态转移方程：$$dp[i] = max(dp[i-1], prices[i] - minPrice)$$  \\
> 其中$$prices[i]$$表示第$$i$$天价格，minPrice表示第0天～第$$i$$天最低价格

不难理解：第0天～第$$i$$天能挣最多的钱，要不就是从某个最低点买入并在第$$i$$天卖出，要不就是第0天～第$$i-1$$天能挣的钱。

时间复杂度：$$O(n)$$  \\
空间复杂度：$$O(n)$$

```C
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        if(prices.size() == 0 || prices.size() == 1)
            return 0;
        // DP
        // dp[i] represent the maximum profit from day 0 to day i
        // dp[i] = max(dp[i-1], prices[i] - minPrice)
        // Time: O(n)
        // Space: O(n)
        int minPrice = prices[0];
        vector<int> dp(prices.size(), 0);
        for(int i=1; i<prices.size(); i++){
            dp[i] = max(dp[i-1], prices[i] - minPrice);
            if(prices[i] < minPrice)
                minPrice = prices[i];
        }//for
        return dp[prices.size() - 1];
};
```

### 2.2 贪心

在使用动态规划求解过程中，使用了一维数组来存储从第0天到第$$i$$天每天的最大利润，这是不必要的，我们只需要得到最后一天能挣多少钱，将上述过程简化为贪心即可避免中间变量存储。

使用变量记录当前最大利润，计算每天最大利润后，更新变量。循环完毕后，该变量即为最终结果。

时间复杂度：$$O(n)$$  \\
空间复杂度：$$O(1)$$

```C
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        if(prices.size() == 0 || prices.size() == 1)
            return 0;
        // Greedy
        // Time: O(n)
        // Space: O(1)
        int minPrice = INT_MAX;
        int maxProfit = 0;
        for(int i=0; i<prices.size(); i++){
            if(prices[i] < minPrice)
                minPrice = prices[i];
            else
                maxProfit = max(maxProfit, prices[i] - minPrice);
        }//for
        return maxProfit;
    }
};
```

## 3. Buy and Sell II

这道题比第一题多了些条件：1. 允许进行多次买卖；2. 多次买卖之间不能重合，即必须卖出去了才能买下一个股票。

<p align="center">
	<img src="/assets/leetcode-best-time-to-buy-ans-sell-stock/buy_and_sell_ii.png" width=350>
</p>

题目经过更改后，反而更简单了。如上图所示，如果能够进行多次买卖，只需要把所有涨的部分买下来就可以了。

思路是寻找所有增长的区间，实际上等于找数组中相邻两个数字是上升的所有数字对，将它们的差求和。

时间复杂度：$$O(n)$$  \\
空间复杂度：$$O(1)$$

```C
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        int ans = 0;
        for(int i=1; i<prices.size(); i++){
            if(prices[i] > prices[i-1])
                ans += prices[i] - prices[i-1];
        }//for
        return ans;
    }
};
```

## 4. Buy and Sell III

这道题相比于第二题修改了第1个条件：最多只能进行**2次**交易，不再是无限次了。

这题的解题思路要从第一题的动态规划法出发：在第一题的动态规划法中使用了一维数组记录了从第0天到第$$i$$天的最大利润，如果能够进行两次买卖的话，那么思考如下场景：A点买入，B点卖出，C点（在B点后）买入，D点卖出。其中A买B卖的过程可以用第一题的动态规划数组记录下来，C买D卖的过程怎么获得呢？答案仍然是动态规划。但是数组dp[i]记录的是从第$$i$$天到最后一天能够最大利润。

所以，这道题用两次DP可以解决：
1. 第一次DP：与第一题一样，$$dp1[i]$$表示从第0天～第$$i$$天最多能挣多少钱；
2. 第二次DP：反方向DP，$$dp2[i]$$表示从第$$i$$天到最后一天能够最大利润；

最多进行两次交易获得最大利润为：$$max(dp1[i] + dp2[i+1]) for \ i \ in \ range(0, n)$$

时间复杂度：$$O(2*n)$$  \\
空间复杂度：$$O(2*n)$$

```C
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        if(prices.size() <= 1)
            return 0;
        int ans = 0;
        int inOrder[prices.size()+1], minnum = prices[0];
        memset(inOrder, 0, sizeof(inOrder));
        for(int i=1; i<prices.size(); i++){
            inOrder[i] = max(prices[i] - minnum, inOrder[i-1]);
            if(prices[i] < minnum)
                minnum = prices[i];
        }//for
        
        int reOrder[prices.size()+1], maxnum = prices[prices.size()-1];
        memset(reOrder, 0, sizeof(reOrder));
        for(int i=prices.size()-2; i>=0; i--){
            reOrder[i] = max(maxnum - prices[i], reOrder[i+1]);
            if(prices[i] > maxnum)
                maxnum = prices[i];
        }//for
        
        ans = inOrder[prices.size()-1];
        for(int i=0; i<prices.size()-1; i++)
            ans = max(ans, inOrder[i] + reOrder[i+1]);

        return ans;
    }
};
```

## 5. Buy and Sell IV

这一题再对第三题进行泛化，求最多交易**k次**最大利润。此时仍然使用动态规划解决：
1. 在某个时间点买入相当于减掉当天的价格，在某个时间点卖出相当于加上当天的价格，在第$$i$$天买第$$j$$天卖，利润为$$-prices[i] + prices[j]$$。
2. $$localMax$$记录到目前为止，仍持有股票没有卖出状态下的最大利润。
3. $$dp[i][j]$$表示从第0天到第$$i$$天，使用$$j$$次交易能达到的最大利润，此时交易已完成，不持有股票。
4. 状态转移方程：$$dp[i][j] = max(dp[i-1][j], prices[i] + localMax)$$

不难理解，$$dp[i][j]$$表示**完成了**$$j$$次交易（重要），此时有两种可能：1. 第$$i$$天不交易（不买不卖），利润为$$dp[i-1][j]$$；2.第$$i$$天卖出，利润为$$prices[i] + localMax$$。$$localMax$$记录的是在第0天到第$$i-1$$天某天买入的最大利润（使用$$j-1$$次交易），第$$i$$天卖出后就完成了第$$j$$次交易。

那么在第$$i$$天买入的情况呢？答案是用$$localMax$$来记录。$$localMax = max(localMax, dp[i-1][j-1] - prices[i])$$，该式表示第0天到第$$i-1$$天的交易完成情况下的最大利润，加上第$$i$$天买入，如果此时得到的钱是最多的，那么记录下来，在未来的某个日子卖掉该股。$$localMal$$计算的式子理解为：尝试一下在第$$i$$天开启第$$j$$次交易，看看有没有可能获得更大的列润，所以它是用$$dp[i-1][j-1]$$的值来减去$$prices[i]$$。

此题有一个细节，如果$$k$$大与总天数的一半，那么相当于可以任意买卖，最大利润计算方法回到第二题。

时间复杂度：$$O(k*n)$$  \\
空间复杂度：$$O(k*n)$$


```C
class Solution {
public:
    int maxProfit(int k, vector<int>& prices) {
        if(k == 0 || prices.size() <= 1)
            return 0;
        if(k >= prices.size() / 2)
            return getMaxProfit(prices);
        /*
            Buying a stock is equivalent to subtracting the price of that day.
            Selling a stock is equivalent to adding the price of that day.
            Suppose that you buy on day i and sell on day j, the profit 
            will be -prices[i] + prices[j].
            
            localMax means that the maximum profit until now with one 
            stock waiting to be sold.
            
            dp[i][j] means that the maximum profit from day 0 to i 
            using j transaction.
            dp[i][j] = max(dp[i-1][j], prices[i] + localMax)
        */
        int dp[prices.size()+1][k+1], localMax;
        memset(dp, 0, sizeof(dp));
        for(int j=1; j<=k; j++){
            localMax = -prices[0];  // Buy on day 1 (the stock waiting to be sold)
            for(int i=1; i<prices.size(); i++){
                dp[i][j] = max(dp[i-1][j], prices[i] + localMax);
                localMax = max(localMax, dp[i-1][j-1] - prices[i]);  // Buy on day i
            }//for
        }//for
        return dp[prices.size()-1][k];
    }
    
    int getMaxProfit(vector<int>& prices){
        int ret = 0;
        for(int i=1; i<prices.size(); i++){
            if(prices[i] > prices[i-1])
                ret += (prices[i] - prices[i-1]);
        }//for
        return ret;
    }
};
```

## 6. Buy and Sell with Cooldown

此题目在第二题的基础上增加条件，第$$i$$天卖出后，第$$i+1$$天要休息一下（Cooldown）。

动态规划：
1. $$dp[i]$$表示从第0天到第$$i$$天的已经Cooldown过的最大利润，$$i+1$$天可以买入了。
2. $$localMax$$表示到现在为止，手上持有股票等着卖出的最大利润。
3. 状态转移方程：$$dp[i] = max(dp[i-1], prices[i-1] + localMax)$$

不难理解，第$$i$$天的最大利润有两种可能：1. 今天不交易了；2. 在$$i-1$$天的时候卖了，第$$i$$天休息一下。

时间复杂度：$$O(n)$$  \\
空间复杂度：$$O(n)$$

```C
class Solution {
public:
    int maxProfit(vector<int>& prices) {
        if(prices.size() <= 1)
            return 0;
        /*
            dp[i] means that the maximum profit with cooldown.
            You can buy day i+1 now.
            
            localMax means that the maximun profit until now
            with one stock waiting to be sold.
        */
        int dp[prices.size()+2], localMax = -prices[0];  // Day 0 waiting to be sold
        memset(dp, 0, sizeof(dp));
        for(int i=2; i<prices.size()+1; i++){
            dp[i] = max(dp[i-1], prices[i-1] + localMax);  // Sell on day i-1, cooldown on day i
            localMax = max(localMax, dp[i-2] - prices[i-1]);  // Buy on day i-1
        }//for
        return dp[prices.size()];
    }
};
```

## 7. Buy and Sell with Transaction Fee

这一题是从第二题引申过来的，即可以无限买卖，但是买卖存在手续费，如果买卖赚的钱还不够手续费，那么这次买卖就不必要了。

从第一题的贪心算法出发思考：每一次考虑卖出时，都要计算卖出后的钱能不能抵回手续费。

时间复杂度：$$O(n)$$  \\
空间复杂度：$$O(1)$$

```C
class Solution {
public:
    int maxProfit(vector<int>& prices, int fee) {
        if(prices.size() <= 1)
            return 0;
        int ans = 0, minPrice = prices[0];
        for(int i=1; i<prices.size(); i++){
            if(prices[i] < minPrice)
                minPrice = prices[i];
            else if(prices[i] > minPrice + fee){  // can be sold
                ans += prices[i] - minPrice - fee;
                minPrice = prices[i] - fee;  // avoid duplicative subtraction
            }//if-else
        }//for
        return ans;
    }
};
```
