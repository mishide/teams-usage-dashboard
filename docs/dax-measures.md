# DAX measures and calculated tables

These are the measures behind the three distribution charts. All data is synthetic.

## How the group is defined

Every measure uses `ALLEXCEPT` on the metrics table, keeping only `ROLE` and `DIVISION`. That means:

- **The group** is whatever the Role and Division filters select (all users if none are chosen).
- **The selected user** is removed from the group calculation, so the curve always describes the whole group, not the individual.

The `_INDIC` measures do the opposite: they keep `USER` as well, so they return the selected person's own average.

## Group mean and standard deviation

```dax
Mean_ScreenShr =
CALCULATE(
    AVERAGE( Teams_Metrics[ScreenShare_hrs] ),
    ALLEXCEPT( Teams_Metrics, Teams_Metrics[ROLE], Teams_Metrics[DIVISION] )
)

stdDev_ScreenShr =
CALCULATE(
    STDEV.P( Teams_Metrics[ScreenShare_hrs] ),
    ALLEXCEPT( Teams_Metrics, Teams_Metrics[ROLE], Teams_Metrics[DIVISION] )
)
```

The same pattern is repeated for meetings (`Mean_Meetings`, `stdDev_Meetings`, on `[Meeting Count]`) and calls (`Mean_CallCount`, `stdDev_CallCnt`, on `[Call Count]`).

## Curve boundaries (mean ± 3 standard deviations)

```dax
_B_X-3a_scrnShr  = [Mean_ScreenShr] - 3 * [stdDev_ScreenShr]
_B_X+3a+scrnShar = [Mean_ScreenShr] + 3 * [stdDev_ScreenShr]
```

Equivalent boundary measures exist for meetings and calls.

## X-axis tables

Each chart has a calculated table of whole-number X values from the lower to the upper boundary. `CALENDAR` is used as a convenient way to generate a sequence of integers.

```dax
Normal Distribution Scrn Shr =
VAR minvalue =
    CALCULATE( FLOOR( [_B_X-3a_scrnShr], 1 ),
        ALLEXCEPT( Teams_Metrics, Teams_Metrics[ROLE], Teams_Metrics[DIVISION] ) )
VAR maxvalue =
    CALCULATE( CEILING( [_B_X+3a+scrnShar], 1 ),
        ALLEXCEPT( Teams_Metrics, Teams_Metrics[ROLE], Teams_Metrics[DIVISION] ) )
RETURN
    SELECTCOLUMNS( CALENDAR( minvalue, maxvalue ), "X", INT( [Date] ) )
```

## Normal curve height

The y-value of each point is the normal probability density function:

```dax
f(x) scrn shr =
CALCULATE(
    EXP( -( VALUES( 'Normal Distribution Scrn Shr'[X] ) - Teams_Metrics[Mean_ScreenShr] ) ^ 2
         / ( 2 * Teams_Metrics[stdDev_ScreenShr] ^ 2 ) )
    / ( SQRT( 2 * PI() ) * Teams_Metrics[stdDev_ScreenShr] ),
    ALLEXCEPT( Teams_Metrics, Teams_Metrics[ROLE], Teams_Metrics[DIVISION] )
)
```

## Selected-user and role averages (the reference lines and cards)

```dax
Mean_ScrnShr_INDIC =
CALCULATE(
    AVERAGE( Teams_Metrics[ScreenShare_hrs] ),
    ALLEXCEPT( Teams_Metrics, Teams_Metrics[USER], Teams_Metrics[ROLE], Teams_Metrics[DIVISION] )
)

Mean_ScrnShr_ROLE =
CALCULATE(
    AVERAGE( Teams_Metrics[ScreenShare_hrs] ),
    ALLEXCEPT( Teams_Metrics, Teams_Metrics[ROLE], Teams_Metrics[DIVISION] )
)
```

## Above or below the group (status arrow)

```dax
Status_ScreenShare =
VAR X_status =
    CALCULATE( AVERAGE( Teams_Metrics[ScreenShare_hrs] ),
        ALLEXCEPT( Teams_Metrics, Teams_Metrics[USER], Teams_Metrics[ROLE], Teams_Metrics[DIVISION] ) )
VAR Group_mean =
    CALCULATE( AVERAGE( Teams_Metrics[ScreenShare_hrs] ),
        ALLEXCEPT( Teams_Metrics, Teams_Metrics[ROLE], Teams_Metrics[DIVISION] ) )
RETURN
    IF( X_status >= Group_mean, UNICHAR( 11165 ), UNICHAR( 11167 ) )
```

## Relationships

- `teamUserLookup[User]` (one) to `Teams_Metrics[USER]` (many), cross-filtering in both directions.
- Three date columns on `Teams_Metrics` each relate to an auto-generated date table.
