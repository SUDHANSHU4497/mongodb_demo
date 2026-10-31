# SENTINEL on MongoDB Atlas

MongoDB assignment, steps 1 to 6, using the SENTINEL fraud data.

## Files
- `sentinel_mongo.ipynb`: the notebook (all steps, with outputs from a test run)
- `sentinel_mongo.py`: the same code as a script
- `requirements.txt`, `.env.example`

## How to run
1. `pip install -r requirements.txt`
2. Add your IP in Atlas > Network Access.
3. Set your connection string, either in PowerShell:
   `$env:MONGODB_URI="mongodb+srv://<user>:<password>@<cluster>.mongodb.net/?retryWrites=true&w=majority"`
   or by copying `.env.example` to `.env` and pasting it there.
4. Check `ZIP_PATH` in Step 3 points to your `Sentinel project data.zip` (default: `C:\Users\sudha\Downloads\Sentinel project data.zip`).
5. Open the notebook and click Run All (or `python sentinel_mongo.py`).

Each run drops and reloads the `customers` and `transactions` collections in the `sentinel` database.

## Step 2: embed or reference?

**customers** (devices embedded):
```json
{"_id": "C_6f4c01a19463073c", "customer_age": 76, "home_city": "Mumbai", "region": "west",
 "devices": [{"device_id": "D_9778c4c36d78e39f", "device_kind": "mobile"},
             {"device_id": "D_7e5b49d297eb0791", "device_kind": "browser"}]}
```

**transactions** (reference the customer by `customer_id`):
```json
{"_id": "TX_0000100001", "customer_id": "C_478e0c3e1339c21e",
 "transaction_time": "2024-12-31T18:32:02.983Z", "amount": 1323.71, "payment_channel": "UPI",
 "merchant": {"merchant_id": "M_f041b8f9a5ea2888", "merchant_category": "electronics",
              "location": {"city": "Delhi", "lat": 28.61, "lon": 77.21}},
 "device": {"device_id": "D_e37a0af139efb59a", "device_kind": "mobile"},
 "fraud_label": 0}
```

**Why devices are embedded:** a customer has only a few devices (about 2 on average in SENTINEL: around 53,000 devices for 25,000 customers), so the list stays small. Devices are always looked at together with the customer (for example, "is this a new device for this customer?"), so one read returns everything with no join.

**Why transactions are referenced:** transactions keep growing (1.1 million in SENTINEL, about 44 per customer), so embedding them would make customer documents grow without limit (MongoDB caps a document at 16 MB). They are also queried on their own (by time, by category, across customers), so they get their own collection and point to the customer with `customer_id`.

Merchant and device details are copied into each transaction, so the reports in step 6 work without a join.

## Results from the test run
- **Step 3:** the code reads the CSV files straight out of the zip (no unzipping): `transactions/train` part 1 and 2 (10,000 rows), plus `labels` (fraud label), `merchants` (name), `devices` (device kind) and `customers`. It joins them with pandas, builds the documents and loads 10,000 transactions and 4,881 customers with `insert_many`. Change `NUM_PARTS` to load more (each part is 5,000 rows).
- **Step 4:** `insert_one` (a demo customer), `insert_many` (3 demo transactions), `find` with filter + projection (electronics payments over 40,000, 4 fields shown), `update_many` (41 payments of 50,000+ flagged `HIGH_VALUE`), `delete_one` (one demo transaction removed).
- **Step 5:** query = one customer's transactions sorted by time.
  - Before indexes: `COLLSCAN`, 10,002 documents examined.
  - After indexes on `customer_id` and `transaction_time`: `IXSCAN` on `customer_id_1`, 16 documents examined.
- **Step 6a, fraud rate per merchant_category:** gift_cards 2.62%, gaming 2.05%, electronics 1.13%, education 0.16%, all others 0%.
- **Step 6b, top 10 customers by amount in their last 24 hours of activity:** for each customer, take their latest transaction and add up what they spent in the 24 hours before it (the data is historical, so "24 hours before now" would be empty). The top customer spent 55,15,832.54 in 2 transactions.
