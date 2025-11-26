---
title: "御林招新题：ipv4"
subtitle: "御林招新题：ipv4"
summary: "记录 wp"
description: "10-14,还没写出来...;10-16,还是没写出来..."
image: ""
date: 2025-10-14
lastmod: 2025-10-16
draft: false
toc:
 enable: true
weight: false
categories: ["CTF"]
tags: ["CTF"]
---

## 题干

附件：

#### docker-compose.yml

```yml
version: '3.8'

services:
  ipv4-challenge-challenge:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - ./server.py:/app/server.py
    environment:
      - PYTHONUNBUFFERED=1
    restart: always

networks:
  default:
    driver: bridge
```

#### dockerfile

```cmd
FROM python:3.9-alpine

RUN apk add --no-cache bash procps

RUN rm -f /bin/cat /bin/ls /usr/bin/cat /usr/bin/ls /usr/bin/nc /usr/bin/curl /usr/bin/wget /usr/bin/id /usr/bin/whoami

WORKDIR /app

COPY server.py .
COPY templates ./templates  
COPY flag.txt /flag

RUN pip install Flask

EXPOSE 5000

CMD ["python", "server.py"]
```



#### server.py:

```python
app = Flask(__name__, template_folder='templates')
app.config['SECRET_KEY'] = 'xxxxxxx'

USERS = {}

def waf_filter(data):
    malicious_keywords = [
        'cat', 'ls', 'id', 'whoami', 'pwd', 
        'nc', 'curl', 'wget', '`', '&', '||', '&&'
    ]
    for keyword in malicious_keywords:
        if keyword in data:
            return True
    return False
# 登录逻辑
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        user_info = USERS.get(username)

        if user_info and user_info['password'] == password:
            user_data = {"username": username, "is_admin": user_info['is_admin']}
            session['user'] = base64.b64encode(json.dumps(user_data).encode('utf-8')).decode('utf-8')
            return redirect(url_for('user_home'))
        
        return render_template('login.html', message="登录失败，用户名或密码错误。")
    
    return render_template('login.html')
......
# ping 逻辑
@app.route('/ping', methods=['GET', 'POST'])
def ping_page():
    if 'user' not in session:
        return redirect(url_for('login'))
    
    try:
        user_data = json.loads(base64.b64decode(session['user']).decode('utf-8'))
        if not user_data.get('is_admin'):
            return render_template('ping.html', message="对不起，只有管理员才能使用这个功能。")
    except Exception:
        return redirect(url_for('logout'))
    
    if request.method == 'POST':
        ip_base64 = request.form.get('ip_base64', '')
        if not ip_base64:
            return render_template('ping.html', message="请提供 IP 地址。")
        
        try:
            decoded_ip = base64.b64decode(ip_base64.encode('utf-8')).decode('utf-8')
            
            if waf_filter(urllib.parse.unquote(ip_base64)):
                return render_template('ping.html', message="WAF: 检测到恶意关键字，请求被阻止。")
            
            if not re.match(r'^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$', decoded_ip.split('\n')[0]):
                return render_template('ping.html', message="验证失败：IP 地址格式不正确")
            
            if not all(0 <= int(part) < 256 for part in decoded_ip.split('.')):
                return render_template('ping.html', message="验证失败：IP 地址格式不正确")
                
            if not ipaddress.ip_address(decoded_ip):
                return render_template('ping.html', message="验证失败：IP 地址格式不正确")

        except Exception:
            return render_template('ping.html', message="验证失败：无效的 Base64 编码或 IP 格式错误。")
        
        command = f"echo \"ping -c 1 $(echo '{ip_base64}' | base64 -d)\" | sh"
        
        try:
            process = subprocess.run(
                command,
                shell=True,
                check=True,
                capture_output=True,
                text=True,
                timeout=5,
                executable="/bin/sh"
            )
            return render_template('ping.html', output=process.stdout)
        except subprocess.TimeoutExpired:
            return render_template('ping.html', message="❌ Ping 操作超时，请重试或检查网络连接。")
        except subprocess.CalledProcessError as e:
            return render_template('ping.html', message=f"命令执行失败：{e.stderr}")
        except Exception as e:
            return render_template('ping.html', message=f"发生未知错误：{e}")
    
    return render_template('ping.html')
```

- templates
  - 各个 html 文件

## 尝试

```cmd
D:\software\tools\flask-session-cookie-manager-1.2.2>python flask_session_cookie_manager2.py encode -s "xxxxxxx" -t "{'username':'admin','is_admin':'true'}"
eyJpc19hZG1pbiI6eyIgYiI6ImRISjFaUT09In0sInVzZXJuYW1lIjp7IiBiIjoiWVdSdGFXND0ifX0.aPCvLQ.2wnGTmxlNZK2w-jaIbz3fvYdfIs
```

