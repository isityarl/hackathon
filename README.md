# Customer RFM Analysis Project

## Overview

This project conducts an RFM (Recency, Frequency, Monetary) analysis on customer data derived from multiple sources. The primary goal is to segment customers.


## Data Sources

1.  **`accounts`**: Contains account information.
    * Expected columns (in order): `login`, `client_id`, `age_segment`, `sex_type`, `canal`, `client_type`.
2.  **`balance.csv`**: Contains client balance information.
    * Expected columns: `client_id`, `balance`.
3.  **`comissions.csv`** (Note: filename in notebook is `comissions.csv`): Contains commission data per login.
    * Expected columns: `login`, `date_trunc` (date of commission), `comission` (commission amount).
4.  **`exchange.csv`**: Contains information about last conversion dates.
    * Expected columns: `client_id`, `date_last_conv`.
5.  **`inouts.csv`**: Contains information about last ins/outs dates and counts.
    * Expected columns: `client_id`, `date_last_inouts`, `cnt_inouts`.
6.  **`ins.csv`**: Contains sum of "ins" per client.
    * Expected columns: `client_id`, `sum_ins`.
7.  **`trades.csv`**: Contains information about last trade activity dates and counts.
    * Expected columns: `login`, `date_last_activity`, `cnt_trades`.

**Data Integrity Note:**
* The notebook handles cases where `client_id` might be missing in the merged `data` DataFrame by generating a random Base64 string. This suggests that some logins might not have an associated `client_id` in the `accounts` data or that the merge logic results in NaNs for `client_id` for some commission records.

## RFM

5.  **Calculation - Frequency (Weighted):**
    * Creates a `data1` DataFrame by grouping `data` by `login` and `year_month` (derived from `date_trunc`) to get a monthly commission count (`count`).
    * Merges `cnt_trades` and `cnt_inouts` (from respective original tables, after merging `inouts` with `acc` to get `login`) into `data1`.
    * Fills NaN values for `cnt_trades` and `cnt_inouts` with 0.
    * Calculates a `ratio` of monthly commissions to total commissions for each login.
    * Distributes `cnt_trades` and `cnt_inouts` across months based on this `ratio` to get monthly `trades` and `inouts`.
    * Calculates a raw monthly `frequency` as `count` (commissions) + `trades` + `inouts`.
    * Applies a recency weight:
        * Sorts data by `year_month`.
        * Assigns a `month_rank` (0 for the most recent month, 1 for the next, etc.).
        * Calculates a `weight` based on the rank (1 - 0.025 \* `month_rank`), clipped at a minimum of 0.
        * Calculates the final weighted `Frequency` for RFM as `frequency * weight`.
6.  **Calculation - Recency:**
    * Determines the `last_activity` date for each client by taking the maximum of `date_last_activity`, `date_last_inouts`, `date_last_conv`, and `date_trunc` from the main `data` DataFrame.
    * Calculates a `snapshot_date` as the overall maximum `last_activity` date in the dataset.
    * Calculates `Recency` for each unique login in `data1` as the number of days between the `snapshot_date` and their `last_activity` date (after merging `client_id` to `data1` and then `last_activity` via `client_id`).
7.  **Calculation - Monetary:**
    * Creates a `forR` DataFrame by grouping the main `data` DataFrame by `login` and `year_month` to sum `comission`, `sum_ins`, and `balance`.
    * Applies a conditional logic:
        * If both `sum_ins` and `balance` are positive, `sum_ins` is replaced by `comission`, and `balance` is scaled by `comission / sum_ins`.
        * Otherwise, `sum_ins` and `balance` are set to 0 for that month/login.
    * Calculates monthly `Monetary` as `comission + sum_ins + balance`.
    * This monthly `Monetary` value is then merged back into `data1`.

