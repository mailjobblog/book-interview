# Nginx面试题

## Nginx如何进行调优

- 进程数worker_processes配置和cpu核数量一致，最好是设置成 auto 自动匹配进程数。
- 配置连接超时
- 限制上传文件的大小
- 配置 gzip 压缩
- 静态资源配置 expires 缓存期限
- 静态资源配置防盗链