# Curl Tests

## Test models endpoint
curl -s https://YOUR_DOMAIN/v1/models \
  -H "Authorization: Bearer YOUR_SECRET" | jq .

## Test completion
curl -s https://YOUR_DOMAIN/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_SECRET" \
  -d '{
    "model": "ollama/llama3:8b",
    "messages": [
      {"role": "user", "content": "Test"}
    ]
  }' | jq .
