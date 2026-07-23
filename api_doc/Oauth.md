# 접근토큰 발급
- API 명: 접근토큰 발급
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /oauth2/token
- 콘텐츠 타입: application/x-www-form-urlencoded
## 개요
본인을 인증하는 확인 절차로, 접근 토큰을 부여받아 오픈 API  활용이 가능합니다.<br />※ API호출 유량은 1분에 1건 입니다. <br /> ※ 접근토큰 유효기간 개인/법인 : 신청일시로부터 24시간 <br /> ※ 유효기간 만료 전 토큰을 발급을 하는경우, 동일한 토큰이 발급됩니다. (만료기간도 동일) 유효기간 만료 전 새 토큰이 필요한 경우 접근토큰 폐기 후  발급 부탁드립니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|OAuth2 호출 Request Body 포맷으로 "application/x-www-form-urlencoded" 설정|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|grant_type|권한부여 Type|string|Y|100|"client_credentials" 고정|
|appkey|고객 앱Key|string|Y|36|포탈에서 발급된 고객의 앱Key|
|appsecretkey|고객 앱 비밀Key|string|Y|36|포탈에서 발급된 고객의 앱 비밀Key|
|scope|scope|string|Y|256|"oob" 고정|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|OAuth2 호출 Request Body 포맷으로 "application/x-www-form-urlencoded"|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|token|접근토큰|string|Y|1000|G/W 에서 발급하는 접근토큰|
|expire_in|접근토큰 유효기간|string|Y|10|유효기간(초)|
|scope|scope|string|Y|256|"oob"|
|token_type|토큰 유형|string|Y|256|Bearer|
## Example
### Request Example
```text
appkey=PSqzSi7jxxxxxxxxxxxxxxxxxxxxxxxxxxxx&appsecretkey=RPclxxxxxxxxxxxxxxxxxxxxxxxxxxxx&grant_type=client_credentials&scope=oob
```
### Response Example
```json
{
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUxxxxxxxxxxxxxxxxxxx.xxxxxxxxxxxxxxxxxxx.xxxxxxxxxxxxxxxxxxxxxx",
    "scope": "oob",
    "token_type": "Bearer",
    "expires_in": 86400
}
```

# 접근토큰 폐기
- API 명: 접근토큰 폐기
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /oauth2/revoke
- 콘텐츠 타입: application/x-www-form-urlencoded
## 개요
발급받은 접근토큰을 더 이상  활용하지 않을 때 사용합니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|OAuth2 호출 Request Body 포맷으로 "application/x-www-form-urlencoded" 설정|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|appkey|고객 앱Key|string|Y|36|포탈에서 발급된 고객의 앱Key|
|appsecretkey|고객 앱 비밀Key|string|Y|36|포탈에서 발급된 고객의 앱 비밀Key|
|token_type_hint|토큰 유형 hint|string|Y|100|access_token, refresh_token 토큰 타입|
|token|접근토큰|string|Y|1000||
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|OAuth2 호출 Request Body 포맷으로 "application/x-www-form-urlencoded"|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|code|응답코드|string|Y|8||
|msg|응답메세지|string|Y|450||
## Example
### Request Example
```text
appkey=PSqzSi7jxxxxxxxxxxxxxxxxxxxxxxxxxxxx&appsecretkey=RPclxxxxxxxxxxxxxxxxxxxxxxxxxxxx&token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUxxxxxxxxxxxxxxxxxxx.xxxxxxxxxxxxxxxxxxx.xxxxxxxxxxxxxxxxxxxxxx&token_type_hint=access_token
```
### Response Example
```json
{
    "code": 200,
    "message": "접근토큰 폐기에 성공하였습니다."
}
```
