# Ploutus

attack tx https://app.blocksec.com/phalcon/explorer/tx/eth/0xa17dc37e1b65c65d20042212fb834974f7faaa961442e3fc05393778705f8474

misconfiguration tx https://app.blocksec.com/phalcon/explorer/tx/eth/0xcfedf63b37a6cd45b21bc94e3de5412fee0765e7dad6b7c8561a01cebd193ab6

Oracle was mistakenly configured to use BTC/USD Chainlink price feed to price USDC leading to severe pricing discrepancy, exploiting this error, attacker was able to borrow 187 ETH by posting only 8 USDC as collateral

Exploit executed at block 24538897, immediatly after misconfiguration transaction was confirmed in block 24538896 indicating the attacker closely monitored and acted on the configuration change
