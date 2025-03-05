Provide your CLI command here:

# 🌟 Command
```bash
for i in $(grep '"symbol": "TSLA"' transaction-log.txt | grep '"side": "sell"' | grep -o '"order_id": "[^"]*"' | sed 's/"order_id": "//;s/"//'); do curl -s https://example.com/api/$i >> output.txt ; done
```
