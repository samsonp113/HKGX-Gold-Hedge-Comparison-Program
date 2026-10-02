# Gold Futures Hedge Roll Optimizer

**A Python backtest, Excel model and web calculator that find the best time to roll a gold futures hedge and measure what each roll earns.**

Tools for hedging physical gold with COMEX gold futures and timing contract rolls to capture the futures-spot premium. Built during my summer 2025 internship at the Hong Kong Gold Exchange (HKGX).

> **How this was built:** I vibecoded this program in Hong Kong in July - August 2025. I designed the hedging logic, roll-timing method and formulas, and used AI to write the code. ChatGPT and Claude weren't officially offered in Hong Kong, so I used Perplexity Pro, the AI tool I had access to at the time. The code was written mainly with GPT-4.1 and Perplexity's in-house Sonar model. Perplexity Pro's lineup that August also included Claude 4.0 Sonnet (and its Thinking variant), Gemini 2.5 Pro, Grok 4, OpenAI o3 and R1 1776, Perplexity's post-trained version of DeepSeek R1. The "How it works" summary and the conclusion below were also written with AI help, from the code and spreadsheets in this repo.

## Context

HKGX holds physical gold in its vaults. Physical gold sitting in a vault pays no interest, and its value moves with the gold price. To protect that value, HKGX hedges: for every 100 oz it holds, it sells (shorts) one gold futures contract. If the gold price falls, the gold loses value but the short futures gain about the same amount. If the price rises, the reverse happens. Either way, the total value stays roughly fixed.

The hedge also earns money. Gold futures usually trade above the spot price (contango), because the futures price includes the cost of storing and financing gold until delivery. A company that already holds the gold collects that premium when it shorts the futures. The premium shrinks to zero as the contract nears expiry, so before each expiry the hedge is **rolled**: the company buys back the expiring contract and shorts the next one, which resets the premium. Done every two months, the hedge earns something like an interest rate on gold that would otherwise sit idle.

This only works while the market is in contango. If futures fall below spot (backwardation), each roll costs money instead.

How much each roll earns depends on its timing. The best moment is when the expiring contract's premium is nearly gone (cheap to buy back) and the next contract's premium is still wide (good price to sell). These tools answer two questions: **when should we roll**, and **what did the roll earn**.

## How it works

### 1. Roll timing backtest (Python)
`rollover/` has one script for each roll in the cycle: Z4→G5, G5→J5, J5→M5, M5→Q5 and Q5→Z5 (Nov 2024 to Jul 2025). Each script:

- loads hourly prices and volume for both contracts, plus spot gold
- keeps only the evening hours (21:00 - 01:00), when futures volume is heaviest, and only hours with at least 1,500 contracts traded
- computes each contract's premium over spot, hour by hour
- scores every hour by its **spread ratio**: the new contract's premium divided by the expiring contract's premium. A high ratio means a cheap buyback and an expensive short.
- ranks the hours and writes them to Excel (`hedge_roll_*_ratio.xlsx`)

`hedge_roll_master.csv` combines the results of all five rolls.

### 2. Excel model
`Gold Futures Calculator Spreadsheet.xlsx` is the trade ledger, with a formula reference sheet. It has four tabs:

- **Settlement** - hold the hedge to expiry
- **Buyback** - close the hedge early
- **Buy Sell Settle / Buy Sell Buy** - multi-leg rolls that carry the cost of the previous buyback into the next contract

### 3. Web calculator (HTML/JS)
The same model as the spreadsheet, as a browser app with no install. Enter prices, contracts, and timestamps to get the net premium and annualized yield. Days held are measured to the minute, and you can save calculations into named sheets and export them to CSV.

To run it, open `calculator/index.html` with the file downloaded in any browser.

### Core formulas
1 contract = 100 oz

| Quantity | Formula |
|---|---|
| Premium | (futures - spot) × contracts × 100 |
| Net premium | entry premium - exit premium - previous buyback cost |
| Base return | net premium / (exit spot × contracts × 100) |
| Annualized return | base return / days held × 360 |

**Example:** 18 contracts shorted on 15 Mar 2025 and rolled on 15 May 2025 earned a net premium of $62,496, about 6.4% annualized on the gold they hedged.

## Repo layout

```
rollover/      Python roll-timing scripts + output rankings
data/          Hourly futures and spot price files (GCZ4 to GCZ5)
spreadsheet/   Excel model + formula reference
calculator/    Web calculator (index.html, style.css, app.js)
```

## Requirements

The Python scripts need `pandas` and `openpyxl`:

```
pip install pandas openpyxl
python rollover/GJ5_Rollover.py
```

The calculator and spreadsheet need nothing beyond a browser and Excel.

## Conclusion

Across all five rolls from Nov 2024 to Jul 2025, the backtest points to the same pattern.

| Roll | Best hour (by spread ratio) | Days before first notice | Premium captured ($/oz) | Best vs worst hour ($/oz) |
|---|---|---|---|---|
| Z4→G5 | 26 Nov 2024, 22:00 | 3 | 24.80 | 1.50 |
| G5→J5 | 28 Jan 2025, 01:00 | 3 | 28.20 | 5.20 |
| J5→M5 | 25 Mar 2025, 23:00 | 6 | 28.90 | 4.80 |
| M5→Q5 | 26 May 2025, 23:00 | 4 | 28.70 | 1.90 |
| Q5→Z5 | 25 Jul 2025, 21:00 | 6 | 57.20 | 2.90 |

*Premium captured = the new contract's premium minus the expiring contract's premium at the moment of the roll.*

**1. Roll late, just before first notice.** In every roll, the best hour fell 3 - 6 days before the expiring contract's first notice day. By then the expiring contract's premium had nearly disappeared (as low as $0.04/oz), while the next contract still traded $25 - 58 above spot.

**2. Timing matters, but the curve matters more.** Choosing the best hour instead of the worst one added $1.50 - 5.20/oz, or $150 - 520 per contract. That counts for a large position, but most of the return comes from the premium itself, about $25 - 29/oz for a two-month roll. The Q→Z roll captured about twice as much ($57/oz) because December is four months past August, not two. The hedge earned roughly 6% a year, as in the example above.

**3. The spread ratio finds the right days, not always the best dollar outcome.** The ratio blows up when the expiring contract's premium is close to zero (718× on the M→Q roll), and it can rank a smaller dollar gain above a larger one. On the J→M roll, the top-ratio hour captured $28.90/oz, but three days later both premiums had widened, and the roll would have captured $31.70/oz. A better version would rank hours by premium captured in dollars and use the ratio as a second check. 

**Limitations.** The backtest covers one annual cycle (five rolls) and uses hourly opening prices, not actual bid/ask fills. It ignores trading fees and margin costs and covers only contango markets. It filters out hours when either contract traded below spot, so it says nothing about rolling in backwardation.
