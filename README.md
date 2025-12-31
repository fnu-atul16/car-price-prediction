# 🏎️ Car Price Regression (Work in Progress)

This project focuses on **data cleaning, feature extraction, and regression model building** to predict car prices.  
The dataset contains multiple vehicle attributes such as name, mileage, drivetrain, fuel type, engine, price, transmission, etc.

---

---

## 🧹 Step 1 — Data Cleaning

### ✔️ 1. Handling Missing Values
- Checked for missing values across columns
- Dropped rows where **all column values were missing**

---

## 🚗 2. Cleaning `name` column (hardest part)
The `name` column was very messy and contained multiple mixed attributes.  
So I broke it down into multiple columns:

### 📍 Extracting Car Year
- Created a new column: **`car_year`**
- Used **regex** to extract year from the `name` column
- Removed the extracted year from `name` using `.replace()`

### 📍 Extracting Car Brand
- Created a new column: **`car_brand`**
- Split `name` and extracted the **first token** (brand)
- Removed brand from name and converted to **UPPER CASE**

---

## 🚘 3. Model / Trim / Badge Extraction

### 🔍 Observations
- After exploding values and applying `.value_counts()`, I noticed repeated keywords
- Categorized high-frequency values into:
  - **Model**
  - **Trim**
  - **Badge**
  - **Unknown**

### 🧠 Model Extraction Logic
- Built a dictionary of model keywords
- Created a function `extract_model_and_engine(dict_)`
- If model found → assign it
- Else → assign `"UNKNOWN"`

However, ~90% of values were UNKNOWN initially → so I customized further.

### ⚙️ Improved Model Logic
- If `"UNKNOWN"` and first token looks like a model → use it
- Otherwise remove that token from the `name` column

---

## 🛠️ 4. Trim & Badge Columns
- Created keyword lists for **trim** and **badge**
- Expanded both lists after observing random samples
- Used vectorized `.str.contains()` to extract values
- Non-matching values → `"UNKNOWN"`
- Removed extracted words from `name` to make it cleaner

⚠️ Still many **unknown** values left → **Work In Progress**

---

##⛽ 5. Cleaning `MPG` Column
- MPG values are often **ranges** (e.g. `25–32 MPG`)
- Plan: Take the **average** for regression
- Some rows contained words like `"Automatic"` which clearly belong to the **Transmission** column → shifted appropriately
- Used `regex + .str.contains()` for detection and separation

---

## 💲 6. Cleaning `price` Column
The price column has multiple formats:

| Example Value                | Issue |
|------------------------------|-------|
| `$20,000`                    | Standard ✔️ |
| `Not Priced`                 | Missing ❌ |
| `$25,000 Price Drop MSRP`    | Mixed information ❌ |

### ✨ Solution
Created two new columns:

| New Column       | Description                         |
|------------------|-------------------------------------|
| `Price`          | Cleaned final numeric price        |
| `Reduced_price`     | Any drop/reduced price information |

### 🔧 Steps
1. Split the `price` column into first and second tokens → store into `Price` & `Reduced_price`
2. Extract only numeric values using regex:
   ```python
   x = df["price"].str.strip().str.split().str[0].str.contains(r"^\$\d+,\d+$", regex=True, na=False)
   df["Price"] = pd.NA
   df.loc[x,"Price"] = (
       df.loc[x,"price"].str.split().str[0]
       .str.split("$").str[1]
       .str.replace(",", "")
       .astype("Int64")
   )
   ## 🚙 Data Cleaning: Filling Missing Prices (Group Median Imputation)

To avoid losing valuable data due to missing values in the `Actual_price` column, a **group-based median filling strategy** was implemented. This approach ensures that price estimations remain realistic and consistent with the vehicle's category, making the dataset more suitable for machine learning.

### 1️⃣ Imputation Strategy
The logic uses a hierarchical approach to fill missing values, moving from specific to general categories:

* **Step 1 (Brand-based):** Missing prices are first filled using the median price of the same **car brand**.
* **Step 2 (Model-based Fallback):** As a secondary check, any remaining missing values are filled using the median price of the same **car model**.
* **Step 3 (Final Cleanup):** Rows that still contain `NaN` values (where neither brand nor model median data is available) are dropped to maintain data integrity.

### 2️⃣ Python Implementation

```python
# 1️⃣ Fill missing price using median of the same car brand
df["Actual_price"] = df["Actual_price"].fillna(
    df.groupby("car_brand")["Actual_price"].transform("median")
)

# 2️⃣ Fallback: fill using median price of the same model
df["Actual_price"] = df["Actual_price"].fillna(
    df.groupby("model")["Actual_price"].transform("median")
)

# 3️⃣ Final fallback: drop remaining NaN values
df = df.dropna(subset=["Actual_price"]).reset_index(drop=True)

