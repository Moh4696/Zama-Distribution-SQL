# Zama-Distribution-SQL 
This is the Zama Holders distribution SQL code used to generate query on Dune Analysis. 
-- ZAMA Holder Distribution & Top Holders (Post-TGE Analysis)
-- Query 1: Tracks top holders across Ethereum and BSC 
-- TGE Date: February 2, 2026

WITH zama_transfers AS (
    -- Ethereum transfers (incoming) 
    SELECT 
        'Ethereum' AS chain
        , "to" AS holder
        , CAST(value AS DOUBLE) AS amount
        , evt_block_time AS transfer_time
    FROM erc20_ethereum.evt_Transfer
    WHERE contract_address = 0xa12cc123ba206d4031d1c7f6223d1c2ec249f4f3
    
    UNION ALL   
    
    -- Ethereum transfers (outgoing)
    SELECT  
        'Ethereum' AS chain  
        , "from" AS holder
        , -CAST(value AS DOUBLE) AS amount
        , evt_block_time AS transfer_time
    FROM erc20_ethereum.evt_Transfer
    WHERE contract_address = 0xa12cc123ba206d4031d1c7f6223d1c2ec249f4f3
        AND "from" != 0x0000000000000000000000000000000000000000
    
    UNION ALL
    
    -- BSC transfers (incoming)
    SELECT 
        'BSC' AS chain
        , "to" AS holder
        , CAST(value AS DOUBLE) AS amount
        , evt_block_time AS transfer_time
    FROM erc20_bnb.evt_Transfer
    WHERE contract_address = 0x6907a5986c4950bdaf2f81828ec0737ce787519f
    
    UNION ALL
    
    -- BSC transfers (outgoing)
    SELECT 
        'BSC' AS chain
        , "from" AS holder
        , -CAST(value AS DOUBLE) AS amount
        , evt_block_time AS transfer_time
    FROM erc20_bnb.evt_Transfer
    WHERE contract_address = 0x6907a5986c4950bdaf2f81828ec0737ce787519f
        AND "from" != 0x0000000000000000000000000000000000000000
),

holder_balances_by_chain AS (
    SELECT 
        holder
        , chain
        , SUM(amount) / 1e18 AS balance_zama
        , MIN(transfer_time) AS first_acquisition
        , MAX(transfer_time) AS last_transfer
        , COUNT(*) AS transfer_count
    FROM zama_transfers
    GROUP BY 1, 2
    HAVING SUM(amount) > 0  -- Only positive balances
),

holder_summary AS (
    SELECT 
        holder
        , SUM(balance_zama) AS total_balance_zama
        , COUNT(DISTINCT chain) AS chains_held_on
        , array_join(array_agg(chain || ': ' || CAST(ROUND(balance_zama, 2) AS VARCHAR)), ', ') AS chain_breakdown
        , MIN(first_acquisition) AS first_acquisition_date
        , MAX(last_transfer) AS last_transfer_date
        , SUM(transfer_count) AS total_transfers
    FROM holder_balances_by_chain
    GROUP BY 1
)

SELECT 
    ROW_NUMBER() OVER (ORDER BY total_balance_zama DESC) AS rank
    , holder AS address
    , ROUND(total_balance_zama, 2) AS balance_zama
    
    -- Percentage of total supply (1B ZAMA total)
    , ROUND(total_balance_zama / 1000000000 * 100, 4) AS pct_of_supply
    
    -- Holder classification
    , CASE 
        WHEN total_balance_zama >= 10000000 THEN 'Whale (10M+)'
        WHEN total_balance_zama >= 1000000 THEN 'Large (1M+)'
        WHEN total_balance_zama >= 100000 THEN 'Medium (100k+)'
        WHEN total_balance_zama >= 10000 THEN 'Small (10k+)'
        ELSE 'Retail (<10k)'
      END AS holder_type
    
    -- Chain distribution
    , chains_held_on
    , chain_breakdown
    
    -- Activity metrics
    , total_transfers
    , first_acquisition_date
    , last_transfer_date
    , DATE_DIFF('day', first_acquisition_date, CURRENT_DATE) AS days_holding
    , DATE_DIFF('day', last_transfer_date, CURRENT_DATE) AS days_since_last_activity
    
    -- Post-TGE flag (acquired on/after Feb 2, 2026)
    , CASE 
        WHEN first_acquisition_date >= TIMESTAMP '2026-02-02 00:00:00' THEN 'Post-TGE'
        ELSE 'Pre-TGE'
      END AS acquisition_period
    
FROM holder_summary
WHERE total_balance_zama > 0
ORDER BY total_balance_zama DESC
LIMIT 10000
