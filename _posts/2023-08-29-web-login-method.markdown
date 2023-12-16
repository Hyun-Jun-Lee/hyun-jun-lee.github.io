---
title: "Login 방식" #Article title.
date: 2023-08-29
category: [NETWORK, NETWORK] #One, more categories or no at all.
tag: [network,login,session,cookie]
---

# Session / Cookie

![image](https://github.com/Log-Stack/logster/assets/76996686/8b8f4fe0-dd68-4221-8aeb-f8b053fb74e9)

1. 클라이언트가 서버에게 로그인을 요청
2. 서버에서 login 인증을하고 전달 받은 request 정보가 맞으면 객체를 생성하고 session ID 를 클라이언트에 전달
3. Session 객체는 서버에 저장
4. 클라이언트가 서버에 요청할 때 요청 할 때 세션 ID가 같이 전달
5. 서버에서는 클라이언트에게 받은 요청헤더에 session ID로 Session 객체를 검색해서 정보 확인

브라우저에 존재하는 Session ID를 발급해, 매 요청마다 브라우저의 쿠키를 검증하여 Session ID를 통해 사용자를 인증하는 방식

Session ID를 전달하는데 쿠키를 이용한다는 것은 서버에 요청을 하지 않기 때문에 클라이언트가 인증 정보를 책임져야한다는 뜻이고 이는 보안상 위험히다.

쿠키는 브라우저에만 있고 네이티브 앱에는 없기 때문에 모바일 앱에서는 인증을 위해 JWT 토큰 방식을 많이 이용한다.

# JWT

JWT는 인증에 필요한 모든 정보들을 암호화 시킨 토큰을 뜻한다. 세션 방식에서는 쿠키에 세션 id를 담아 사용자를 인증할 수 있는 정보를 확인했다면 JWT에서는 Token에 이러한 정보를 넣어서 보낸다.

![image](https://github.com/Log-Stack/logster/assets/76996686/62fa334f-f0b8-4539-aa83-cd0a60319993)

Header에는 해쉬 알고리즘 관련 정보, Payload에는 암호화 하고자 하는 정보가 저장되어 있고

Signature는 사용자 정보를 기반으로 헤더에 있는 알고리즘을 통해 암호화 되어 있다. 그래서 해커가 payload 값을 바꿔도 유효하지 않은 토큰으로 인식된다.

Header, Payload는 16진수로 인코딩만 할 뿐 암호화하지 않기 때문에 payload에 담긴 유저의 정보가 쉽게 노출 될 수 있지만, Signature에 저장된 secrect key를 알지 못하면 복호화 할 수 없다. 

서버는 토큰의 유효성을 검증하기 위해 토큰을 디코딩하고, 토큰을 서명한 키를 사용하여 서명을 확인한다. 이때 서버 측에서는 토큰 검증을 위한 추가적인 데이터베이스 저장소를 사용하지 않는다.

만약 A가 B의 데이터를 훔치려고 payload에 있는 ID를 B의 ID로 변경후 16진수로 인코딩하여 토큰을 서버로 보냈다 가정해보면 

서버는 암호화된 Signature를 검사하게 되는데, payload에는 B 사용자의 정보가 들어가 있으나 Signature는 A의 payload를 기반으로 암호화 되어 있기 때문에 유효하지 않은 토큰으로 판정된다.

![image](https://github.com/Log-Stack/logster/assets/76996686/d2cdf0af-d748-4eed-9296-ba652981dc2d)

1. 클라이언트가 서버에게 로그인을 요청
2. 서버는 유저 확인 후 JWT 생성하여 클라이언트에 Access Token, Refresh Token 전달
3. 클라이언트는 요청 때 마다 서버에게서 받은 Access Token과 같이 request 전달
4. 서버는 Access Token을 검증하고 응답

만약 클라이언트가 요청을 보냈는데 Access Token이 만료되었다면, 서버에서 Access Token이 만료된 것을 응답을 클라이언트에 보낸다.

이후 클라이언트가 토큰 만료 응답을 받고 Aceess Token과 Refresh Toekn을 담아 Access Token 발급 요청을 보낸다. 서버는 Refresh Token을 확인한 후 Access Token 재발급 해준다.

Session 방식은 서버 쪽에서 관리가 가능하기 때문에 제어가 쉽지만 JWT 토큰은 클라이언트에 전달 하고나면 서버 쪽에서 제어할 수가 없기 때문에, RefreshToken을 활용한다.

로그인시 발급되는 AccessToken은 인증에 사용되며 유효기간이 짧게 설정된다. 이 유효기간이 만료되었을 때 재발급을 위해 RefreshToken을 사용한다.

# OAuth

별도 회원가입 없이 외부 서비스에서 인증을 가능하게 하고 해당 서비스의 API를 이용하게 해주는 프로토콜

![image](https://github.com/Log-Stack/logster/assets/76996686/99a9b6a8-2c8b-4d05-9a8e-83fabb2815c6)

1. 클라이언트가 인증 서버에 로그인 요청
2. 인증 서버가 로그인 페이지 제공하고 클라이언트가 정보 전달
3. 정보 확인 후 인증 서버에 Authorization Code를 발급하고, 이 코드로 인증 서버에 Access Token 발급 요청
4. 인증 서버에서 Access Token을 발급해주면 클라이언트는 요청할 때 마다 이 토큰일 이용해서 요청

JWT 방식과 마찬가지로 Access Token 만료시 인증서버에서 Refresh Token을 보내 재발급 받는다.