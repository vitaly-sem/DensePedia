# Безопасность в .NET — разделы

Практическое руководство по безопасности .NET-приложений: от OWASP Top 10 до защиты конфигурации и транспортного уровня. Каждый раздел содержит факты, примеры кода и чек-лист для аудита.

| # | Файл | Темы |
|---|------|------|
| 1 | [OWASP Top 10](01-owasp-top10.md) | SQL Injection, XSS, CSRF, SSRF, Path Traversal, XXE, Deserialization, Broken Access Control, Security Misconfiguration, Logging & Monitoring |
| 2 | [Аутентификация и авторизация](02-authentication-authorization.md) | JWT, OAuth 2.0, OpenID Connect, WebAuthn/Passkeys, ASP.NET Core Identity, Claims, Policies, API Keys, Certificate Auth |
| 3 | [Шифрование и хеширование](03-encryption-hashing.md) | AES-GCM, ChaCha20-Poly1305, RSA, ECDSA, DPAPI, PBKDF2, bcrypt, argon2id, X.509, Hybrid Encryption |
| 4 | [Безопасность API](04-api-security.md) | Rate Limiting, CORS, Input Validation, Mass Assignment, Audit Logging, Versioning, GraphQL Security, WebSocket Security |
| 5 | [Конфигурация и транспорт](05-configuration-transport.md) | Secrets, Key Vault, HTTPS, HSTS, mTLS, TLS 1.3, Certificate Pinning, Secure Headers, Dependency Confusion, Supply Chain, Kubernetes Security |
