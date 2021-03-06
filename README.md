This role is intented for installation of alerta server and webui.  
Alerta server is installed from pip3 and conifigured for running under uwsgi and systemd.  
Work in progress.  
You can see parameters in defaults/main.yml, but at the very least you need to change those:
```yaml
alerta_server_secret_key: changeme
alerta_server_cors_origins:
  - https://alerta.example.org
alerta_admin_users:
  - admin
alerta_server_allowed_environments:
  - production
  - staging
  - development
```

### Nginx
This role only installs and manages alerta. It does not install and configure mongo and nginx. Use separate roles for that.  
I recommend awesome [nginx role by jdauphant](https://github.com/jdauphant/ansible-role-nginx). Server block for alerta:
```yaml
nginx_sites:
  alert.example.org:
    - listen {{ ansible_default_ipv4["address"] }}:80
    - listen {{ ansible_default_ipv4["address"] }}:443 ssl http2
    - server_name alert.example.org
    - ssl_certificate /etc/nginx/ssl/example.org/example.org.crt
    - ssl_certificate_key /etc/nginx/ssl/example.org/example.org.key
    - ssl_dhparam /etc/nginx/ssl/dhparam.pem
    - access_log /var/log/nginx/alert_example_org_access.log
    - error_log /var/log/nginx/alert_example_org_error.log
    - client_max_body_size 10M
    - if ($scheme = http) { return 301 https://$server_name$request_uri; }
    - location /api { try_files $uri @api; }
    - location @api {
        include uwsgi_params;
        uwsgi_pass 127.0.0.1:3031;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
    - location / {
        root /srv/www/alerta;
        try_files $uri $uri/ /index.html;
      }
```

As for mongo, [this mongodb role](https://github.com/UnderGreen/ansible-role-mongodb) is pretty good

### Socket / http-socket
This role supports for both uwsgi socket and http-socket mode.  
Set _alerta_server_uwsgi_mode_ to _socket_ or _http-socket_ to select which mode you will be using.  
When in file socket mode, you will need to specify following parameters:
```yaml
alerta_server_socket
```
When in http-socket you will need to specify
```yaml
alerta_server_listen_address
```

### Plugins
This role supports installation of plugins with pip3 via alerta_server_external_plugins array.   
Example config:
```yaml
alerta_server_external_plugins:
  - git+https://github.com/alerta/alerta-contrib.git#subdirectory=plugins/telegram
  - git+https://github.com/alerta/alerta-contrib.git#subdirectory=plugins/prometheus
```

#### [Prometheus](https://github.com/alerta/alerta-contrib/tree/master/plugins/prometheus)
After installation you will need to enable plugin:
```yaml
alerta_server_plugins:
  - remote_ip
  - reject
  - heartbeat
  - blackout
  - prometheus
```
And configure it:
```yaml
alerta_server_alertmanager_api_url: http://localhost:9093
alerta_server_alertmanager_silence_days: 1
```

#### [Telegram](https://github.com/alerta/alerta-contrib/tree/master/plugins/telegram)
Enable the same way as prometheus.  
Possible options for telegram plugin:
```
alerta_server_telegram_token: your_telegram_bot_token
alerta_server_telegram_chat_id: your_chat_id
alerta_server_telegram_webhook_url: https://alerta.example.org/api/webhook/telegram
```
Also allows to customize message:
```yaml
alerta_server_telegram_template: |
  """
  {% raw %}*[{{ status }}]: {{ environment }} {{ severity }}*
  {{ event | replace("_","\_") }} on {{ resource }}{% endraw %}
  """
```
