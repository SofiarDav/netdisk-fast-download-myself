# NFD 加密备份

- **备份时间**: 2026-10-06T21:38:58+03:00
- **源码 Commit**: ab8b6878375b2cc6964e07414be1708cae5a3368
ab8b687 Merge pull request #231 from qaiu/cursor/release-047-9e8b
- **加密方式**: age (X25519)
- **公钥**: age12tvhqe7kff54zydg46zf7fz7u88nhk93lk66qznf26hlahcf6cus5u23n9
- **文件**: backup-20261006-213858.tar.gz.age

## 解密方法
```bash
age -d -i AGE-SECRET-KEY-... backup-20261006-213858.tar.gz.age | tar -xz
```
