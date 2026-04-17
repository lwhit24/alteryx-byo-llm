# Troubleshooting

## 401 Unauthorized
Check bearer token matches Caddy config.

## 403 from Ollama
Ensure Host header is set:
header_up Host localhost:11434

## 504 timeout (Alteryx)
- Reduce prompt size
- Split into multiple calls
- CPU inference may exceed timeout

## Service checks
sudo systemctl status ollama
sudo systemctl status caddy
