# Masaniello betting strategy (sometimes spelled Masaniello system). 
It is a bankroll management and staking strategy originally developed in Italy for sports betting, but it is also sometimes used in trading and binary options.

The system is designed to maximize profit over a fixed number of bets while controlling risk, assuming you will win a predetermined number of bets out of a sequence.

## System Concept

1. Core Idea

Instead of risking the same amount each trade, the stake changes after every result.

You define three things:
	1.	Total number of bets/trades
	2.	Expected number of wins
	3.	Starting bankroll

The system calculates the optimal stake for each bet so that if you reach the expected number of wins, you hit a target profit.

Example structure:
	•	Bankroll: $1,000
	•	Total bets: 10
	•	Expected wins: 6
	•	Odds: 2.0

The system calculates the stake for each step depending on previous results.

⸻

2. How It Works

The Masaniello strategy uses probability progression rather than Martingale-style doubling.

Key characteristics:

✔ Stake adjusts after every win or loss
✔ You do not need to win every trade
✔ Risk is spread across a sequence

If you achieve the planned wins within the series, you reach the target profit before the sequence ends.

⸻

3. Example (Simplified)

Suppose:
	•	Bankroll: $1,000
	•	Trades: 10
	•	Expected wins: 6
	•	Odds: 2.0

The staking table might look like:

![Staking Table](/assets/modelimage-stakingtable.jpeg "Staking Table")

4. Why Some Traders Use It

Traders like Masaniello because:

* It allows losing trades within the sequence
* It prevents runaway Martingale risk
* It creates structured bankroll growth

It is often used in:
	•	Sports betting
	•	Binary options
	•	Short-term trading systems

5. The Hidden Assumption

The system only works if your win rate matches the planned probability.

Example:
	•	Plan = 6 wins out of 10 (60%)
	•	Actual = 4 wins

Then the sequence fails and bankroll drops.

So the strategy does not create edge — it only manages capital.

6. How Quants Adapt It for Trading

Algorithmic traders sometimes adapt Masaniello for:

* fixed trade batches (20–50 trades)
* dynamic position sizing
* probability-based bet sizing

It becomes similar to optimal bet sizing / Kelly variations.

For algorithmic trading, Masaniello becomes interesting when combined with:
	•	probability forecasts from a model
	•	Markov models or HMM signals
	•	volatility-adjusted position sizing

1) How quants adapt Masaniello for trading systems

Classic Masaniello assumes:
	•	a fixed number of trades
	•	a target number of wins
	•	known payout/odds
	•	stake adjustments after each result

In trading, quants usually adapt those assumptions like this:

A. Replace “bet odds” with reward-to-risk

Instead of bookmaker odds, use your trade’s expected payoff.

Example:
	•	stop loss = 1R
	•	take profit = 1.5R

Then your effective payoff is 1.5 instead of fixed sports odds.

B. Replace “expected wins” with model win probability

Instead of saying “I need 6 wins out of 10,” use your signal model.

Example:
	•	HMM says next-bar-up probability = 0.64
	•	breakout model confidence = 0.71
	•	regime filter says trending regime = high quality

That lets you size larger only when the signal quality is stronger.

C. Use trade batches

Quants often run Masaniello over a rolling block such as:
	•	10 trades
	•	20 trades
	•	30 trades

At the end of the block, they reset the staking schedule based on updated equity and updated edge.

D. Add volatility normalization

A fixed dollar stake is weak for trading because volatility changes.

So instead of:
	•	“risk $100 every time”

use:
	•	“risk x% of equity adjusted by ATR or stop distance”

That makes the sizing more realistic for markets.

E. Cap exposure

Real trading versions nearly always include:
	•	max risk per trade
	•	max daily drawdown
	•	max losing streak cutoff
	•	max correlated positions

Because even a good Masaniello sequence can become aggressive late in the run if results lag.

2) A formula to automate it inside your algorithm

Here is the clean way to think about it.

Define:
	•	B = current bankroll or equity
	•	N = total trades in the sequence
	•	W = target wins needed
	•	i = current trade number
	•	w = wins achieved so far
	•	l = losses achieved so far
	•	p = estimated probability of winning next trade
	•	R = reward-to-risk ratio on the trade
	•	f_i = fraction of bankroll to risk on this trade

A simple trading adaptation is:

f_i = \alpha \cdot \frac{(W - w)}{(N - i + 1)} \cdot \left(p - p_{min}\right) \cdot g(R)

Where:
	•	\alpha = aggressiveness factor
	•	\frac{(W-w)}{(N-i+1)} = required win density remaining
	•	p - p_{min} = edge above your minimum acceptable probability
	•	g(R) = payoff adjustment, such as R or \frac{R}{1+R}

Then convert that fraction to position size:

\text{Risk Amount} = B \cdot f_i

\text{Position Size} = \frac{\text{Risk Amount}}{\text{Stop Distance in \$}}

Practical bounded version

In live systems, use:

f_i = \min(f_{max}, \max(f_{min}, \alpha \cdot E_i \cdot D_i))

Where:
	•	E_i = p_i \cdot R - (1-p_i) is expected edge
	•	D_i = \frac{W-w}{N-i+1} is remaining target pressure
	•	f_{min} = floor risk, say 0.25%
	•	f_{max} = ceiling risk, say 1% or 1.5%

This is much safer than raw progression.

Example

Suppose:
	•	equity = $10,000
	•	batch = 10 trades
	•	target wins = 6
	•	current trade = 4
	•	wins so far = 2
	•	model probability = 0.63
	•	reward/risk = 1.4
	•	alpha = 0.8

Then:

E_i = 0.63 \cdot 1.4 - (1-0.63) = 0.882 - 0.37 = 0.512

D_i = \frac{6-2}{10-4+1} = \frac{4}{7} \approx 0.571

f_i = 0.8 \cdot 0.512 \cdot 0.571 \approx 0.234

That raw number is too large as a bankroll fraction, so in practice you cap it:
	•	f_{max} = 0.01

So risk becomes:
	•	$10,000 × 1% = $100

That is the important quant lesson: use Masaniello logic as a signal-aware scaler, not as an uncapped betting progression.

3) A better version: Dynamic Masaniello

This is the version that makes more sense for algorithmic trading and prop-style discipline.

Instead of using a rigid table, Dynamic Masaniello updates in real time based on:
	•	current equity
	•	actual win rate versus expected win rate
	•	current regime
	•	signal confidence
	•	drawdown state

Core idea

Each new trade recalculates the required pace to hit the batch target, but also respects live conditions.

A useful structure is:

f_i = \beta \cdot M_i \cdot Q_i \cdot DD_i

Where:
	•	M_i = Masaniello progress factor
	•	Q_i = model quality factor
	•	DD_i = drawdown adjustment factor
	•	\beta = base risk multiplier

Components

A. Masaniello progress factor
This reflects where you are in the sequence.

M_i = \frac{W-w}{N-i+1}

If you are behind schedule, it rises.
If you are ahead, it falls.

B. Model quality factor
This uses your signal confidence.

Example:

Q_i = \max(0, \frac{p_i - 0.5}{0.2})

So:
	•	0.55 probability = small boost
	•	0.70 probability = strong boost
	•	below 0.50 = no trade or minimum size

You can also include regime quality:

Q_i = q_{signal} \cdot q_{regime}

For example:
	•	trending regime = 1.2
	•	choppy regime = 0.7

C. Drawdown adjustment
This is where Dynamic Masaniello beats many betting systems.

Example:

DD_i = \max(0.3,\ 1 - \frac{\text{Current Drawdown}}{\text{Max Allowed Drawdown}})

If drawdown increases, risk shrinks automatically.

So even if the sequence says “bet bigger,” the drawdown governor says “not so fast.”

A robust live-trading version

Here is a very practical formula:

f_i = \min(f_{max},\ \max(f_{min},\ \beta \cdot M_i \cdot Q_i \cdot DD_i \cdot V_i))

Where:
	•	V_i = volatility adjustment factor
	•	for example V_i = \frac{ATR_{baseline}}{ATR_{current}}

This reduces size when volatility expands too much.

Suggested settings for a quant system

For a cautious version:
	•	batch length: 10 to 20 trades
	•	target wins: based on actual backtested win rate minus a safety buffer
	•	base risk: 0.25% to 0.50%
	•	max risk: 1.00%
	•	stop trading batch if:
	•	daily drawdown breached
	•	4 to 5 losses in a row
	•	regime filter turns unfavorable

For an HMM or Markov model:
	•	use state probability as Q_i
	•	only activate Dynamic Masaniello in favorable hidden states
	•	revert to minimum size in uncertain states

Best way to use it with your HMM work

Because you are already working with probabilistic models, the cleanest setup is:

Entry engine

Your HMM / Markov / breakout model decides:
	•	long or short
	•	estimated win probability
	•	regime label

Sizing engine

Dynamic Masaniello decides:
	•	how much to risk based on
	•	probability
	•	batch progress
	•	drawdown
	•	volatility

Risk engine

Hard constraints override everything:
	•	max daily drawdown
	•	max trade risk
	•	max open exposure
	•	cooldown after consecutive losses

That separation is much better than embedding money management directly into the signal logic.

Below is a practical framework you can actually build around.

## HMM/Markov strategy breakdown

1. What the model is doing

Your HMM/Markov model is trying to answer:
	•	What market state are we likely in right now?
	•	Given that state, what is the probability of the next bar moving up or down?
	•	How confident should we be in taking the trade?

A useful distinction:
	•	Markov chain: models transitions between observable states
	•	Hidden Markov Model (HMM): assumes the true market regime is hidden, and you infer it from observable features

In trading, HMM is usually stronger because the real regime is not directly visible.

2. The core idea for trading

Instead of treating every bar the same, you assume price action behaves differently in different hidden regimes, such as:
	•	trending bullish
	•	trending bearish
	•	ranging / mean-reverting
	•	high-volatility chaotic
	•	low-volatility compression

The HMM estimates which regime is most likely now, then your trading logic uses that regime to decide whether to:
	•	go long
	•	go short
	•	stand aside
	•	reduce size
	•	increase size

3. Strategy architecture

A solid HMM/Markov trading system has 4 layers:

Layer 1: Feature extraction

You convert raw candles into model inputs.

Examples:
	•	return over 1 bar
	•	return over 3 bars
	•	ATR or normalized ATR
	•	rolling volatility
	•	candle body size
	•	HH/HL/LH/LL structure
	•	distance from moving average
	•	breakout above prior high / below prior low
	•	volume spike
	•	momentum slope

For your direction model, HHLL structure is very relevant.

Examples of HHLL features:
	•	is_higher_high
	•	is_higher_low
	•	is_lower_high
	•	is_lower_low
	•	count of HH in last N bars
	•	count of LL in last N bars
	•	net structure score = HH_count - LL_count

4. Hidden states

The HMM tries to learn states like:
	•	State 0 = quiet range
	•	State 1 = bullish trend
	•	State 2 = bearish trend
	•	State 3 = volatile transition

You do not hardcode the meaning initially. The model discovers the states, then you interpret them from the data.

For each state, you inspect:
	•	average forward return
	•	average volatility
	•	win rate for long trades
	•	win rate for short trades
	•	average duration in state

Then you label the states in trading terms.

5. Transition matrix

This is the Markov piece.

The model estimates probabilities like:

P(S_{t+1}=j \mid S_t=i)

![Markov Probabilities](/assets/modelimage-markovprobabilities.jpeg "Markov Probabilities")

6. Emission model

In an HMM, each hidden state emits observable patterns.

For example:
	•	bullish trend state may emit positive returns, repeated HH/HL, moderate ATR
	•	bearish trend state may emit negative returns, LH/LL, expanding ATR
	•	range state may emit small returns, mixed HH/LL, lower ATR

This is how the model links hidden state to the features you observe.

7. Trading logic

The model itself does not create a full strategy. It provides probabilities and regime context.

A practical workflow is:

Step A: Infer regime

For the current bar, estimate:
	•	most likely hidden state
	•	probability of each hidden state

Example:
	•	bull trend = 0.68
	•	range = 0.20
	•	bear trend = 0.12

Step B: Predict next-bar direction

Then estimate the probability of:
	•	next bar up
	•	next bar down

This can come from:
	•	state-conditioned historical transition stats
	•	a secondary classifier
	•	a Markov transition table of HHLL states
	•	forward-return frequencies by state

Example:
	•	P(up next bar | current state, features) = 0.64

Step C: Apply entry rule

Only trade if probability exceeds threshold.
Example:
	•	go long if p_up >= 0.60
	•	go short if p_down >= 0.60
	•	no trade otherwise

Step D: Apply regime filter

Even with a directional signal, block trades in poor conditions.

Example:
	•	only allow breakout longs in bullish or expansion states
	•	suppress breakouts in range state
	•	suppress all trades if uncertainty is high

Step E: Pass confidence to sizing engine

This is where your Dynamic Masaniello module comes in.

8. Simple HMM/Markov strategy example

Here is a clean design.

Inputs
	•	OHLCV bars
	•	lookback = 20
	•	HMM states = 4
	•	HHLL window = 10

Features
	•	1-bar return
	•	3-bar return
	•	ATR normalized by price
	•	rolling volatility
	•	HHLL structure score
	•	distance from 20 EMA

Regime model
	•	fit HMM on features
	•	infer current hidden state probabilities

Direction model

Compute next-bar-up probability using:
	•	current hidden state
	•	state transition tendency
	•	HHLL score
	•	short-term momentum

Example formula:

![Markov Formula](/assets/modelimage-markovformula.jpeg "Markov Formula")


Then clamp to [0,1].

Entry
	•	Long if p_up > 0.60 and bullish state probability is dominant
	•	Short if p_down > 0.60 and bearish state probability is dominant

Exit
	•	fixed stop based on ATR
	•	profit target based on R multiple
	•	optional regime exit if state flips strongly against the trade

Position size
	•	computed by the position-sizing module below

9. How HHLL fits your Markov idea

For your HHLL-based concept, you can build a simpler observable-state Markov model first before full HMM.

Observable state examples

Define current structure state from recent bars:
	•	HH_HL
	•	HH_LL
	•	LH_HL
	•	LH_LL
	•	INSIDE
	•	OUTSIDE

Then estimate transitions like:

P(\text{next bar up} \mid \text{current HHLL pattern})

Now here is an example:

Pattern
Next Up Prob
HH_HL 0.67
HH_LL 0.52
LH_LL 0.31
LH_HL 0.49

That gives you a direct Markov layer.

Then later, the HMM can sit above that and say which broader regime you are in.

So a very good progression is:
	1.	build HHLL Markov model
	2.	validate predictive power
	3.	add HMM regime filter
	4.	add probability-weighted sizing

That is the most practical path.

10. Best signals to pass into position sizing

Your sizing engine should receive these from the model:
	•	p_win: probability trade wins
	•	regime_quality: how favorable current state is
	•	state_confidence: certainty of state classification
	•	reward_risk: expected R multiple
	•	volatility_factor: ATR baseline / current ATR
	•	trade_allowed: true/false
	•	direction: long/short

These are enough to build a serious sizing system.

Python position-sizing module

Below is a practical module you can plug into a research or live system.

It uses:
	•	fixed-fractional base risk
	•	Dynamic Masaniello pressure factor
	•	model confidence
	•	reward/risk
	•	drawdown adjustment
	•	volatility normalization
	•	hard caps


## Reference
- `https://github.com/Noble-Trading-App/documentation/blob/main/class/dataclass.md` - dataclass

## What each part is doing

### base_risk

Your default position risk.
Example: 0.005 = 0.5% of equity.

### masaniello_factor

Increases or decreases pressure depending on how far behind or ahead you are in the trade batch.

If you still need many wins with few trades left, the factor rises.

### quality_factor

Combines:
	•	how far probability is above threshold
	•	how favorable the regime is
	•	how confident the state classification is

This is the most important model-driven component.

### drawdown_factor

Shrinks risk when the system is under stress.

### volatility_factor

Reduces size if current ATR is higher than your normal baseline.

### expected_edge

This checks whether the trade is mathematically favorable:

E = p \cdot R - (1-p)

If this is negative, the trade is blocked.

How to connect this to your HMM/Markov model

## Strategy Engine
signal = {
    "direction": "long",
    "p_win": 0.63,
    "reward_risk": 1.5,
    "regime_quality": 1.1,
    "state_confidence": 0.78,
    "trade_allowed": True,
}

Then your execution layer converts stop distance and account size into a final position size using the module.

That separation is ideal:
	•	model decides whether the setup is good
	•	sizer decides how much to risk
	•	executor places orders
	•	risk manager can still override everything

Example of a chart with signal for the hybrid risk management model:

![Example Chart](/assets/modelimage-examplechart.jpeg "Example Chart")






