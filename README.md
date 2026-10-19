# Kit End Date Prediction

## Summary

For every order in `inference.parquet` I predicted the date the kit phase will finish (`kit_end`), so that planners can schedule production and give customers realistic dates. I predicted the number of days between `kit_start` and `kit_end` and added it to `kit_start`.

My model is a linear regression. I compared it with a simple rule that needs no machine learning: the company's own expected kit time (`expected_kit_minutes`) turned into days. The model did better than the rule in all five test periods I tried. On average the error went down from 0.78 days to 0.69 days, which is about 12% better.

The three main things I found:
- The weekday a kit starts is the biggest factor. Kits that start on a Saturday take about a day longer.
- The company's expected kit times are not much use on their own. They are no better than guessing the same number for every order.
- A few kits take 5 days or more. They use a lot of kit time and the data cannot explain them.

## How I approached the problem

I treated it as a regression problem, because the answer is a number of days. I measured the error as MAE, which is the average number of days my prediction is wrong by.

I did not split the data randomly. The inference orders start after the training orders, so I tested the model on the last five weeks of the training data (from 2025-12-27 to 2026-01-31) and trained it on everything before. A random split would let the model learn from orders that happen after the ones it is tested on, which is not possible in real use. For the final predictions I trained the model again on all the cleaned training data. When testing, I also left out the training orders that had not finished before the test period started, so the model does not use information from the future (about 1,200 orders, 0.7% of the data).

I went through the five files one by one (read, check, clean, explore), then merged them into one table and built the model.

## Problems in the data and what I did

- Site: the orders are for site KHO but the calendar is for site FCJ. As per email communication I treated them as the same site.
- Capacity file: it has 191,934 rows but only 317 different records, one for each date. I checked that on every date the number of rows is the same as the number of orders that started that day, so the table was copied once for every order. I kept one row per date and dropped 12 rows with no date.
- Wrong targets: 734 training rows (0.4%) have `kit_end` before `kit_start`, which is not possible. The bad values bunch up around -11, -12 and -2 days, so it looks like a systematic entry problem. I left these rows out of training and testing and saved them in a separate file.
- Placeholder dates: `est_ship_date` is 1900-01-01 in 28 training rows and 1 inference row. I changed it to a missing value and kept the orders.
- Expected minutes file: one row has no family name, 31 rows have no `expected_burn_minutes`, and the minutes were stored as text. I removed the row with no name and changed the minutes to numbers. I left the missing burn minutes alone because burn time is not part of the kit phase.
- Families with no expected minutes: 24.5% of training rows and 19.8% of inference rows belong to a family that is not in the expected minutes file. Fixing spaces and capital letters did not help. These orders take about the same time as the others (1.44 days against 1.37), so I kept them and added a column `minutes_missing`.
- Other small things: `quantity_produced` and a few other columns were stored as `Decimal` so I changed them to numbers. `site` and `alt_type_desc` have the same value in every row so I ignored them.

After every merge I checked that the number of rows did not change, and I did not drop any inference row.

## What I found in the data

Most kits are quick. The middle value is 1 day and about 68% of kits finish within a day. Only about 2% take 5 days or more.

The start weekday matters most. About one in five kits starts on a Saturday. Kits that start from Tuesday to Thursday take about 1.1 days on average, while kits that start on a Saturday take about 2.1 days.

To understand the Saturday effect I looked at the day the kits end. About two thirds of the Saturday kits end on Monday or Tuesday and only about a quarter end during the weekend. So I think the extra day is mostly the weekend gap and not slower work. This is my reading of the pattern and I cannot prove it from the data.

The calendar flag `is_weekend` marks Saturday and Sunday, but about 20% of kits start on those days, so the flag does not show when the factory is closed. I used the real start weekday instead.

Capacity follows the weekday too. Saturday and Sunday have about 10.5 regular hours and 7.5 overtime hours, while Tuesday to Friday have about 15 to 16 regular hours and 5.3 overtime hours.

The other columns (quantity, order amount, waiting time before the start, `cfs_flag`) have little or no relation with the duration. Software customisation (`is_cfi`) and a few warehouses (for example KL3) take a little longer, but adding them to the model did not make it better, so I left them out.

The expected kit minutes are only weakly related to the real number of days (correlation 0.13).

The level also changes over time. The first weeks of data (March 2025) were slow, the middle of the year was about 1.0 to 1.2 days, and November to January was about 1.5 to 1.7 days. The share of kits taking 5 days or more rose to about 3% in the last two months.

The inference orders are different from the training orders in one way: only about 7% of them start on a Saturday (about 20% in training) and about 4% start on a Sunday (less than 1% in training).

## Model and results

The baseline is `expected_kit_minutes` divided by 1440 and rounded to whole days. For families with no expected minutes it uses the median duration.

The model is a linear regression. Its inputs are the start weekday, `regular_hours` and `ot_hours` on the start date, `expected_kit_minutes` (with the `minutes_missing` flag) and the order quantity. The predictions are rounded to whole days because the real duration is always whole days.

I used linear regression as a first attempt because it is simple and I can explain every result. It is a starting point and not the final answer.

On the five-week test period the average error was 0.737 days for the model and 0.814 days for the baseline. Always guessing 1 day gave 0.853.

One test period can be lucky, so I repeated the test on five separate five-week periods. Each time the model only learned from the data before the period. The table shows the average error in days.

| Test period starts | Always same number | Baseline | Weekday only | Final model |
|---|---|---|---|---|
| Aug 9 | 0.602 | 0.639 | 0.510 | 0.548 |
| Sep 13 | 0.590 | 0.629 | 0.508 | 0.533 |
| Oct 18 | 0.799 | 0.799 | 0.712 | 0.721 |
| Nov 22 | 1.029 | 1.017 | 0.938 | 0.897 |
| Dec 27 | 0.853 | 0.814 | 0.749 | 0.737 |
| Average | 0.775 | 0.779 | 0.683 | 0.687 |

The final model was better than the baseline in all five periods (average 0.687 against 0.779, about 12% better). A model with only the weekday does just as well (0.683), so the weekday is doing nearly all the work. Adding capacity, expected minutes and quantity does not improve the average. I kept the final model, but the difference between the two is too small to matter.

The model is not better everywhere. On Thursday and Friday starts the baseline is slightly better. It also does badly on the very long kits: kits that take 5 days or more are wrong by about 5 days, and kits that finish on the same day are predicted about a day too long.

## What this means for the business

On the five-week test period there were about 3,450 orders per week.

The model was within one day of the real end date for 89% of the orders (84% for the baseline). The orders that finished 2 or more days later than predicted went down from 14% to 9%, which is about 170 fewer orders per week. These are the orders that upset the schedule.

If planners promise the predicted date plus one day, 91% of the orders are on time. If they promise only the predicted date, about 75% are on time.

About 650 orders per week started on a Saturday in the test period and took 2.3 days on average, against 1.2 days for orders that started from Tuesday to Thursday. If the business wants these faster, it can work on Sundays for urgent orders or start urgent orders earlier in the week. This is a decision about capacity and not about prediction. Fewer inference orders start on a Saturday, so the effect will be smaller there.

Kits that take 5 days or more were about 3% of the orders (about 115 per week) but used about 15% of all the kit days, and their share is going up. The data does not say why they are delayed, so I would ask the business to record the reason (for example a parts shortage or rework).

I would also suggest updating the expected kit times, because on their own they are no better than a constant guess and a quarter of the order lines have no entry.

## Limits and things to be careful about

- Capacity for future dates may be a plan and not what really happened. If it is what really happened, the model would be using the future. The weekday-only model does not use capacity and does just as well (0.683 against 0.687), so the conclusions do not depend on it.
- Sunday starts are rare in the training data (about 1,300 rows) but are about 4% of the inference orders, so the predictions for them are less reliable.
- About 80% of the predictions for the inference orders are 1 day. This is normal for a model that tries to keep the average error small, and the model cannot predict the rare long kits. The average predicted duration (1.21 days) is lower than in the history (1.39 days) because fewer inference orders start on a Saturday.
- The model only predicts orders that already have a `kit_start`. It does not predict when an order will start.
- Kit times have been higher in the last months. If this continues the model will predict too low, so it should be trained again regularly.

## What I would do next

This is my first attempt, so I kept the model simple on purpose. I would like to extend it and try other regression models, for example random forest or gradient boosting, and compare them with the linear regression on the same five test periods. 
I am not sure they will do better, because the weekday already explains most of what can be predicted, but it is worth testing.

## Files

- `README.md': this summary.
- `Amarendar_DS_Case_Notebook_2026.ipynb`: the notebook with the code, plots and explanations.
- `Amarendar_DS_Case_kit_end_predictions_2026.npy`: the predicted kit end dates, 12,069 values in the same order as `inference.parquet`.
- `Amarendar_DS_Case_days_predictions_2026.npy`: the predicted number of days, in the same order.
- `Amarendar_DS_Case_inference_predictions_2026.csv`: the same predictions with the order id, family and kit start, easy to read.

To run the notebook, extract the data folder, change `DATA_DIR` in the setup cell to that folder, and install `pandas`, `pyarrow`, `numpy`, `matplotlib` and `scikit-learn`. The predictions can be loaded with `np.load("predictions_kit_end.npy")`.
