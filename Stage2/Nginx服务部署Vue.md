# Nginx服务部署vue

## 构建Vue项目

```bash
npm run build
# 或者
yarn build
```

将生成的dist文件上传至Nginx服务器中指定的web根目录

### 写一个server块配置Vue文件位置

```nginx
server{
    listen 80;
    server_name my-vue-app.com;

    root /code/my-vue-app;

    index index.html;

    # -------------------------------------------------------------------
    # 核心配置：处理 Vue 路由（History 模式）
    # 这是至关重要的，它确保所有非静态文件的请求都被重写到 index.html。
    # 这样，无论是访问 /users 还是 /about，Nginx 都会返回 index.html，
    # 从而让 Vue Router 接管并处理前端路由。
    # -------------------------------------------------------------------
    location / {
        try_files $uri $uri/ /index.html;
    }
    # -------------------------------------------------------------------
    # 可选配置：设置缓存策略 (提高性能)
    # -------------------------------------------------------------------
    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        expires 30d;
        log_not_found off;
    }

}
```

