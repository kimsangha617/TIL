# Nginx

## nginx, tomcat, ssl 설정 관련

### As-Is
 [] tomcat - ssl
 
### To-Be
 [] nginx - tomcat - ssl


Nginx 를 앞단에 두고 Tomcat을 Reverse Proxy로 사용할 때, SSL 설정 및 Reverse Proxy 구성을 변경해야 한다.

1. Nginx SSL 설정
Nginx에서 SSL 설정하려면 SSL 인증서 및 개인 키파일이 필요. 이 파일들을 획득 후 Nginx 설정 파일을 수정.

아래는 기존 Tomcat 에서 SSL이 설정되어 있고 Nginx를 Reverse Proxy로 사용할 경우의 설정 가이드라인이다.

***
server {
    listen 443 ssl;
    server_name your_domain.com;

    ssl_certificate /path/to/your/certificate.crt;
    ssl_certificate_key /path/to/your/private.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384';

    location ~* /(login|signin) {
        proxy_pass http://tomcat_server:tomcat_port;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
	
	location /api {
        rewrite /api/(.*) /$1 break;
        proxy_pass http://internal-was-internal-lb-862573307.ap- northeast-2.elb.amazonaws.com:8080;
        proxy_set_header Host $host; 
    }
	
}
***
여기서 `your_domain.com` 은 사용자 도메인으로 대체 되어야 한다. SSL 인증서 및 개인 키 파일의 경로도 실제 파일 경로로 변경해야 한다.

* location ~* /(login|signin){ ...} : nginx 요청으로 login, signin URL 값이 들어오는 경우 해당 페이지로 라우팅

* Nginx proxy_pass 설정 : GET http://{nginx-lb-dns}/api/user/login 같이 api 요청이 들어 왔을 경우 nginx-lb-dns 으로 프록시 해준다.
  * proxy_pass : 뒤에 오는 dns/ip 주소로 요청을 전달한다.
  * rewrite : 들어온 요청의 RUL의 규칙을 변경한다 (재작성)
   * rewrite {a}를 {b}로 변경 (정규표현식)
   * /api/user/login -> /user/login 으로 api 제거 이후 받은 요청을 private-dns 서버로 프록시
  * proxy_set_header: 전달할 요청의 header 값 설정, 웹 서버로 받은 요청은 그대로 전달


2. Tomcat 설정 수정
Tomcat에서는 일반적으로 SSL을 설정하는데 Connector를 사용한다.

그러나 Nginx에서 SSL을 처리하므로 Tomcat에서는 일반 HTTP로 통신하도록 설정을 변경해야 한다.

server.xml 파일에서 SSL Connector를 주석 처리하거나 제거한다.

위의 예시에서 Connector를 주석 처리했다.

그리고 context.xml 파일에서 scheme 속성을 https에서 http로 변경.

***
<Context ... scheme="http">
***

3. Nginx와 Tomcat 통합 테스트
설정 변경 후에는 Nginx와 Tomcat이 올바르게 통합되는지 확인하기 위해 테스트를 수행.

브라우저나 curl 명령 등을 사용하여 HTTPS로 접속하고, 요청이 Nginx를 통해 Tomcat으로 전달되는지 확인.

4. 재시작 및 로깅
Nginx와 Tomcat을 재시작하고, 설정 변경 시 발생하는 로그를 모니터링하여 문제가 없는지 확인한다.

이러한 단계를 통해 Nginx를 Reverse Proxy로 사용하는 환경에서 Tomcat의 SSL 및 Reverse Proxy 설정을 적절하게 조정할 수 있다.

### Nginx와 Tomcat 맵핑

Nginx는 클라이언트로부터의 요청을 받아서 해당 요청을 백엔드 서버(Tomcat)로 전달하고, 백엔드 서버의 응답을 클라이언트에게 다시 전달합니다.

중요한 설정 및 고려해야 할 사항은 다음과 같다.

1. proxy_pass 설정
`proxy_pass` 지시어를 사용하여 어느 백엔드 서버로 요청을 전달할지 지정한다. 이를 사용하여 Tomcat의 주소와 포트를 지정한다

***
location / {
    proxy_pass http://tomcat_server:tomcat_port;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
***

여기서 `tomcat_server`는 Tomcat 서버의 호스트 이름이나 IP 주소, `tomcat_port`는 Tomcat이 리스닝하는 포트이다.

2. HTTP 및 HTTPS 구성

Nginx에서는 클라이언트와의 통신을 HTTPS로 설정하고, 백엔드 서버와의 통신은 일반 HTTP로 설정하는 것이 일반적.

위의 Nginx 설정 예시에서는 SSL 설정이 포함되어 있다.

3. 프로토콜 및 헤더 전달 설정
`proxy_set_header` 지시어를 사용하여 필요한 헤더를 전달할 수 있습니다. 예를 들어, `X-Forwarded-For` 헤더를 사용하여 실제 클라이언트의 IP 주소를 전달할 수 있습니다.

4. Context Path 맵핑
톰캣에서는 애플리케이션의 context path를 지정할 수 있다. 만약 톰캣의 애플리케이션에 context path가 적용되어 있다면, Nginx에서도 해당 path를 고려하여 맵핑해야 한다.

예를 들어, 톰캣의 애플리케이션이 http://tomcat_server:tomcat_port/myapp에 배포되어 있을 경우, Nginx 설정에서도 해당 경로를 고려하여 프록시 패스를 지정해야 한다.

***
location /myapp/ {
    proxy_pass http://tomcat_server:tomcat_port;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
***