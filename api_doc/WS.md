# 웹소켓 세션 초기화
- API 명: 웹소켓 세션 초기화
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/websocket/disconnectSession
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
접속중인 모든 웹소켓 세션을 초기화 하는 API 입니다.<br><br><b>※ 발급받은 토큰정보와 일치하는 계좌의 세션이 초기화 됩니다.</b><br>
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB증권 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|||||||
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB증권 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|object|Y|||
|acntNo|계좌번호|string|Y|11|웹소켓 세션 초기화를 완료한 계좌번호|
|result|result|string|Y|19|처리 메세지|
## Example
### Request Example
```json
{
}
```
### Response Example
```json
{
	"acntNo" : "11122333344",
	"result": "접속중인 세션이 초기화 되었습니다."
}
```

# [실시간]해외주식 체결가
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:7070
- URL: /websocket
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8

## 개요
해외주식(미국) 실시간 체결가 API 입니다.<br><br><br>※ 해외주식(미국) 무료실시간시세 신청을 하지 않을 경우<br>   실시간 시세를 수신 하실 수 없습니다. (V10,V11 지연시세는 별도 신청없이 사용 가능하십니다.)<br>※ 실시간무료시세(0분) 신청방법<br>     HTS : [7325] 해외주식 실시간 시세 신청<br>     MTS : 해외주식 &gt; 서비스신청 &gt; 실시간시세신청<br>※ 무료 실시간 시세는 전체 시세에 비해 50% 수준의 체결 데이터를 제공합니다.<br>※ 무료 실시간 시세 자동결제 신청 시 당월 거래가 없거나 말일에 보유한 미국주식 잔고가 없을 시 자동 해지됩니다.

## Request Header
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|token|토큰|Y|1000|G/W 에서 발급하는 접근토큰|
|tr_type|TR 타입|Y|1|1: 실시간 시세 등록, 2: 실시간 시세 해제, 3: 계좌 등록|

## Request Body
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|tr_cd|거래코드|Y|3|TR코드입력: V60|
|tr_key|종목코드|Y|20|뉴욕거래소 주식/ETF: "FY" + "종목코드" 나스닥거래소 주식/ETF: "FN + "종목코드" 아멕스거래소 주식/ETF: "FA" + "종목코드"|

## Response Header
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|tr_cd|거래코드|Y|3|TR코드|

## Response Body
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|RealYn|실시간여부(0:지연, 1:실시간)|Y|1||
|symbol|거래소구분(2)+종목코드(16)|Y|18||
|busidate|현지영업일자|Y|8||
|locdate|현지일자|Y|8||
|loctime|현지시간|Y|6||
|kordate|한국일자|Y|8||
|kortime|한국시간|Y|6||
|open|시가|Y|20||
|OpenClr|색참조(+상승, -하락)|Y|1||
|high|고가|Y|20||
|HighClr|색참조(+상승, -하락)|Y|1||
|low|저가|Y|20||
|LowClr|색참조(+상승, -하락)|Y|1||
|last|현재가|Y|20||
|LastClr|색참조(+상승, -하락)|Y|1||
|sign|대비부호|Y|1||
|diff|전일대비|Y|20||
|DiffClr|색참조(+상승, -하락)|Y|1||
|rate|등락율|Y|20||
|RateClr|색참조(+상승, -하락)|Y|1||
|bid|매수호가|Y|20||
|BidClr|색참조(+상승, -하락)|Y|1||
|bidsize|매수잔량|Y|12||
|ask|매도호가|Y|20||
|AskClr|색참조(+상승, -하락)|Y|1||
|asksize|매도잔량|Y|12||
|exevol|체결량|Y|12||
|ExevolClr|색참조(+상승, -하락)|Y|1||
|volume|누적거래량|Y|18||
|amount|누적거래대금|Y|18||
|SessionId|장구분 (0:장중|Y|1||
|BidExevolsum|매수누적체결량|Y|18||
|AskExevolsum|매도누적체결량|Y|18||
|rltv|체결강도|Y|18||
|RltvClr|색참조(+상승, -하락)|Y|1||
|clos|기준가|Y|20||

## Example
### Request Example
```json
{
 "header": {
  "token" : "{{ _.access_token }}",
  "tr_type": "1"
 },
 "body": {
  "tr_cd": "V60",
  "tr_key": "FNAAPL"
 }
}
```
### Response Example
```json
{
 "header": {
  "tr_cd": "V60"
 },
 "body": {
  "symbol": "FNAAPL",
  "RealYn": "1",
  "busidate": "20240215",
  "locdate": "20240215",
  "loctime": "071531",
  "kordate": "20240215",
  "kortime": "211531",
  "open": "183.0000",
  "OpenClr": "-",
  "high": "183.1900",
  "HighClr": "-",
  "low": "182.3100",
  "LowClr": "-",
  "last": "182.9700",
  "LastClr": "-",
  "sign": "5",
  "diff": "-1.1800",
  "DiffClr": "-",
  "rate": "-0.64",
  "RateClr": "-",
  "bid": "182.9200",
  "BidClr": "-",
  "bidsize": "1",
  "ask": "182.9900",
  "AskClr": "-",
  "asksize": "24",
  "exevol": "51",
  "ExevolClr": "+",
  "volume": "57978",
  "amount": "10592396",
  "SessionId": "1",
  "BidExevolsum": "48105",
  "AskExevolsum": "9873",
  "rltv": "487.24",
  "RltvClr": "+",
  "clos": "184.1500"
 }
}
```

# [실시간]해외주식 주문체결 조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:7070
- URL: /websocket
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8

## 개요
해외주식 실시간 주문체결 API 입니다. 주문 체결시 내역이 출력됩니다.

## Request Header
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|token|토큰|Y|1000|G/W 에서 발급하는 접근토큰|
|tr_type|TR 타입|Y|1|1: 실시간 시세 등록, 2: 실시간 시세 해제, 3: 계좌 등록|

## Request Body
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|tr_key|종목코드|Y|20|입력 X|
|tr_cd|거래코드|Y|3|TR코드입력: IS2|

## Response Header
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|tr_cd|거래코드|Y|3|TR코드|

## Response Body
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|Sordxctptncode|주문체결유형코드|Y|2|00: 해당없음<br/>01: 주문<br/>02: 정정<br/>03: 취소<br/>11: 체결<br/>12: 정정확인<br/>13: 취소확인<br/>14: 거부|
|Strancode|서비스 코드|Y|10||
|Sorddt|주문일자|Y|8||
|Sordno|주문번호|Y|10||
|Sorgordno|원주문번호|Y|10||
|Sastkisuno|해외주식종목번호|Y|20||
|Ssymcode|심볼코드|Y|12||
|Sastkisunm|해외주식종목명|Y|40||
|Sastkmktcode|해외주식시장코드|Y|2||
|Sastkmktnm|해외주식시장명|Y|20||
|Sownsecode|자체증권거래소코드|Y|2||
|Sastkbnstpcode|해외주식매매구분코드|Y|1|0: 전체<br/>1: 매도<br/>2: 매수|
|Sastkbnstpnm|해외주식매매구분명|Y|10||
|Sordtrdtpcode|주문거래구분코드|Y|1|0: 원주문<br/>1: 정정주문<br/>2: 취소주문|
|Sastkordqty|해외주식주문수량|Y|16||
|Sastkordprc|해외주식주문가격|Y|16||
|Sastkordprcptncode|해외주식호가유형코드|Y|1|1: 지정가<br/>2: 시장가|
|Sastkordprcptnnm|해외주식호가유형명|Y|20||
|Sastkordcnditpcode|해외주식주문조건구분코드|Y|1|1:FAS <br/>2:IOC <br/>3:FOK|
|Sastkordcnditpnm|해외주식주문조건구분명|Y|10||
|Scrcycode|통화코드|Y|3||
|Sshtncntrysymcode|단축국가심볼코드|Y|2||
|Sastkordstatnm|해외주식주문상태명|Y|20||
|Sastkorddttm|해외주식주문일시|Y|17||
|Sastklclorddttm|해외주식현지주문일시|Y|17||
|Sastkexecqty|체결수량|Y|16||
|Sastkexecprc|체결가격|Y|16||
|Sastkacmexecqty|누적체결수량|Y|16||
|Sastkexecavruprc|체결평균단가|Y|16||
|Sastkunercqty|미체결수량|Y|16||
|Sastkexecamt|체결금액|Y|16||
|Sastkexecdttm|외주식체결일시|Y|17||
|Sastklclexecdttm|해외주식현지체결일시|Y|17||
|Smgntrncode|신용거래코드|Y|3||
|Sloandt|대출일자|Y|8||
|Sastkexecbaseqty|해외주식체결기준수량|Y|16||
|Sastkordableqty|해외주식주문가능수량|Y|16||
|Sastkbuyamt|해외주식매수금액|Y|16||
|Sastkbuycmsn|해외주식매수수수료|Y|16||
|Sthdayrlzpnlamt|당일실현손익금액|Y|16||
|Sastkselllclcmsnrat|해외주식매도현지수수료율|Y|16||
|Sastkbuylclcmsnrat|해외주식매수현지수수료율|Y|16||
|Sacntno|게좌번호|Y|20||

## Example
### Request Example
```json
{
 "header": {
  "token" : "{{ _.access_token }}",
  "tr_type": "3"
 },
 "body": {
  "tr_cd": "IS2"
 }
}
```
### Response Example
```json
{
 "header": {
  "tr_cd": "IS2"
 },
 "body": {
  "Strancode": "CAZCT00100",
  "Sordxctptncode": "01",
  "Sacntno": "11111111111",
  "Sorddt": "20240202",
  "Sordno": "0000000005",
  "Sorgordno": "0000000000",
  "Sastkisuno": "000001.SZ",
  "Ssymcode": "000001",
  "Sastkisunm": "평안은행",
  "Sastkmktcode": "SZ",
  "Sastkmktnm": "심천",
  "Sownsecode": "ZS",
  "Sastkbnstpcode": "2",
  "Sastkbnstpnm": "매수",
  "Sordtrdtpcode": "0",
  "Sastkordqty": "0000000000000100",
  "Sastkordprc": "00000000009.4300",
  "Sastkordprcptncode": "1",
  "Sastkordprcptnnm": "지정가",
  "Sastkordcnditpcode": "1",
  "Sastkordcnditpnm": "일반",
  "Scrcycode": "CNY",
  "Sshtncntrysymcode": "CN",
  "Sastkordstatnm": "송신",
  "Scommdacode": "50",
  "Scommdacodenm": "해피+",
  "Sastkorddttm": "20240202103110235",
  "Sastklclorddttm": "20240202093110023",
  "Sastkexecqty": "0000000000000000",
  "Sastkexecprc": "00000000000.0000",
  "Sastkacmexecqty": "0000000000000000",
  "Sastkexecavruprc": "00000000000.0000",
  "Sastkunercqty": "0000000000000100",
  "Sastkexecamt": "0000000000000.00",
  "Sastkexecdttm": "",
  "Sastklclexecdttm": "",
  "Smgntrncode": "000",
  "Sloandt": "",
  "Sastkexecbaseqty": "0000000000000100",
  "Sastkordableqty": "0000000000000000",
  "Sastkbuyamt": "0000000000941.00",
  "Sastkbuycmsn": "0000000000000.55",
  "Sthdayrlzpnlamt": "0000000000000000",
  "Sastkselllclcmsnrat": "0000.05000000000",
  "Sastkbuylclcmsnrat": "0000.05841000000"
 }
}
```

# [실시간]해외주식 호가
- API 명: 웹소켓 세션 초기화
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/websocket/disconnectSession
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식(미국) 실시간 호가 API 입니다.

## Request Header
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|token|토큰|Y|1000|G/W 에서 발급하는 접근토큰|
|tr_type|TR 타입|Y|1|1: 실시간 시세 등록, 2: 실시간 시세 해제, 3: 계좌 등록|

## Request Body
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|tr_cd|거래코드|Y|3|TR코드입력: V61|
|tr_key|종목코드|Y|20|뉴욕거래소 주식/ETF: "FY" + "종목코드" 나스닥거래소 주식/ETF: "FN + "종목코드" 아멕스거래소 주식/ETF: "FA" + "종목코드"|

## Response Header
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|tr_cd|거래코드|Y|3|TR코드|

## Response Body
|요소|한글명|필수여부|길이|설명|
|---|---|---|---|---|
|RealYn|실시간여부(0:지연, 1:실시간)|Y|1||
|symbol|종목코드|Y|18||
|locdate|현지일자|Y|8||
|loctime|현지시간|Y|6||
|kordate|한국일자|Y|8||
|kortime|한국시간|Y|6||
|totbidsize|매수호가 총잔량|Y|12||
|totasksize|매도호가 총잔량|Y|12||
|totbidcount|매도호가 총건수|Y|12||
|totaskcount|매수호가 총건수|Y|12||
|Bid1|매수1호가|Y|20||
|Bid1clr|색참조(+상승, -하락)|Y|1||
|Ask1|매도1호가|Y|20||
|Ask1clr|색참조(+상승, -하락)|Y|1||
|Bidsize1|매수1호가잔량|Y|12||
|Asksize1|매도1호가잔량|Y|12||
|Bidcount1|매수1호가건수|Y|12||
|Askcount1|매도1호가건수|Y|12||
|Bid2|매수2호가|Y|20||
|Bid2clr|색참조(+상승, -하락)|Y|1||
|Ask2|매도2호가|Y|20||
|Ask2clr|색참조(+상승, -하락)|Y|1||
|Bidsize2|매수2호가잔량|Y|12||
|Asksize2|매도2호가잔량|Y|12||
|Bidcount2|매수2호가건수|Y|12||
|Askcount2|매도2호가건수|Y|12||
|Bid3|매수3호가|Y|20||
|Bid3clr|색참조(+상승, -하락)|Y|1||
|Ask3|매도3호가|Y|20||
|Ask3clr|색참조(+상승, -하락)|Y|1||
|Bidsize3|매수3호가잔량|Y|12||
|Asksize3|매도3호가잔량|Y|12||
|Bidcount3|매수3호가건수|Y|12||
|Askcount3|매도3호가건수|Y|12||
|Bid4|매수4호가|Y|20||
|Bid4clr|색참조(+상승, -하락)|Y|1||
|Ask4|매도4호가|Y|20||
|Ask4clr|색참조(+상승, -하락)|Y|1||
|Bidsize4|매수4호가잔량|Y|12||
|Asksize4|매도4호가잔량|Y|12||
|Bidcount4|매수4호가건수|Y|12||
|Askcount4|매도4호가건수|Y|12||
|Bid5|매수5호가|Y|20||
|Bid5clr|색참조(+상승, -하락)|Y|1||
|Ask5|매도5호가|Y|20||
|Ask5clr|색참조(+상승, -하락)|Y|1||
|Bidsize5|매수5호가잔량|Y|12||
|Asksize5|매도5호가잔량|Y|12||
|Bidcount5|매수5호가건수|Y|12||
|Askcount5|매도5호가건수|Y|12||
|Bid6|매수6호가|Y|20||
|Bid6clr|색참조(+상승, -하락)|Y|1||
|Ask6|매도6호가|Y|20||
|Ask6clr|색참조(+상승, -하락)|Y|1||
|Bidsize6|매수6호가잔량|Y|12||
|Asksize6|매도6호가잔량|Y|12||
|Bidcount6|매수6호가건수|Y|12||
|Askcount6|매도6호가건수|Y|12||
|Bid7|매수7호가|Y|20||
|Bid7clr|색참조(+상승, -하락)|Y|1||
|Ask7|매도7호가|Y|20||
|Ask7clr|색참조(+상승, -하락)|Y|1||
|Bidsize7|매수7호가잔량|Y|12||
|Asksize7|매도7호가잔량|Y|12||
|Bidcount7|매수7호가건수|Y|12||
|Askcount7|매도7호가건수|Y|12||
|Bid8|매수8호가|Y|20||
|Bid8clr|색참조(+상승, -하락)|Y|1||
|Ask8|매도8호가|Y|20||
|Ask8clr|색참조(+상승, -하락)|Y|1||
|Bidsize8|매수8호가잔량|Y|12||
|Asksize8|매도8호가잔량|Y|12||
|Bidcount8|매수8호가건수|Y|12||
|Askcount8|매도8호가건수|Y|12||
|Bid9|매수9호가|Y|20||
|Bid9clr|색참조(+상승, -하락)|Y|1||
|Ask9|매도9호가|Y|20||
|Ask9clr|색참조(+상승, -하락)|Y|1||
|Bidsize9|매수9호가잔량|Y|12||
|Asksize9|매도9호가잔량|Y|12||
|Bidcount9|매수9호가건수|Y|12||
|Askcount9|매도9호가건수|Y|12||
|Bid10|매수10호가|Y|20||
|Bid10clr|색참조(+상승, -하락)|Y|1||
|Ask10|매도10호가|Y|20||
|Ask10clr|색참조(+상승, -하락)|Y|1||
|Bidsize10|매수10호가잔량|Y|12||
|Asksize10|매도10호가잔량|Y|12||
|Bidcount10|매수10호가건수|Y|12||
|Askcount10|매도10호가건수|Y|12||
|TotbidsizeIcdc|매수호가 총잔량 증감|Y|20||
|TotbidsizeIcdcclr|색참조(+상승, -하락)|Y|1||
|TotasksizeIcdc|매도호가 총잔량 증감|Y|20||
|TotasksizeIcdcclr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc1|매수 1호가잔량 증감|Y|20||
|BidsizeIcdc1clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc1|매도 1호가잔량 증감|Y|20||
|AsksizeIcdc1clr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc2|매수 2호가잔량 증감|Y|20||
|BidsizeIcdc2clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc2|매도 2호가잔량 증감|Y|20||
|AsksizeIcdc2clr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc3|매수 3호가잔량 증감|Y|20||
|BidsizeIcdc3clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc3|매도 3호가잔량 증감|Y|20||
|AsksizeIcdc3clr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc4|매수 4호가잔량 증감|Y|20||
|BidsizeIcdc4clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc4|매도 4호가잔량 증감|Y|20||
|AsksizeIcdc4clr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc5|매수 5호가잔량 증감|Y|20||
|BidsizeIcdc5clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc5|매도 5호가잔량 증감|Y|20||
|AsksizeIcdc5clr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc6|매수 6호가잔량 증감|Y|20||
|BidsizeIcdc6clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc6|매도 6호가잔량 증감|Y|20||
|AsksizeIcdc6clr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc7|매수 7호가잔량 증감|Y|20||
|BidsizeIcdc7clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc7|매도 7호가잔량 증감|Y|20||
|AsksizeIcdc7clr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc8|매수 8호가잔량 증감|Y|20||
|BidsizeIcdc8clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc8|매도 8호가잔량 증감|Y|20||
|AsksizeIcdc8clr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc9|매수 9호가잔량 증감|Y|20||
|BidsizeIcdc9clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc9|매도 9호가잔량 증감|Y|20||
|AsksizeIcdc9clr|색참조(+상승, -하락)|Y|1||
|BidsizeIcdc10|매수10호가잔량 증감|Y|20||
|BidsizeIcdc10clr|색참조(+상승, -하락)|Y|1||
|AsksizeIcdc10|매도10호가잔량 증감|Y|20||
|AsksizeIcdc10clr|색참조(+상승, -하락)|Y|1||
|Bid1krw|매수1호가 원화|Y|20||
|Bid1krwclr|색참조(+상승, -하락)|Y|1||
|Ask1krw|매도1호가 원화|Y|20||
|Ask1krwclr|색참조(+상승, -하락)|Y|1||
|Bid2krw|매수2호가 원화|Y|20||
|Bid2krwclr|색참조(+상승, -하락)|Y|1||
|Ask2krw|매도2호가 원화|Y|20||
|Ask2krwclr|색참조(+상승, -하락)|Y|1||
|Bid3krw|매수3호가 원화|Y|20||
|Bid3krwclr|색참조(+상승, -하락)|Y|1||
|Ask3krw|매도3호가 원화|Y|20||
|Ask3krwclr|색참조(+상승, -하락)|Y|1||
|Bid4krw|매수4호가 원화|Y|20||
|Bid4krwclr|색참조(+상승, -하락)|Y|1||
|Ask4krw|매도4호가 원화|Y|20||
|Ask4krwclr|색참조(+상승, -하락)|Y|1||
|Bid5krw|매수5호가 원화|Y|20||
|Bid5krwclr|색참조(+상승, -하락)|Y|1||
|Ask5krw|매도5호가 원화|Y|20||
|Ask5krwclr|색참조(+상승, -하락)|Y|1||
|Bid6krw|매수6호가 원화|Y|20||
|Bid6krwclr|색참조(+상승, -하락)|Y|1||
|Ask6krw|매도6호가 원화|Y|20||
|Ask6krwclr|색참조(+상승, -하락)|Y|1||
|Bid7krw|매수7호가 원화|Y|20||
|Bid7krwclr|색참조(+상승, -하락)|Y|1||
|Ask7krw|매도7호가 원화|Y|20||
|Ask7krwclr|색참조(+상승, -하락)|Y|1||
|Bid8krw|매수8호가 원화|Y|20||
|Bid8krwclr|색참조(+상승, -하락)|Y|1||
|Ask8krw|매도8호가 원화|Y|20||
|Ask8krwclr|색참조(+상승, -하락)|Y|1||
|Bid9krw|매수9호가 원화|Y|20||
|Bid9krwclr|색참조(+상승, -하락)|Y|1||
|Ask9krw|매도9호가 원화|Y|20||
|Ask9krwclr|색참조(+상승, -하락)|Y|1||
|Bid10krw|매수10호가 원화|Y|20||
|Bid10krwclr|색참조(+상승, -하락)|Y|1||
|Ask10krw|매도10호가 원화|Y|20||
|Ask10krwclr|색참조(+상승, -하락)|Y|1||

## Example
### Request Example
```json
{
 "header": {
  "token" : "{{ _.access_token }}",
  "tr_type": "1"
 },
 "body": {
  "tr_cd": "V61",
  "tr_key": "FNTSLA"
 }
}
```
### Response Example
```json
{
 "header": {
  "tr_cd": "V61"
 },
 "body": {
  "symbol": "FNTSLA",
  "RealYn": "1",
  "locdate": "20240215",
  "loctime": "071618",
  "kordate": "20240215",
  "kortime": "211618",
  "totbidsize": "2",
  "totasksize": "100",
  "totbidcount": "0",
  "totaskcount": "0",
  "Bid1": "190.8000",
  "Bid1clr": "+",
  "Ask1": "190.8500",
  "Ask1clr": "+",
  "Bidsize1": "2",
  "Asksize1": "100",
  "Bidcount1": "0",
  "Askcount1": "0",
  "Bid2": "190.7900",
  "Bid2clr": "+",
  "Ask2": "190.8600",
  "Ask2clr": "+",
  "Bidsize2": "0",
  "Asksize2": "0",
  "Bidcount2": "0",
  "Askcount2": "0",
  "Bid3": "190.7800",
  "Bid3clr": "+",
  "Ask3": "190.8700",
  "Ask3clr": "+",
  "Bidsize3": "0",
  "Asksize3": "0",
  "Bidcount3": "0",
  "Askcount3": "0",
  "Bid4": "190.7700",
  "Bid4clr": "+",
  "Ask4": "190.8800",
  "Ask4clr": "+",
  "Bidsize4": "0",
  "Asksize4": "0",
  "Bidcount4": "0",
  "Askcount4": "0",
  "Bid5": "190.7600",
  "Bid5clr": "+",
  "Ask5": "190.8900",
  "Ask5clr": "+",
  "Bidsize5": "0",
  "Asksize5": "0",
  "Bidcount5": "0",
  "Askcount5": "0",
  "Bid6": "190.7500",
  "Bid6clr": "+",
  "Ask6": "190.9000",
  "Ask6clr": "+",
  "Bidsize6": "0",
  "Asksize6": "0",
  "Bidcount6": "0",
  "Askcount6": "0",
  "Bid7": "190.7400",
  "Bid7clr": "+",
  "Ask7": "190.9100",
  "Ask7clr": "+",
  "Bidsize7": "0",
  "Asksize7": "0",
  "Bidcount7": "0",
  "Askcount7": "0",
  "Bid8": "190.7300",
  "Bid8clr": "+",
  "Ask8": "190.9200",
  "Ask8clr": "+",
  "Bidsize8": "0",
  "Asksize8": "0",
  "Bidcount8": "0",
  "Askcount8": "0",
  "Bid9": "190.7200",
  "Bid9clr": "+",
  "Ask9": "190.9300",
  "Ask9clr": "+",
  "Bidsize9": "0",
  "Asksize9": "0",
  "Bidcount9": "0",
  "Askcount9": "0",
  "Bid10": "190.7100",
  "Bid10clr": "+",
  "Ask10": "190.9400",
  "Ask10clr": "+",
  "Bidsize10": "0",
  "Asksize10": "0",
  "Bidcount10": "0",
  "Askcount10": "0",
  "TotbidsizeIcdc": "0",
  "TotbidsizeIcdcclr": "",
  "TotasksizeIcdc": "81",
  "TotasksizeIcdcclr": "+",
  "BidsizeIcdc1": "0",
  "BidsizeIcdc1clr": "",
  "AsksizeIcdc1": "81",
  "AsksizeIcdc1clr": "+",
  "BidsizeIcdc2": "0",
  "BidsizeIcdc2clr": "",
  "AsksizeIcdc2": "0",
  "AsksizeIcdc2clr": "",
  "BidsizeIcdc3": "0",
  "BidsizeIcdc3clr": "",
  "AsksizeIcdc3": "0",
  "AsksizeIcdc3clr": "",
  "BidsizeIcdc4": "0",
  "BidsizeIcdc4clr": "",
  "AsksizeIcdc4": "0",
  "AsksizeIcdc4clr": "",
  "BidsizeIcdc5": "0",
  "BidsizeIcdc5clr": "",
  "AsksizeIcdc5": "0",
  "AsksizeIcdc5clr": "",
  "BidsizeIcdc6": "0",
  "BidsizeIcdc6clr": "",
  "AsksizeIcdc6": "0",
  "AsksizeIcdc6clr": "",
  "BidsizeIcdc7": "0",
  "BidsizeIcdc7clr": "",
  "AsksizeIcdc7": "0",
  "AsksizeIcdc7clr": "",
  "BidsizeIcdc8": "0",
  "BidsizeIcdc8clr": "",
  "AsksizeIcdc8": "0",
  "AsksizeIcdc8clr": "",
  "BidsizeIcdc9": "0",
  "BidsizeIcdc9clr": "",
  "AsksizeIcdc9": "0",
  "AsksizeIcdc9clr": "",
  "BidsizeIcdc10": "0",
  "BidsizeIcdc10clr": "",
  "AsksizeIcdc10": "0",
  "AsksizeIcdc10clr": "",
  "Bid1krw": "254374",
  "Bid1krwclr": "+",
  "Ask1krw": "254441",
  "Ask1krwclr": "+",
  "Bid2krw": "254361",
  "Bid2krwclr": "+",
  "Ask2krw": "254454",
  "Ask2krwclr": "+",
  "Bid3krw": "254347",
  "Bid3krwclr": "+",
  "Ask3krw": "254467",
  "Ask3krwclr": "+",
  "Bid4krw": "254334",
  "Bid4krwclr": "+",
  "Ask4krw": "254481",
  "Ask4krwclr": "+",
  "Bid5krw": "254321",
  "Bid5krwclr": "+",
  "Ask5krw": "254494",
  "Ask5krwclr": "+",
  "Bid6krw": "254307",
  "Bid6krwclr": "+",
  "Ask6krw": "254507",
  "Ask6krwclr": "+",
  "Bid7krw": "254294",
  "Bid7krwclr": "+",
  "Ask7krw": "254521",
  "Ask7krwclr": "+",
  "Bid8krw": "254281",
  "Bid8krwclr": "+",
  "Ask8krw": "254534",
  "Ask8krwclr": "+",
  "Bid9krw": "254267",
  "Bid9krwclr": "+",
  "Ask9krw": "254547",
  "Ask9krwclr": "+",
  "Bid10krw": "254254",
  "Bid10krwclr": "+",
  "Ask10krw": "254561",
  "Ask10krwclr": "+"
 }
}
```
