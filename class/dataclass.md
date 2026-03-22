## from dataclasses 

import dataclass

@dataclass
class SizingConfig:
    base_risk: float = 0.005       # 0.50% of equity
    min_risk: float = 0.0025       # 0.25%
    max_risk: float = 0.01         # 1.00%
    min_prob: float = 0.50         # minimum acceptable win probability
    max_drawdown: float = 0.10     # 10% max strategy drawdown
    regime_floor: float = 0.50     # minimum regime quality
    use_kelly_overlay: bool = False
    kelly_fraction: float = 0.25   # fractional Kelly if enabled


@dataclass
class TradeContext:
    equity: float
    stop_distance_price: float
    point_value: float             # dollar value per point/unit
    p_win: float                   # estimated win probability
    reward_risk: float             # expected reward/risk ratio, e.g. 1.5
    regime_quality: float          # 0 to 1.5
    state_confidence: float        # 0 to 1
    current_drawdown: float        # 0.0 to 1.0
    atr_baseline: float
    atr_current: float
    wins_so_far: int
    losses_so_far: int
    trade_index: int               # 1-based index within batch
    batch_size: int
    target_wins: int
    direction: str                 # "long" or "short"


@dataclass
class PositionSizeResult:
    allowed: bool
    risk_fraction: float
    risk_amount: float
    units: float
    contracts: int
    masaniello_factor: float
    quality_factor: float
    drawdown_factor: float
    volatility_factor: float
    expected_edge: float
    reason: str


class DynamicMasanielloSizer:
    def _init_(self, config: SizingConfig):
        self.config = config

    def _clamp(self, x: float, low: float, high: float) -> float:
        return max(low, min(high, x))

    def _expected_edge(self, p_win: float, reward_risk: float) -> float:
        """
        Expected edge in R terms:
        E = p*R - (1-p)
        """
        return p_win * reward_risk - (1.0 - p_win)

    def _masaniello_factor(self, wins_so_far: int, trade_index: int, batch_size: int, target_wins: int) -> float:
        trades_left = max(1, batch_size - trade_index + 1)
        wins_needed = max(0, target_wins - wins_so_far)
        raw = wins_needed / trades_left
        return self._clamp(raw, 0.0, 1.5)

    def _quality_factor(self, p_win: float, regime_quality: float, state_confidence: float) -> float:
        """
        Combines:
        - edge above threshold
        - regime quality
        - state confidence
        """
        prob_edge = max(0.0, p_win - self.config.min_prob)
        # Normalize prob edge: e.g. 0.10 edge above 0.50 becomes 1.0
        prob_factor = self._clamp(prob_edge / 0.10, 0.0, 1.5)

        regime_factor = self._clamp(regime_quality, 0.0, 1.5)
        confidence_factor = self._clamp(state_confidence, 0.0, 1.0)

        return prob_factor * regime_factor * confidence_factor

    def _drawdown_factor(self, current_drawdown: float) -> float:
        dd_ratio = current_drawdown / max(self.config.max_drawdown, 1e-9)
        return self._clamp(1.0 - dd_ratio, 0.25, 1.0)

    def _volatility_factor(self, atr_baseline: float, atr_current: float) -> float:
        if atr_current <= 0:
            return 1.0
        raw = atr_baseline / atr_current
        return self._clamp(raw, 0.5, 1.5)

    def _fractional_kelly(self, p_win: float, reward_risk: float) -> float:
        """
        Kelly for payoff b = reward_risk:
        f* = (b*p - q) / b
        """
        b = reward_risk
        q = 1.0 - p_win
        if b <= 0:
            return 0.0
        kelly = (b * p_win - q) / b
        return max(0.0, kelly) * self.config.kelly_fraction

    def size_trade(self, ctx: TradeContext) -> PositionSizeResult:
        if ctx.equity <= 0:
            return PositionSizeResult(
                allowed=False, risk_fraction=0.0, risk_amount=0.0, units=0.0, contracts=0,
                masaniello_factor=0.0, quality_factor=0.0, drawdown_factor=0.0,
                volatility_factor=0.0, expected_edge=0.0,
                reason="Equity must be positive."
            )

        if ctx.stop_distance_price <= 0 or ctx.point_value <= 0:
            return PositionSizeResult(
                allowed=False, risk_fraction=0.0, risk_amount=0.0, units=0.0, contracts=0,
                masaniello_factor=0.0, quality_factor=0.0, drawdown_factor=0.0,
                volatility_factor=0.0, expected_edge=0.0,
                reason="Stop distance and point value must be positive."
            )

        if ctx.regime_quality < self.config.regime_floor:
            return PositionSizeResult(
                allowed=False, risk_fraction=0.0, risk_amount=0.0, units=0.0, contracts=0,
                masaniello_factor=0.0, quality_factor=0.0, drawdown_factor=0.0,
                volatility_factor=0.0, expected_edge=0.0,
                reason="Regime quality below threshold."
            )

        if ctx.p_win < self.config.min_prob:
            return PositionSizeResult(
                allowed=False, risk_fraction=0.0, risk_amount=0.0, units=0.0, contracts=0,
                masaniello_factor=0.0, quality_factor=0.0, drawdown_factor=0.0,
                volatility_factor=0.0, expected_edge=0.0,
                reason="Win probability below threshold."
            )

        edge = self._expected_edge(ctx.p_win, ctx.reward_risk)
        if edge <= 0:
            return PositionSizeResult(
                allowed=False, risk_fraction=0.0, risk_amount=0.0, units=0.0, contracts=0,
                masaniello_factor=0.0, quality_factor=0.0, drawdown_factor=0.0,
                volatility_factor=0.0, expected_edge=edge,
                reason="Expected edge is non-positive."
            )

        masaniello_factor = self._masaniello_factor(
            wins_so_far=ctx.wins_so_far,
            trade_index=ctx.trade_index,
            batch_size=ctx.batch_size,
            target_wins=ctx.target_wins,
        )

        quality_factor = self._quality_factor(
            p_win=ctx.p_win,
            regime_quality=ctx.regime_quality,
            state_confidence=ctx.state_confidence,
        )

        drawdown_factor = self._drawdown_factor(ctx.current_drawdown)
        volatility_factor = self._volatility_factor(ctx.atr_baseline, ctx.atr_current)

        risk_fraction = (
            self.config.base_risk
            * (0.5 + masaniello_factor)
            * quality_factor
            * drawdown_factor
            * volatility_factor
        )

        if self.config.use_kelly_overlay:
            kelly_cap = self._fractional_kelly(ctx.p_win, ctx.reward_risk)
            risk_fraction = min(risk_fraction, kelly_cap if kelly_cap > 0 else self.config.min_risk)

        risk_fraction = self._clamp(risk_fraction, self.config.min_risk, self.config.max_risk)

        risk_amount = ctx.equity * risk_fraction

        dollars_at_risk_per_unit = ctx.stop_distance_price * ctx.point_value
        units = risk_amount / dollars_at_risk_per_unit
        contracts = int(units)

        if contracts < 1:
            return PositionSizeResult(
                allowed=False, risk_fraction=risk_fraction, risk_amount=risk_amount,
                units=units, contracts=0,
                masaniello_factor=masaniello_factor, quality_factor=quality_factor,
                drawdown_factor=drawdown_factor, volatility_factor=volatility_factor,
                expected_edge=edge,
                reason="Calculated size is smaller than 1 contract/unit."
            )

        return PositionSizeResult(
            allowed=True,
            risk_fraction=risk_fraction,
            risk_amount=risk_amount,
            units=units,
            contracts=contracts,
            masaniello_factor=masaniello_factor,
            quality_factor=quality_factor,
            drawdown_factor=drawdown_factor,
            volatility_factor=volatility_factor,
            expected_edge=edge,
            reason="Trade allowed."
        )

