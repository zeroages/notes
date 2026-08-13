# ssl证书申请部署与nginx配置流程
参考教程：[SSL 免费证书申请 – Certbot | 菜鸟教程](https://www.runoob.com/http/ssl-certbot.html#:~:text=Certbot%20%E6%98%AF%E4%B8%80%E4%B8%AA%E5%BC%80%E6%BA%90%E7%9A%84%E8%87%AA%E5%8A%A8%E5%8C%96%E5%B7%A5%E5%85%B7%EF%BC%8C%E7%94%A8%E4%BA%8E%E8%8E%B7%E5%8F%96%E5%92%8C%E7%BB%AD%E8%AE%A2%E7%94%B1%20Let%27s%20Encrypt%20%E6%8F%90%E4%BE%9B%E7%9A%84%E5%85%8D%E8%B4%B9%20SSL%2FTLS%20%E8%AF%81%E4%B9%A6%E3%80%82,Let%27s%20Encrypt%20%E6%98%AF%E4%B8%80%E4%B8%AA%E7%94%B1%E4%BA%92%E8%81%94%E7%BD%91%E5%AE%89%E5%85%A8%E7%A0%94%E7%A9%B6%E5%B0%8F%E7%BB%84%EF%BC%88ISRG%EF%BC%89%E8%BF%90%E8%90%A5%E7%9A%84%E8%AF%81%E4%B9%A6%E9%A2%81%E5%8F%91%E6%9C%BA%E6%9E%84%EF%BC%88CA%EF%BC%89%EF%BC%8C%E5%AE%83%E6%8F%90%E4%BE%9B%E4%BA%86%E4%B8%80%E4%B8%AA%E8%87%AA%E5%8A%A8%E5%8C%96%E7%9A%84%E6%B5%81%E7%A8%8B%E6%9D%A5%E7%94%9F%E6%88%90%E5%92%8C%E6%9B%B4%E6%96%B0%E8%AF%81%E4%B9%A6%EF%BC%8C%E4%BD%BF%E5%BE%97%E7%BD%91%E7%AB%99%E7%AE%A1%E7%90%86%E5%91%98%E5%8F%AF%E4%BB%A5%E8%BD%BB%E6%9D%BE%E5%9C%B0%E4%B8%BA%E4%BB%96%E4%BB%AC%E7%9A%84%E7%AB%99%E7%82%B9%E5%90%AF%E7%94%A8%20HTTPS%20%E5%8A%A0%E5%AF%86%E3%80%82%20Certbot%20%E7%9A%84%E4%B8%BB%E8%A6%81%E7%89%B9%E7%82%B9%E5%8C%85%E6%8B%AC%EF%BC%9A%20%E8%87%AA%E5%8A%A8%E5%8C%96%EF%BC%9A%E5%AE%83%E5%8F%AF%E4%BB%A5%E8%87%AA%E5%8A%A8%E9%AA%8C%E8%AF%81%E5%9F%9F%E5%90%8D%E6%89%80%E6%9C%89%E6%9D%83%EF%BC%8C%E5%B9%B6%E7%94%B3%E8%AF%B7%E8%AF%81%E4%B9%A6%E3%80%82)

*  仅申请证书
sudo certbot certonly --nginx -d xx.xx.com

*  nginx配置示例
```nginx
server {
  listen 443 ssl;
  server_name xx.xx.com;

  # SSL 证书配置
  ssl_certificate /etc/letsencrypt/live/xx.xx.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/xx.xx.com/privkey.pem;
  include /etc/letsencrypt/options-ssl-nginx.conf;

  # 安全响应头
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-Frame-Options "SAMEORIGIN" always;

  #前端vue项目
    location / {
        root /home/project/vue/;
        index index.html;
        try_files $uri $uri/ index.html;;
    }
  # 反向代理到 Java 服务
  location /api {
    proxy_pass http://localhost:9520/api;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
   }
}
```