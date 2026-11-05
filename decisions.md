# Decision Log: Olist Late-Delivery Project

Data: Olist Brazilian E-Commerce Public Dataset (Kaggle), CC BY-NC-SA 4.0. Credit: Olist.
Table: `olist_orders_dataset.csv`, 99,441 rows, one row per `order_id`.

## 1. Problem scope (draft, to be completed)

- Prediction moment: at the time of purchase
- Target: order arrives late (actual arrival date later than the promised date)
- Unit of analysis: one order
- Success metric: TODO (decide after the cost-of-errors discussion)
- Baseline: TODO

## 2. Which orders get a label

An order can only be labeled late or on time if both dates exist: the actual arrival (`order_delivered_customer_date`) and the promise (`order_estimated_delivery_date`).

| Status | Count | Action | Reason type | Note |
|---|---|---|---|---|
| delivered, with arrival date | 96,470 | Keep | Labelable | Both dates exist, so late or on time can be computed. |
| delivered, no arrival date | 8 | Exclude | Outcome not observed (contradictory row) | Status says delivered but no arrival is recorded, so the row can't be trusted or labeled. About 0.008% of delivered orders, so the impact is negligible. |
| shipped | 1,107 | Exclude | Outcome not observed | Left the seller (carrier date exists) but no arrival recorded. Reviews can't prove arrival, since surveys also go out when the promised date passes. |
| invoiced | 314 | Exclude | Outcome not observed | Billed but no carrier or arrival date. Earlier in the order's life, so the outcome is unknown. |
| processing | 301 | Exclude | Outcome not observed | No carrier or arrival date. Still at the seller stage when the data was captured, and we don't know how it ends. |
| approved | 2 | Exclude | Outcome not observed | No carrier or arrival date. Too few rows to affect the model. |
| created | 5 | Exclude | Outcome not observed | Same pattern as approved. |
| canceled | 625 | Exclude | Out of scope | 619 of 625 (about 99%) have no arrival date, and a canceled order never became a delivery. 6 have an arrival date; excluded anyway. |
| unavailable | 609 | Exclude | Out of scope | All 609 have no carrier or arrival date. All were approved, so the likely story is "approved, then failed later" (a hypothesis, not confirmed). 6 have item rows; cause unknown. |
| **Total** | **99,441** | | | Reconciles: 96,470 kept + 2,971 excluded |

## 3. Limitations

- **Excluding canceled and unavailable orders:** The model only learns from orders that reached delivery, so it never sees what failed orders look like. If failures are linked to certain sellers or product categories, the model's picture of the business is rosier than reality.
- **Excluding shipped, invoiced and processing orders:** These outcomes aren't observed yet, and we don't know how they end. If orders that stall are more likely to be late, the training data under-represents lateness.
- **Review-timing rescue for shipped orders:** A survey created before the promised date suggests the package arrived, but this can only recover on-time labels. Late shipped orders stay invisible, so rescuing would inflate the on-time share.
- **Delivery timestamps:** The documentation doesn't say how the arrival date was recorded (carrier scan, system update, or customer confirmation), so it may lag behind the real arrival.
- **Scope of the data:** Orders from 2016 to 2018 on one platform, so results describe that period and may not generalize.

## 4. Open items

- Define "late" in code (timestamp vs date, ties)
- Decide which features pass the purchase-time test
- Geolocation join: zip prefixes need zero-padding to 5 characters
