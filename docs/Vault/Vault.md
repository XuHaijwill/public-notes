# Vault Usage Resources
- [Vault 官方网站](https://www.vaultproject.io/)
- [Vault 入门指引](https://www.vaultproject.io/guides/index.html)
- [Vault 用户文档](https://www.vaultproject.io/docs/index.html)
- [Vault 下载地址](https://www.vaultproject.io/downloads.html)
- [Vault 代码库](https://github.com/hashicorp/vault)
- [Vault 用户社区](https://www.vaultproject.io/community.html)

# create token via API
```bash
policy

path "secret/data/*" {
  capabilities = ["create","read","update","delete","list"]
}
path "kv/data/*" {
  capabilities = ["create","read","update","delete","list"]
}


curl -X 'POST' \
'http://192.168.60.134:8300/v1/auth/token/create' \
-H 'accept: */*' \
-H 'Content-Type: application/json' \
-H 'X-Vault-Token: hvs.qnMooHoMIGsSGl0p5sb8I3Uz' \
-d '{
 "policies": ["my-policy"],
 "ttl": "24h",
 "display_name": "api-created-token",
 "num_uses": 0,
 "renewable": true
}'

# 1) 用 API 创建 token（查看完整响应，确认 auth.client_token 存在）
curl -s \
-H "X-Vault-Token: ${ADMIN_TOKEN}" \
-H "Content-Type: application/json" \
-X POST \
-d '{"policies":["my-policy"],"ttl":"24h","display_name":"api-test-token"}' \
"${VAULT_ADDR}/v1/auth/token/create" | jq .

# 2) 验证 token 自身信息（是否过期、关联策略等）
# CLI:
vault token lookup ${USER_TOKEN}
# 或 API:
curl -s -H "X-Vault-Token: ${USER_TOKEN}" "${VAULT_ADDR}/v1/auth/token/lookup-self" | jq .

# 3) 检查 token 在目标路径的实际能力（是否有 read/create/update/list 等）
curl -s \
-H "X-Vault-Token: ${USER_TOKEN}" \
-H "Content-Type: application/json" \
-X POST \
-d '{"paths":["'"${TARGET_PATH}"'"]}' \
"${VAULT_ADDR}/v1/sys/capabilities-self" | jq .
```