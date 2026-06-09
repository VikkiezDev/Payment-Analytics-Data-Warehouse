1. The value '$97.15' of the type "STRING" cannot be cast to "BIGINT" in 'paymentdw.bronze.transactions_data'
    - In the silver layer, remove the '$' sign.

2. The table 'olist_reviews' has values like "" and 'seria mais coerente."' meaning there could be more such string data in the review_score column.
    - In the silver layer,  using regexp or other functions to remove such data.

3. The table 'olist_products' has total 32951 rows but has 610 null rows for the 'categories' column
    - In the silver layer, replace all the null with other/unknown as value.

4. The table 'olist_order_payments' has 9 zero or negative values for the payment_value column
    - Not sure is this a problem and if it needs any handling.

5. The table 'transactions_data' has 13305915 rows of which there are 1652706 null zip values and the column named errors has 13094522 nulls
    - Not sure how to deal with this.

6. In the table 'cards_data' there are total 6146 records of which 31 have credit limit =0 in the credit_limit column
    - Its either wrong or may be they have used up their limit.. so probably made no transactions? i dont know.