# Decision Log: Olist Late-Delivery Project

Data: Olist Brazilian E-Commerce Public Dataset (Kaggle), CC BY-NC-SA 4.0. Credit: Olist.
Table: `olist\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_orders\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_dataset.csv`, 99,441 rows, one row per `order\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_id`.

## 1\. Problem scope (draft, to be completed)

* Prediction moment: at the time of purchase
* Target: order arrives late (actual arrival date later than the promised date)
* Unit of analysis: one order
* Success metric: Best precision subject to a minimum recall X.

&#x09; - Action: when the model flags an order as likely late, the business upgrades it to expedited shipping at its own expense.

&#x20; 	 - Cost of a false positive: the premium shipping fee paid for an order that would have arrived on time anyway.

&#x20; 	 - Cost of a false negative: a late order that was not rescued (assumed to risk customer churn and lost trust; not yet tested).

&#x20; 	 - X is set with the business. Flagging more orders raises recall but lowers precision, so X decides how much wasted shipping the business accepts.

&#x20;	 - Accuracy is not used, because only 6.8% of orders are late.

&#x09;

.Baseline: "always predict on time".

&#x09;  - Recall 0%, precision undefined (it flags nothing).

&#x20; 	  - Accuracy 93.2%, recorded for reference only.

&#x20; 	  - A real model must catch late orders at the target recall with precision above the base rate of 6.8%, which is what random flagging would give.



## 2\. Which orders get a label

An order can only be labeled late or on time if both dates exist: the actual arrival (`order\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_delivered\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_customer\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_date`) and the promise (`order\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_estimated\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_delivery\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\_date`).

|Status|Count|Action|Reason type|Note|
|-|-|-|-|-|
|delivered, with arrival date|96,470|Keep|Labelable|Both dates exist, so late or on time can be computed.|
|delivered, no arrival date|8|Exclude|Outcome not observed (contradictory row)|Status says delivered but no arrival is recorded, so the row can't be trusted or labeled. About 0.008% of delivered orders, so the impact is negligible.|
|shipped|1,107|Exclude|Outcome not observed|Left the seller (carrier date exists) but no arrival recorded. Reviews can't prove arrival, since surveys also go out when the promised date passes.|
|invoiced|314|Exclude|Outcome not observed|Billed but no carrier or arrival date. Earlier in the order's life, so the outcome is unknown.|
|processing|301|Exclude|Outcome not observed|No carrier or arrival date. Still at the seller stage when the data was captured, and we don't know how it ends.|
|approved|2|Exclude|Outcome not observed|No carrier or arrival date. Too few rows to affect the model.|
|created|5|Exclude|Outcome not observed|Same pattern as approved.|
|canceled|625|Exclude|Out of scope|619 of 625 (about 99%) have no arrival date, and a canceled order never became a delivery. 6 have an arrival date; excluded anyway.|
|unavailable|609|Exclude|Out of scope|All 609 have no carrier or arrival date. All were approved, so the likely story is "approved, then failed later" (a hypothesis, not confirmed). 6 have item rows; cause unknown.|
|**Total**|**99,441**|||Reconciles: 96,470 kept + 2,971 excluded|

## 3\. Limitations

* **Excluding canceled and unavailable orders:** The model only learns from orders that reached delivery, so it never sees what failed orders look like. If failures are linked to certain sellers or product categories, the model's picture of the business is rosier than reality.
* **Excluding shipped, invoiced and processing orders:** These outcomes aren't observed yet, and we don't know how they end. If orders that stall are more likely to be late, the training data under-represents lateness.
* **Review-timing rescue for shipped orders:** A survey created before the promised date suggests the package arrived, but this can only recover on-time labels. Late shipped orders stay invisible, so rescuing would inflate the on-time share.
* **Delivery timestamps:** The documentation doesn't say how the arrival date was recorded (carrier scan, system update, or customer confirmation), so it may lag behind the real arrival.
* **Scope of the data:** Orders from 2016 to 2018 on one platform, so results describe that period and may not generalize.
* **Rates in small groups** (for example multi-seller orders) rest on very few late orders and are unreliable.
* **Associations, not causes:** The late-rate patterns above are associations. Features such as freight and price may stand in for distance, weight or product type.

## 4\. Open items

* n\_sellers vs n\_items overlap test (parked)
* Features from the products table (weight, dimensions, category) and distance via geolocation (zip prefixes need zero-padding to 5 characters)
* Split strategy (time-based)
* Baseline model, then real models





## 5\. Features

* \- promised\_days (estimated delivery date minus purchase date, in days): safe at purchase time. Weak on its own, with signal at the extremes (10.4% late under 10 days, 2.25% at 40+, against 6.8% overall). Kept as a candidate. Details: notebooks/01\_data\_exploration.ipynb.



|Feature|Source|Safe at purchase?|Late-rate finding|Status|
|-|-|-|-|-|
|promised\_days|derived (estimated delivery date minus purchase date)|Yes|10.4% late under 10 days, 2.25% at 40+ days, close to 6.8% in the middle|Weak candidate|
|n\_items|order\_items|Yes|6.9% for 1 item, about 5% for 2+ items|Weak candidate|
|n\_sellers|order\_items|Yes|About 1% for 2+ sellers, but only about 13 late orders|Weak candidate, may overlap with n\_items|
|total\_freight|order\_items|Yes|Rises steadily across five equal-sized bins, from about 4.4% (lowest fifth) to about 8.1% (highest fifth)|Candidate, strongest so far|
|total\_price|order\_items|Yes|Rises more weakly, from about 5.9% (lowest fifth) to about 7.5% (highest fifth)|Weak candidate|



Correlations (Spearman): total\_price and total\_freight 0.47, total\_freight and n\_items 0.38, total\_price and n\_items 0.18. No pair is close to a copy, so all are kept for now. Total\_freight mixes weight, package size and distance; the data cannot separate them yet.



Rule for every feature: check the late rate across its range against the 6.8% baseline, and check the group sizes before trusting a rate.

