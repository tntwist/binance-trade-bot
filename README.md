# binance-trade-bot

![github](https://img.shields.io/github/workflow/status/edeng23/binance-trade-bot/binance-trade-bot)
![docker](https://img.shields.io/docker/pulls/idkravitz/binance-trade-bot)

> Automated cryptocurrency trading bot

## Why?

This project was inspired by the observation that all cryptocurrencies pretty much behave in the same way. When one spikes, they all spike, and when one takes a dive, they all do. _Pretty much_. Moreover, all coins follow Bitcoin's lead; the difference is their phase offset.

So, if coins are basically oscillating with respect to each other, it seems smart to trade the rising coin for the falling coin, and then trade back when the ratio is reversed.

## How?

The trading is done in the Binance market platform, which of course, does not have markets for every altcoin pair. The workaround for this is to use a bridge currency that will complement missing pairs. The default bridge currency is Tether (USDT), which is stable by design and compatible with nearly every coin on the platform.

<p align="center">
  Coin A → USDT → Coin B
</p>

The way the bot takes advantage of the observed behaviour is to always downgrade from the "strong" coin to the "weak" coin, under the assumption that at some point the tables will turn. It will then return to the original coin, ultimately holding more of it than it did originally. This is done while taking into consideration the trading fees.

<div align="center">
  <p><b>Coin A</b> → USDT → Coin B</p>
  <p>Coin B → USDT → Coin C</p>
  <p>...</p>
  <p>Coin C → USDT → <b>Coin A</b></p>
</div>

The bot jumps between a configured set of coins on the condition that it does not return to a coin unless it is profitable in respect to the amount held last. This means that we will never end up having less of a certain coin. The risk is that one of the coins may freefall relative to the others all of a sudden, attracting our reverse greedy algorithm.

## Binance Setup

-   Create a [Binance account](https://accounts.binance.com/en/register?ref=DDA1SQA) (Includes my referral link, I'll be super grateful if you use it).
-   Enable Two-factor Authentication.
-   Create a new API key and whitelist your IP for it.
-   Get a cryptocurrency. If its symbol is not in the default list, add it.

## Tool Setup

### Required Python version

Currently Python 3.7 is the minimum required version.

### Install Python dependencies

Run the following line in the terminal: `pip install -r requirements.txt`.

### Create user configuration

Create a .cfg file named `user.cfg` based off `.user.cfg.example`, then add your API keys and current coin.

**The configuration file consists of the following fields:**

-   **api_key** - Binance API key generated in the Binance account setup stage.
-   **api_secret_key** - Binance secret key generated in the Binance account setup stage.
-   **current_coin** - This is your starting coin of choice. This should be one of the coins from your supported coin list. If you want to start from your bridge currency, leave this field empty - the bot will select a random coin from your supported coin list and buy it.
-   **bridge** - Your bridge currency of choice. Notice that different bridges will allow different sets of supported coins. For example, there may be a Binance particular-coin/USDT pair but no particular-coin/BUSD pair.
-   **tld** - 'com' or 'us', depending on your region. Default is 'com'.
-   **hourToKeepScoutHistory** - Controls how many hours of scouting values are kept in the database. After the amount of time specified has passed, the information will be deleted.
-   **scout_multiplier** - Controls the value by which the difference between the current state of coin ratios and previous state of ratios is multiplied. For bigger values, the bot will wait for bigger margins to arrive before making a trade.
-   **trade_fee** - Controls trade fee for calculating profitable jumps. By default it doesn't have value: gets values through the binance api calls. Otherwise use float values for the fee. [Binance fee table for reference](https://www.binance.com/en/fee/schedule)
-   **strategy** - The trading strategy to use. See [`binance_trade_bot/strategies`](binance_trade_bot/strategies/README.md) for more information
-   **enable_paper_trading** - (`True` or `False` default `False`) run bot with virtual wallet to check its performance without risking any money.
-   **buy_timeout/sell_timeout** - Controls how many minutes to wait before cancelling a limit order (buy/sell) and returning to "scout" mode. 0 means that the order will never be cancelled prematurely.
-   **scout_sleep_time** - Controls how many seconds bot should wait between analysis of current prices. Since the bot now operates on websockets this value should be set to something low (like 1), the reasons to set it above 1 are when you observe high CPU usage by bot or you got api errors about requests weight limit.
-   **buy_order_type** - Controls the type of placed buy orders, types available: market, limit (default=limit)
-   **sell_order_type** - Controls the type of placed sell orders, types available: market, limit (default=market)
-   **buy_max_price_change/sell_max_price_change** - Controls how much price change in decimal percentage is accepted between calculation of ratios and trading.
-   **price_type** - Controls the type of prices used by the bot, types available: orderbook, ticker (default=orderbook). Please note that using the orderbook prices increase the CPU usage.
-   **ratio_calc** - Controls the calculation for ratios between coins. Options available: default, scout_margin. Keep in mind you need to decrease your scout_multiplier for the scout_margin. A scout_multiplier of 12 with the default calc would be a scout_multiplier of 2 with the scout_margin calc.
-   **accept_losses** - Needs to be set to true for highly risky and gamling strategies. Otherwise the bot wont start.
-   **auto_adjust_bnb_balance** - Controls the bot to auto buy BNB while there is no enough BNB balance in your account, to get the benifits of using BNB to pay the commisions. Default is false. Effective if you have enabled to [use BNB to pay for any fees on the Binance platform](https://www.binance.com/en/support/faq/115000583311-Using-BNB-to-Pay-for-Fees), reade more information [here](#paying-fees-with-bnb).
-   **auto_adjust_bnb_balance_rate** - The multiplying power of buying quantity of BNB compares to evaluated comission of the coming order, effective only if auto_adjust_bnb_balance is true. Default value is 3.
-   **allow_coin_merge** - Allow multiple_coins strategy to merge coins into one. It turned out that its more profitable if the strategy can merge the held coins into one. If you dont want this you may be safer for one falling coin but you also pay with potential gains. Default is true to ensure the behavior is like in the origin repo.

#### Environment Variables

All of the options provided in `user.cfg` can also be configured using environment variables.

```
CURRENT_COIN_SYMBOL:
BRIDGE_SYMBOL: USDT
API_KEY: vmPUZE6mv9SD5VNHk4HlWFsOr6aKE2zvsw0MuIgwCIPy6utIco14y7Ju91duEh8A
API_SECRET_KEY: NhqPtmdSJYdKjVHjA7PZj4Mge3R5YNiP1e3UZjInClVN65XAbvqqM6A7H5fATj0j
SCOUT_MULTIPLIER: 5
SCOUT_SLEEP_TIME: 1
TLD: com
STRATEGY: default
BUY_TIMEOUT: 0
SELL_TIMEOUT: 0
BUY_ORDER_TYPE: limit
SELL_ORDER_TYPE: market
AUTO_ADJUST_BNB_BALANCE: false
AUTO_ADJUST_BNB_BALANCE_RATE: 3
ALLOW_COIN_MERGE: true
```

### Paying Fees with BNB
You can [use BNB to pay for any fees on the Binance platform](https://www.binance.com/en/support/faq/115000583311-Using-BNB-to-Pay-for-Fees), which will reduce all fees by 25%. In order to support this benefit, the bot will always perform the following operations:
-   Automatically detect that you have BNB fee payment enabled.
-   Make sure that you have enough BNB in your account to pay the fee of the inspected trade.
-   Take into consideration the discount when calculating the trade threshold.

### Notifications with Apprise

Apprise allows the bot to send notifications to all of the most popular notification services available such as: Telegram, Discord, Slack, Amazon SNS, Gotify, etc.

To set this up you need to create a apprise.yml file in the config directory.

There is an example version of this file to get you started.

If you are interested in running a Telegram bot, more information can be found at [Telegram's official documentation](https://core.telegram.org/bots).

### Run

```shell
python -m binance_trade_bot
```

### Docker

The official image is available [here](https://hub.docker.com/r/idkravitz/binance-trade-bot) and will update on every new change.

```shell
docker-compose up
```

If you only want to start the SQLite browser

```shell
docker-compose up -d sqlitebrowser
```

## Backtesting

You can test the bot on historic data to see how it performs.

```shell
python backtest.py
```

Feel free to modify that file to test and compare different settings and time periods

## Database warmup

You can warmup your database with coins wich you might want to add later to your supported coin list. 
This should prevent uncontrolled jumps when you add a new coin to your supported coin list.

After the execution you should wait one or two trades of the bot before adding any new coin to your list.

By running the script without parameters, it will warm up the bots default database with all available coins for the bridge.

```shell
python3 database_warmup.py
```

If you want to specify a separate db file you can use the -d or --dbfile parameter.
If not provided, the script will use the bots default db file.

```shell
python3 database_warmup.py -d data/warmup.db
```

You can also specify the coins you want to warmup with the -c or --coinlist parameter.
If not provided the script will warmup all coins available for the bridge.

```shell
python3 database_warmup.py -c 'ADA BTC ETH LTC'
```

## Developing

To make sure your code is properly formatted before making a pull request,
remember to install [pre-commit](https://pre-commit.com/):

```shell
pip install pre-commit
pre-commit install
```

The scouting algorithm is unlikely to be changed. If you'd like to contribute an alternative
method, [add a new strategy](binance_trade_bot/strategies/README.md).

## Related Projects

Thanks to a group of talented developers, there is now a [Telegram bot for remotely managing this project](https://github.com/lorcalhost/BTB-manager-telegram).

## Support the Project

Fist of all, support the originator of this bot and buy him a coffee. ☕

<a href="https://www.buymeacoffee.com/edeng" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" height="41" width="174"></a>

If you like my adjustments and you want to support me I would appreciate to get a coffee too. 😜

<a href="https://www.buymeacoffee.com/tntwist" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" height="41" width="174"></a>

## Join the Chat

-   **Discord**: [Invite Link](https://discord.gg/m4TNaxreCN)

## FAQ

A list of answers to what seem to be the most frequently asked questions can be found in our discord server, in the corresponding channel.

<p align="center">
  <img src = "https://usercontent2.hubstatic.com/6061829.jpg">
</p>

## Disclaimer

This project is for informational purposes only. You should not construe any
such information or other material as legal, tax, investment, financial, or
other advice. Nothing contained here constitutes a solicitation, recommendation,
endorsement, or offer by me or any third party service provider to buy or sell
any securities or other financial instruments in this or in any other
jurisdiction in which such solicitation or offer would be unlawful under the
securities laws of such jurisdiction.

If you plan to use real money, USE AT YOUR OWN RISK.

Under no circumstances will I be held responsible or liable in any way for any
claims, damages, losses, expenses, costs, or liabilities whatsoever, including,
without limitation, any direct or indirect damages for loss of profits.
