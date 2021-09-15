# dca_bot
A basic Coinbase Pro buying bot that completes trades in any of their available market pairings

Relies on [gdax-python](https://github.com/danpaquin/gdax-python). Props to [danpaquin](https://github.com/danpaquin) and thanks!

## Trading Philosophy
### Basic investing strategy: Dollar Cost Averaging
You have to be extremely lucky or extremely good to time the market perfectly. Rather than trying to achieve the perfect timing for when to execute a purchase just set up your investment on a regular schedule. Buy X amount every Y days. Sometimes the market will be up, sometimes down. But over time your cache will more closely reflect the average market price with volatile peaks and valleys averaged out.

This approach is common for retirement accounts; you invest a fixed amount into your 401(k) every month and trust that the market trend will be overall up over time.

### Micro Dollar Cost Averaging for cryptos
While I believe strongly in dollar cost averaging, the crypto world is so volatile that making a single, regular buy once a month is still leaving too much to chance. The market can swing 30%, 50%, even 100%+ in a single day. I'd rather invest $20 every day for a month than agonize over deciding on just the right time to do a single $600 buy.

## Technical Details
### Basic approach
dca_bot pulls the current bid and ask prices, averages the two to set our order's price, then submits it as a limit order.

### Making a valid limit order
Buy orders will be rejected if they are at or above the lowest sell order (think: too far right on the order book) (see: https://stackoverflow.com/a/47447663) and vice-versa for sells. When the price is plummeting this is likely to happen. In this case dca_bot will pause for a minute and then grab the latest price and re-place the order. It will currently attempt this 100 times before it gives up.

_*Longer pauses are probably advantageous--if the price is crashing, you don't want to be rushing in._

### Setup
#### Create a Coinbase account

#### Install pip modules
```
pip install -r requirements.txt
pip install boto3
pip install gspread oauth2client
```

#### Create Coinbase Pro API key
Try this out on Coinbase Pro's sandbox first. The sandbox is a test environment that is not connected to your actual fiat or crypto balances.

Log into your Coinbase/Coinbase Pro account in their test sandbox:
https://public.sandbox.pro.coinbase.com/

Find and follow existing guides for creating an API key. Only grant the "Trade" permission. Note the passphrase, the new API key, and API key's secret.

While you're in the sandbox UI, fund your fiat account by transferring from the absurd fake balance that sits in the linked Coinbase account (remember, this is all just fake test data; no real money or crypto goes through the sandbox).

#### (Optional) Create an AWS Simple Notification System topic
This is out of scope for this document, but generate a set of AWS access keys and a new SNS topic to enable the bot to send email reports.

Set these values in the `settings-local.conf` file if the SNS topic was created

```
SNS_TOPIC = arn:aws:sns:us-east-1:account_id:topicname
AWS_ACCESS_KEY_ID = access_key
AWS_SECRET_ACCESS_KEY = secret_key
AWS_REGION = us-east-1
```
#### (Optional) Create a Google Spreadsheet

Set these values in the `settings-local.conf` file if the Google spreadsheet was created

```
GOOGLE_SPREADSHEET_KEY=key_of_google_spreadsheet (i.e. 1KArbyA-s2IJwxP6zqazZ3IzkLNFHBFzek9HLziB6ITw)
```

#### Customize settings
Update ```settings.conf``` with your API key info in the "sandbox" section. I recommend saving your version as ```settings-local.conf``` as that is already in the ```.gitignore``` so you don't have to worry about committing your sensitive info to your forked repo.

If you have an AWS SNS topic, enter the access keys and SNS topic.

#### Try a basic test run
Run against the Coinbase Pro sandbox by including the ```-sandbox``` flag. Remember that the sandbox is just test data. The sandbox only supports BTC trading.

Activate your virtualenv and try a basic buy of $100 USD worth of BTC:
```
python3 dca_bot.py BTC-USD BUY 100 USD -sandbox -c ../settings-local.conf
```

Check the sandbox UI and you'll see your limit order listed. Unfortunately your order probably won't fill unless there's other activity in the sandbox.

### Usage
Run ```python3 dca_bot.py -h``` for usage information:

```
usage: dca_bot.py [-h] [-sandbox] [-warn_after WARN_AFTER] [-j]
                   [-c CONFIG_FILE]
                   market_name {BUY,SELL} amount amount_currency

        This is a basic Coinbase Pro DCA buying/selling bot.

        ex:
            BTC-USD BUY 14 USD          (buy $14 worth of BTC)
            BTC-USD BUY 0.00125 BTC     (buy 0.00125 BTC)
            ETH-BTC SELL 0.00125 BTC    (sell 0.00125 BTC worth of ETH)
            ETH-BTC SELL 0.1 ETH        (sell 0.1 ETH)
    

positional arguments:
  market_name           (e.g. BTC-USD, ETH-BTC, etc)
  {BUY,SELL}
  amount                The quantity to buy or sell in the amount_currency
  amount_currency       The currency the amount is denominated in

optional arguments:
  -h, --help            show this help message and exit
  -sandbox              Run against sandbox, skips user confirmation prompt
  -warn_after WARN_AFTER
                        secs to wait before sending an alert that an order isn't done
  -j, --job             Suppresses user confirmation prompt
  -c CONFIG_FILE, --config CONFIG_FILE
                        Override default config file location
```

### Scheduling your recurring buys
This is meant to be run as a crontab to make regular purchases on a set schedule. Here are some example cron jobs:

$50 USD of ETH every Monday at 17:23:
```
23 17 * * 1 /your/virtualenv/path/bin/python -u /your/dca_bot/path/src/dca_bot.py -j ETH-USD BUY 50.00 USD -c /your/settings/path/your_settings_file.conf >> /your/cron/log/path/cron.log
```
*The ```-u``` option makes python output ```stdout``` and ```stderr``` unbuffered so that you can watch the progress in real time by running ```tail -f cron.log```.*

### Unfilled orders will happen
The volatility may quickly carry the market away from you. Here we see a bunch of unfilled orders that are well below the current market price of $887.86:
![alt text](https://github.com/kdmukai/dca_bot/blob/master/docs/img/gdax_unfilled_orders.png "Unfilled dca_bot orders")

The dca_bot will keep checking on the status of the order for up to an hour, after which it will report it as OPEN/UNFILLED. Hopefully the market will cool down again and return to your order's price, at which point it will fill (though dca_bot will not send a notification). You can also manually cancel the order to free up the reserved fiat again. 

I would recommend patience and let the unfilled order ride for a few hours or days. With micro dollar cost averaging it doesn't really matter if you miss a few buy orders.

#### Mac notes
Edit the crontab:
```
env EDITOR=nano crontab -e
```

View the current crontab:
```
crontab -l
```
