# API 목록

|순번|서비스 명|API 명|HTTP Method|URL|Domain|포맷|TPS|
|---|---|---|---|---|---|---|---|
|1|해외주식주문|해외주식 주문|POST|/api/v1/trading/overseas-stock/order|https://openapi.dbsec.co.kr:8443|JSON|10|
|2|해외주식주문|해외주식 체결내역조회|POST|/api/v1/trading/overseas-stock/inquiry/transaction-history|https://openapi.dbsec.co.kr:8443|JSON|2|
|3|해외주식주문|해외주식 잔고/증거금 조회|POST|/api/v1/trading/overseas-stock/inquiry/balance-margin|https://openapi.dbsec.co.kr:8443|JSON|3|
|4|해외주식주문|해외주식 매매내역 조회|POST|/api/v1/trading/overseas-stock/inquiry/trading-history|https://openapi.dbsec.co.kr:8443|JSON|2|
|5|해외주식주문|해외주식 거래내역 조회|POST|/api/v1/trading/overseas-stock/inquiry/trade-history|https://openapi.dbsec.co.kr:8443|JSON|2|
|6|해외주식주문|해외주식 주문가능금액조회|POST|/api/v1/trading/overseas-stock/inquiry/able-orderqty|https://openapi.dbsec.co.kr:8443|JSON|2|
|7|해외주식주문|해외주식 실현손익 조회|POST|/api/v1/trading/overseas-stock/inquiry/day-rlzpnl|https://openapi.dbsec.co.kr:8443|JSON|2|
|8|해외주식주문|해외주식 예수금상세|POST|/api/v1/trading/overseas-stock/inquiry/deposit-detail|https://openapi.dbsec.co.kr:8443|JSON|2|
|9|해외주식주문|해외주식 평균매입단가 조회|POST|/api/v1/trading/overseas-stock/inquiry/avg-pur-price|https://openapi.dbsec.co.kr:8443|JSON|2|

# 해외주식 주문
- API 명: 해외주식 주문
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/trading/overseas-stock/order
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식(미국) 주문이 가능한 API입니다.<br><br>   <br> ※ 모의투자 계좌로 사용가능한 API입니다.  <br>※ 해외주식 거래신청 후 이용 가능합니다. <br>    - 신청방법<br>      HTS : [7322] 해외주식 시작하기<br>      MTS : 해외주식 &gt; 서비스신청  &gt; 해외주식 거래신청, 통합증거금 이용신청, 해외ETP(ETF,ETN) 거래신청<br><br>   ※ 수수료 및 제세금 안내는 아래 링크 참고 부탁드립니다. <br> <a href="https://www.dbsec.co.kr/custcenter/jobservice/cu_FeeTrading_viw10.do" target="blank">https://www.dbsec.co.kr/custcenter/jobservice/cu_FeeTrading_viw10.do</a> ※ 해외주식 양도소득세 안내는 아래 링크 참고 부탁드립니다. <br> <a href="https://www.dbsec.co.kr/research/osst/re_OsstDealInfo_viw40.do" target="blank">https://www.dbsec.co.kr/research/osst/re_OsstDealInfo_viw40.do</a> <br>※ 거래시간 <br>    미국 정규장: 23:30 ~ 익일 06:00 (서머타임 적용시 22:30 ~ 익일 05:00) <br>    미국 프리장: 18:00 ~ 23:30 (서머타임 적용시 17:00 ~ 22:30) <br>    미국 데이장(주간): 10:00 ~ 17:30 (서머타임 적용시 09:00 ~ 16:30) <br>    미국 애프터장: 익일 06:00 ~ 07:00 (서머타임 적용시 05:00 ~ 07:00) <br><br>  ※ 해외주식 주문유형 안내<br>  <b>1. 지정가: 희망 거래 가격을 지정하여 주문</b> <br>  &#183; 이용가능 국가: 미국 <br>  &#183; 거래가능 시간: 정규장 프리/애프터마켓 <br>   <b>2. 시장가: 즉시 체결될 수 있는 가격으로 주문</b> <br>  &#183; 이용가능 국가: 미국 <br>  &#183; 거래가능 시간: 정규장 <br>   *  당사 시장가 매수 주문은 자체 기준에 따라 현재가 대비 불리한 가격의 지정가로 주문됩니다. <br>     더불어 현재가보다 기준가격을 높게 산정하므로, 주문가능수량은 지정가 주문수량보다 적을 수 있으니 참고하시기 바랍니다.<br>   <b>3. 장개시 주문: 정규장 시작과 동시에 지정가 또는 시장가로 주문</b> <br>  &#183; MOO(장개시 시장가): 정규장 시작과 동시에 시초가로 체결 시키는 주문(단, 매도만 가능)<br>  &#183; LOO(장개시 지정가): 정규장 시작과 동시에 시초가가 지정한 가격보다 같거나 유리하면 체결 시키는 주문<br>  &#183; 이용가능 국가: 미국 <br>  &#183; 거래가능 시간: 프리마켓 (장 시작 10분 전까지 가능) <br>   <b>4. 장마감 주문: 정규장 종료와 동시에 지정가 또는 시장가로 주문</b> <br>  &#183; MOC(장마감 시장가): 정규장 종료와 동시에 종가로 체결 시키는 주문(단, 매도만 가능)<br>  &#183; LOC(장마감 지정가): 정규장 종료와 동시에 종가가 지정한 가격보다 같거나 유리하면 체결시키는 주문<br>  &#183; 이용가능 국가: 미국 <br>  &#183; 거래가능 시간: 프리마켓, 정규장(장 마감 10분 전까지 가능) <br>   <b>5. 알고리즘 주문: 거래소가 아닌 해외중계업체의 자체 알고리즘에 따라 주문</b> <br>  &#183; TWAP(시간 가중평균가격): 주문 시점 기준부터 비슷한 시간 간격으로 비슷한 수량을 분할해 체결 시키는 주문<br>  &#183; VWAP(거래량 가중평균가격): 주문 시점 기준부터 시장 가격 및 거래량의 변화를 모니터링하여, <br>            거래량이 많을 때는 많이, 거래량이 적을 때는 적게 분할 체결시키는 주문<br>  &#183; 이용가능 국가: 미국 <br>  &#183; 거래가능 시간: 정규장 <br>  * TWAP/VWAP 주문은 지정가, 시장가로 구분되며 매수할 때는 지정가, 매도할 때는 시장가로 주문 가능합니다. <br> * TWAP/VWAP 지정가 매수 주문은 지정한 가격 이하에서만 유형 특성에 따라 분할 체결시키는 주문입니다. <br> * TWAP/VWAP 주문은 100주 단위로 체결되며, 100주 미만으로 주문 가능하나 한 번에 체결될 수 있습니다. <br>   <b>※ 유의사항</b> <br>  &#183; 시장가(MOO, MOC 포함) 주문의 경우 시장 급변동으로 인해 원치 않는 가격으로 체결될 수 있음에 유의하시기 바랍니다.  <br>  &#183; 장전(프리)/정규장 거래시간에 접수된 미체결 주문은 장후(애프터) 거래시간까지 유효합니다.  <br>  &#183; 미국주식 주간장의 주문가능 유형은 "시장가", "지정가" 유형만 지원됩니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|AstkIsuNo|해외주식종목번호|string|Y|20|미국주식/ETF: "종목코드" (EX. AAPL, SOXL, QQQ)|
|AstkBnsTpCode|해외주식매매구분코드|string|Y|1|1:매도	2:매수|
|AstkOrdprcPtnCode|해외주식호가유형코드|string|Y|1|1: 지정가 	2: 시장가 	3: LOO 	4: MOO  (매도주문 시 사용가능)	5: LOC 	6: MOC  (매도주문 시 사용가능)	7: VWAP지정가 (매수/매도 주문 모두 사용가능)	8: TWAP지정가 (매수/매도 주문 모두 사용가능)	9: VWAP시장가 (매도주문시에만 사용가능)	A: TWAP시장가  (매도주문시에만 사용가능)|
|AstkOrdCndiTpCode|해외주식주문조건구분코드|string|Y|1|1:FAS(일반)	2:IOC 	3:FOK|
|AstkOrdQty|해외주식주문수량|number|Y|28||
|AstkOrdPrc|해외주식주문가격|number|Y|28|시장가주문시: 0|
|OrdTrdTpCode|주문거래구분코드|string|Y|1|0:주문	1:정정주문 	2:취소주문|
|OrgOrdNo|원주문번호|number|Y|10|정정주문, 취소주문시 원 주문번호 입력	매수/매도주문시: 0|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|object|Y|||
|OrdNo|주문번호|number|Y|10||
## Example
### Request Example
```json
{ 
  "In": {
    "AstkIsuNo" : "SOXL",
    "AstkBnsTpCode" : "2",
    "AstkOrdprcPtnCode" : "2",
    "AstkOrdCndiTpCode" : "1",
    "AstkOrdQty" : 1,
    "AstkOrdPrc" : 0,
    "OrdTrdTpCode" : "0",
    "OrgOrdNo" : 0
   }
 }
```
### Response Example
```json
{
 "Out": {
  "OrdNo": 14
 },
 "rsp_cd": "00000",
 "rsp_msg": "매수 주문이 완료되었습니다."
}
```

# 해외주식 체결내역조회
- API 명: 해외주식 체결내역조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/trading/overseas-stock/inquiry/transaction-history
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 체결/미체결 내역을 조회하는 API입니다.<br><br> <br> ※ 모의투자 계좌로 사용가능한 API입니다. <br>※ 원화환산은 가장 최근 최초고시환율을 기준으로 계산합니다.<br>※ 체결내역이 전부 표시되지 않는경우 연속키 조회를 통해 확인하실 수 있습니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|QrySrtDt|조회시작일자|string|Y|8|"": 당일조회	기간조회시 YYYYMMDD 형식의 날짜 입력 (EX.20240101)|
|QryEndDt|조회종료일자|string|Y|8|"": 당일조회	기간조회시 YYYYMMDD 형식의 날짜 입력 (EX.20240102)|
|AstkIsuNo|해외주식종목번호|string|Y|20|미국주식/ETF: "종목코드" (EX. AAPL, SOXL, QQQ)	종목코드 미입력시 전체 종목 조회|
|AstkBnsTpCode|해외주식매매구분코드|string|Y|1|0.전체	1.매도	2.매수|
|OrdxctTpCode|주문체결구분코드|string|Y|1|0:전체	1:체결	2:미체결|
|StnlnTpCode|정렬구분코드|string|Y|1|0:역순	1:정순|
|QryTpCode|조회구분코드|string|Y|1|0:합산별	1:건별|
|OnlineYn|온라인여부|string|Y|1|0:전체	Y:온라인	N:오프라인|
|CvrgOrdYn|반대매매주문여부|string|Y|1|0:전체	Y:반대매매	N:일반주문|
|WonFcurrTpCode|원화외화구분코드|string|Y|1|1:원화	2:외화|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|array|Y|||
|OrdDt|주문일자|string|Y|8||
|OrdNo|주문번호|number|Y|10||
|ExecNo|체결번호|number|Y|10||
|OrgOrdNo|원주문번호|number|Y|10||
|AstkIsuNo|해외주식종목번호|string|Y|20||
|AstkHanglIsuNm|해외주식한글종목명|string|Y|200||
|SymCode|심볼코드|string|Y|12||
|OwnSeCode|자체증권거래소코드|string|Y|2||
|AstkSeNm|해외주식증권거래소명|string|Y|40||
|ShtnCntrySymCode|단축국가심볼코드|string|Y|2||
|CntryNm|국가명|string|Y|40||
|AstkBnsTpCode|해외주식매매구분코드|string|Y|1||
|OtptItemNm1|출력항목명1|string|Y|20||
|AstkOrdQty|해외주식주문수량|number|Y|28||
|AstkOrdPrc|해외주식주문가격|number|Y|28||
|AstkExecQty|해외주식체결수량|number|Y|28||
|AstkExecPrc|해외주식체결가격|number|Y|28||
|AstkOrdRmqty|해외주식주문잔량|number|Y|28||
|AstkExecAmt|해외주식체결금액|number|Y|28||
|AstkOrdStatCode|해외주식주문상태코드|string|Y|1||
|AstkBnsMdfyCode|해외주식매매정정코드|string|Y|1||
|OtptItemNm2|출력항목명2|string|Y|20||
|OrdTrdTpCode|주문거래구분코드|string|Y|1||
|OtptItemNm3|출력항목명3|string|Y|20||
|AstkOrdDttm|해외주식주문일시|string|Y|17||
|AstkLclOrdDttm|해외주식현지주문일시|string|Y|17||
|AstkExecDttm|해외주식체결일시|string|Y|17||
|AstkLclExecDttm|해외주식현지체결일시|string|Y|17||
|AstkOrdprcPtnCode|해외주식호가유형코드|string|Y|1||
|OtptItemNm4|출력항목명4|string|Y|20||
|AstkOrdCndiTpCode|해외주식주문조건구분코드|string|Y|1||
|OtptItemNm5|출력항목명5|string|Y|20||
|AstkMktCode|해외주식시장코드|string|Y|2||
|AstkMktNm|해외주식시장명|string|Y|100||
|CrcyCode|통화코드|string|Y|3||
|AstkRjtCode|해외주식거부코드|string|Y|10||
|AstkRjtRsnCnts|해외주식거부사유내용|string|Y|200||
|RsvOrdYn|예약주문여부|string|Y|1||
|RsvOrdDt|예약주문일자|string|Y|8||
|RsvOrdNo|예약주문번호|number|Y|10||
|AstkRsvOrdPtnCode|해외주식예약주문유형코드|string|Y|1||
|OtptItemNm6|출력항목명6|string|Y|20||
|LoanDt|대출일자|string|Y|8||
|LoanSeqno|대출일련번호|number|Y|6||
|WonAmt1|원화금액1|number|Y|16|원화환산 주문가격|
|WonAmt2|원화금액2|number|Y|16|원화환산 체결가격|
|WonAmt3|원화금액3|number|Y|16|원화환산 체결금액|
|Out1|Out1|array|Y|||
|ShtnCntrySymCode|단축국가심볼코드|string|Y|2||
|CntryNm|국가명|string|Y|40||
|CrcyCode|통화코드|string|Y|3||
|AstkSellOrdQty|해외주식매도주문수량|number|Y|28||
|AstkBuyOrdQty|해외주식매수주문수량|number|Y|28||
|AstkSellExecQty|해외주식매도체결수량|number|Y|28||
|AstkBuyExecQty|해외주식매수체결수량|number|Y|28||
## Example
### Request Example
```json
{
  "In": {
   "QrySrtDt": "20240101",
    "QryEndDt": "20240124",
  "AstkIsuNo": "",
  "AstkBnsTpCode": "0",
  "OrdxctTpCode": "0",
  "StnlnTpCode": "1",
    "QryTpCode": "0",
    "OnlineYn": "0",
  "WonFcurrTpCode": "2",
    "CvrgOrdYn": "0"
  }
}
```
### Response Example
```json
{
 "Out": [
  {
   "OrdDt": "20230105",
   "OrdNo": 1017,
   "ExecNo": 0,
   "OrgOrdNo": 0,
   "AstkIsuNo": "RBLX.US",
   "AstkHanglIsuNm": "로블록스",
   "SymCode": "RBLX",
   "OwnSeCode": "YS",
   "AstkSeNm": "뉴욕",
   "ShtnCntrySymCode": "US",
   "CntryNm": "미국",
   "AstkBnsTpCode": "2",
   "OtptItemNm1": "매수",
   "AstkOrdQty": "10.000000",
   "AstkOrdPrc": "38241.000000",
   "AstkExecQty": "10.000000",
   "AstkExecPrc": "38043.000000",
   "AstkOrdRmqty": "0.000000",
   "AstkExecAmt": "380434.000000",
   "AstkOrdStatCode": "7",
   "AstkBnsMdfyCode": "0",
   "OtptItemNm2": "체결",
   "OrdTrdTpCode": "0",
   "OtptItemNm3": "신규",
   "AstkOrdDttm": "20230106042819426",
   "AstkLclOrdDttm": "20230105142819942",
   "AstkExecDttm": "20230106042819697",
   "AstkLclExecDttm": "20230105142819969",
   "AstkOrdprcPtnCode": "1",
   "OtptItemNm4": "지정가",
   "AstkOrdCndiTpCode": "1",
   "OtptItemNm5": "일반",
   "AstkMktCode": "US",
   "AstkMktNm": "미국",
   "CrcyCode": "USD",
   "UserId": "son7665",
   "UserNm": "",
   "CommdaCode": "50",
   "MdaCodeNm": "해피+",
   "OnlineYn": "Y",
   "UserIpAddr": "112175250222",
   "AstkRjtCode": "",
   "AstkRjtRsnCnts": "",
   "RsvOrdYn": "N",
   "RsvOrdDt": "",
   "RsvOrdNo": 0,
   "AstkRsvOrdPtnCode": "",
   "OtptItemNm6": "",
   "LoanDt": "",
   "LoanSeqno": 0,
   "WonAmt1": 38241,
   "WonAmt2": 38043,
   "WonAmt3": 380434
  },
  {
   "OrdDt": "20230109",
   "OrdNo": 1586,
   "ExecNo": 0,
   "OrgOrdNo": 0,
   "AstkIsuNo": "RBLX.US",
   "AstkHanglIsuNm": "로블록스",
   "SymCode": "RBLX",
   "OwnSeCode": "YS",
   "AstkSeNm": "뉴욕",
   "ShtnCntrySymCode": "US",
   "CntryNm": "미국",
   "AstkBnsTpCode": "2",
   "OtptItemNm1": "매수",
   "AstkOrdQty": "10.000000",
   "AstkOrdPrc": "39384.000000",
   "AstkExecQty": "10.000000",
   "AstkExecPrc": "39017.000000",
   "AstkOrdRmqty": "0.000000",
   "AstkExecAmt": "390170.000000",
   "AstkOrdStatCode": "7",
   "AstkBnsMdfyCode": "0",
   "OtptItemNm2": "체결",
   "OrdTrdTpCode": "0",
   "OtptItemNm3": "신규",
   "AstkOrdDttm": "20230110050802469",
   "AstkLclOrdDttm": "20230109150802246",
   "AstkExecDttm": "20230110050802764",
   "AstkLclExecDttm": "20230109150802276",
   "AstkOrdprcPtnCode": "1",
   "OtptItemNm4": "지정가",
   "AstkOrdCndiTpCode": "1",
   "OtptItemNm5": "일반",
   "AstkMktCode": "US",
   "AstkMktNm": "미국",
   "CrcyCode": "USD",
   "UserId": "son7665",
   "UserNm": "",
   "CommdaCode": "50",
   "MdaCodeNm": "해피+",
   "OnlineYn": "Y",
   "UserIpAddr": "112175250218",
   "AstkRjtCode": "",
   "AstkRjtRsnCnts": "",
   "RsvOrdYn": "N",
   "RsvOrdDt": "",
   "RsvOrdNo": 0,
   "AstkRsvOrdPtnCode": "",
   "OtptItemNm6": "",
   "LoanDt": "",
   "LoanSeqno": 0,
   "WonAmt1": 39384,
   "WonAmt2": 39017,
   "WonAmt3": 390170
  }
 ],
 "Out1": [
  {
   "ShtnCntrySymCode": "US",
   "CntryNm": "미국",
   "CrcyCode": "USD",
   "AstkSellOrdQty": "149.000000",
   "AstkBuyOrdQty": "157.000000",
   "AstkSellExecQty": "149.000000",
   "AstkBuyExecQty": "149.000000"
  }
 ],
 "rsp_cd": "00000",
 "rsp_msg": "조회가 계속 됩니다. 계속하시려면 연속버튼을 누르십시오."
}
```

# 해외주식 잔고_증거금 조회
- API 명: 해외주식 잔고_증거금 조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/trading/overseas-stock/inquiry/balance-margin
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 잔고조회 API 입니다. 보유중인 주식/증거금 잔고에 대한 정보를 제공합니다.<br><br> <br> ※ 모의투자 계좌로 사용가능한 API입니다. <br> ※ 잔고가 전부 표시되지 않는경우 연속키 조회를 통해 확인하실 수 있습니다.<br>※ 원화환산은 가장 최근 최초고시환율을 기준으로 계산합니다.<br>※ 예수금, 잔고평가, 출금가능 등의 금액은 추정치로 실제와 차이가 있을 수 있습니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|TrxTpCode|처리구분코드|string|Y|1|1:외화잔고	2:주식잔고상세	3:주식잔고(국가별)	9:당일실현손익|
|CmsnTpCode|수수료구분코드|string|Y|1|0:전부 미포함	1:매수제비용만 포함 	2:매수제비용+매도제비용|
|WonFcurrTpCode|원화외화구분코드|string|Y|1|1:원화 	2:외화|
|DpntBalTpCode|소수점잔고구분코드|string|Y|1|0: 전체	1: 일반	2: 소수점|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|object|Y|||
|Dps|예수금|number|Y|16||
|OrdAbleAmt|주문가능금액|number|Y|16||
|MnyoutAbleAmt|출금가능금액|number|Y|16||
|BalEvalAmt|잔고평가금액|number|Y|16||
|EvalPnlAmt|평가손익금액|number|Y|16||
|ErnRat|수익율|number|Y|18||
|PchsAmt|매입금액|number|Y|16||
|Cmsn|수수료|number|Y|16||
|TaxAmt|세금금액|number|Y|16||
|ThdayRlzPnlAmt|당일실현손익금액|number|Y|16||
|AssetAmtTotamt|자산금액총액|number|Y|16||
|UnsttAmt|미결제금액|number|Y|16||
|Out1|Out1|array|Y|||
|CrcyCode|통화코드|string|Y|3||
|CrcyCodeNm|통화코드명|string|Y|50||
|FcurrDps|외화예수금|number|Y|21||
|AstkEstiDps|해외주식추정예수금|number|Y|28||
|AstkMgn|해외주식증거금|number|Y|28||
|AstkReBuyAbleAmt|해외주식재매수가능금액|number|Y|28||
|AstkOrdAbleAmt|해외주식주문가능금액|number|Y|28||
|AstkMnyoutAbleAmt|해외주식출금가능금액|number|Y|28||
|AstkUnsttSellAmt|해외주식미결제매도금액|number|Y|28||
|AstkUnsttBuyAmt|해외주식미결제매수금액|number|Y|28||
|AstkEvalAmt|해외주식평가금액|number|Y|28||
|AstkAssetEvalAmt|해외주식자산평가금액|number|Y|28||
|WonEvalAmt|원화평가금액|number|Y|16||
|AvrXchrat|평균환율|number|Y|15||
|Xchrat|환율|number|Y|15||
|FcurrRcvblAmt|외화미수금액|number|Y|21||
|AstkOrdAbleAmt0|해외주식주문가능금액0|number|Y|28|통합증거금주문가능금액|
|FcurrOthrCrcyMgn|외화타통화증거금|number|Y|28||
|OthrCrcyMgnWtAmt|타통화증거금원화환산금액|number|Y|16||
|Out2|Out2|array|Y|||
|AstkIsuNo|해외주식종목번호|string|Y|20||
|AstkHanglIsuNm|해외주식한글종목명|string|Y|200||
|SymCode|심볼코드|string|Y|12||
|OwnSeCode|자체증권거래소코드|string|Y|2||
|AstkSeNm|해외주식증권거래소명|string|Y|40||
|ShtnCntrySymCode|단축국가심볼코드|string|Y|2||
|CntryNm|국가명|string|Y|40||
|CrcyCode|통화코드|string|Y|3||
|AstkMktCode|해외주식시장코드|string|Y|2||
|AstkMktNm|해외주식시장명|string|Y|100||
|AstkExecBaseQty|해외주식체결기준수량|number|Y|28||
|AstkSettBaseQty|해외주식결제기준수량|number|Y|28||
|AstkRopSetupQty|해외주식질권설정수량|number|Y|28||
|AstkOrdAbleQty|해외주식주문가능수량|number|Y|28||
|AstkAvrPchsPrc|해외주식평균매입가|number|Y|28||
|WonAmt1|원화금액1|number|Y|16|원화환산평균매입가|
|AstkNowPrc|해외주식현재가|number|Y|28||
|WonAmt2|원화금액2|number|Y|16|원화환산현재가|
|AstkBuyAmt|해외주식매수금액|number|Y|28||
|WonAmt3|원화금액3|number|Y|16|원화환산매수금액|
|AstkEvalAmt|해외주식평가금액|number|Y|28||
|WonAmt4|원화금액4|number|Y|16|원화환산평가금액|
|AstkEvalPnlAmt|해외주식평가손익금액|number|Y|28||
|WonAmt5|원화금액5|number|Y|16|원화환산평가손익|
|EvalPnlRat|평가손익율|number|Y|9||
|EvalPnlRat0|평가손익율0|number|Y|9|원화환산손익율|
|AstkCmsn|해외주식수수료|number|Y|28||
|WonAmt6|원화금액6|number|Y|16|원화환산수수료|
|AstkTaxAmt|해외주식세금금액|number|Y|28||
|WonAmt7|원화금액7|number|Y|16|원화환산세금금액|
|AstkPrdayCmpPrc|해외주식전일대비가격|number|Y|28||
|WonAmt8|원화금액8|number|Y|16|원화환산전일대비가격|
|AstkUpdnRat|해외주식등락율|number|Y|28||
|HldWeght|보유비중|number|Y|18||
|TpVal|구분값|string|Y|40|현금/대출|
|DueDt|만기일자|string|Y|8||
|LoanDt|대출일자|string|Y|8||
|LoanAmt|대출금액|number|Y|16||
|AstkBuyCmsn|해외주식매수수수료|number|Y|28||
|WonAmt9|원화금액9|number|Y|16|원화환산매수수수료|
|AstkSellCmsn|해외주식매도수수료|number|Y|28||
|WonAmt0|원화금액0|number|Y|16|원화환산매도수수료|
|AstkCmsnRat|해외주식수수료율|number|Y|28|고객수수료율|
|AstkCmsnRat0|해외주식수수료율0|number|Y|28|현지매도수수료율|
|ThdayRlzPnlAmt|당일실현손익금액|number|Y|16||
|AstkSellExecQty|해외주식매도체결수량|number|Y|28|당일매도수량|
|AstkSellExecAmt|해외주식매도체결금액|number|Y|28|당일매도금액|
|AstkExecBaseBuyCost|해외주식체결기준매수비용|number|Y|28|당일매수수수료|
|AstkCmsnRat1|해외주식수수료율1|number|Y|28|현지매수수수료율|
|Out3|Out3|array|Y|||
|ShtnCntrySymCode|단축국가심볼코드|string|Y|2||
|CntryNm|국가명|string|Y|40||
|CrcyCode|통화코드|string|Y|3||
|AstkBuyAmt|해외주식매수금액|number|Y|28||
|WonAmt1|원화금액1|number|Y|16|원화환산매수금액|
|AstkEvalAmt|해외주식평가금액|number|Y|28||
|WonAmt2|원화금액2|number|Y|16|원화환산평가금액|
|AstkEvalPnlAmt|해외주식평가손익금액|number|Y|28||
|WonAmt3|원화금액3|number|Y|16|원화환산평가손익금액|
|EvalPnlRat|평가손익율|number|Y|9||
|EvalPnlRat0|평가손익율0|number|Y|9||
|AstkCmsn|해외주식수수료|number|Y|28||
|WonAmt4|원화금액4|number|Y|16|원화환산수수료|
|AstkTaxAmt|해외주식세금금액|number|Y|28||
|WonAmt5|원화금액5|number|Y|16|원화환산세금금액|
|AstkBuyCmsn|해외주식매수수수료|number|Y|28||
|WonAmt6|원화금액6|number|Y|16|원화환산매수수수료|
|AstkSellCmsn|해외주식매도수수료|number|Y|28||
|WonAmt7|원화금액7|number|Y|16|원화환산매도수수료|
## Example
### Request Example
```json
{
  "In": {
    "WonFcurrTpCode": "2",
    "TrxTpCode": "2",
    "CmsnTpCode": "2",
    "DpntBalTpCode": "1"
  }
}
```
### Response Example
```json
{
 "Out": {
  "Dps": 1157670213,
  "OrdAbleAmt": 0,
  "MnyoutAbleAmt": 871557533,
  "BalEvalAmt": 5974845,
  "EvalPnlAmt": -2783345,
  "ErnRat": "-31.800000",
  "PchsAmt": 8752085,
  "Cmsn": 6065,
  "TaxAmt": 40,
  "ThdayRlzPnlAmt": 0,
  "AssetAmtTotamt": 1163597777,
  "UnsttAmt": -47281
 },
 "Out1": [],
 "Out2": [
  {
   "AstkIsuNo": "T.US",
   "AstkHanglIsuNm": "AT&T",
   "SymCode": "T",
   "OwnSeCode": "YS",
   "AstkSeNm": "뉴욕",
   "ShtnCntrySymCode": "US",
   "CntryNm": "미국",
   "CrcyCode": "USD",
   "AstkMktCode": "US",
   "AstkMktNm": "미국",
   "AstkExecBaseQty": "208.000000",
   "AstkSettBaseQty": "208.000000",
   "AstkRopSetupQty": "0.000000",
   "AstkOrdAbleQty": "208.000000",
   "AstkAvrPchsPrc": "28.613519",
   "WonAmt1": 38227,
   "AstkNowPrc": "16.680000",
   "WonAmt2": 22284,
   "AstkBuyAmt": "5951.612000",
   "WonAmt3": 7951353,
   "AstkEvalAmt": "3469.440000",
   "WonAmt4": 4635171,
   "AstkEvalPnlAmt": "-2485.362000",
   "WonAmt5": -3320443,
   "EvalPnlRat": "-41.75",
   "EvalPnlRat0": "-41.75",
   "AstkCmsn": "3.160000",
   "WonAmt6": 4221,
   "AstkTaxAmt": "0.030000",
   "WonAmt7": 40,
   "AstkPrdayCmpPrc": "-0.510000",
   "WonAmt8": -681,
   "AstkUpdnRat": "-2.970000",
   "HldWeght": "77.570000",
   "TpVal": "현금",
   "DueDt": "00000000",
   "LoanDt": "00000000",
   "LoanAmt": 0,
   "AstkBuyCmsn": "0.390000",
   "WonAmt9": 521,
   "AstkSellCmsn": "2.770000",
   "WonAmt0": 3700,
   "AstkCmsnRat": "0.08000000000",
   "AstkCmsnRat0": "0.00080000000",
   "ThdayRlzPnlAmt": 0,
   "AstkSellExecQty": "0.000000",
   "AstkSellExecAmt": "0.000000",
   "AstkExecBaseBuyCost": "0.000000",
   "AstkCmsnRat1": "0.00000000000"
  },
  {
   "AstkIsuNo": "TLT.US",
   "AstkHanglIsuNm": "ISHARES 20+Y TREASURY BOND ETF",
   "SymCode": "TLT",
   "OwnSeCode": "NS",
   "AstkSeNm": "나스닥",
   "ShtnCntrySymCode": "US",
   "CntryNm": "미국",
   "CrcyCode": "USD",
   "AstkMktCode": "US",
   "AstkMktNm": "미국",
   "AstkExecBaseQty": "5.000000",
   "AstkSettBaseQty": "5.000000",
   "AstkRopSetupQty": "0.000000",
   "AstkOrdAbleQty": "5.000000",
   "AstkAvrPchsPrc": "103.590000",
   "WonAmt1": 138396,
   "AstkNowPrc": "93.350000",
   "WonAmt2": 124715,
   "AstkBuyAmt": "517.950000",
   "WonAmt3": 691981,
   "AstkEvalAmt": "466.750000",
   "WonAmt4": 623578,
   "AstkEvalPnlAmt": "-52.090000",
   "WonAmt5": -69591,
   "EvalPnlRat": "-10.05",
   "EvalPnlRat0": "-10.05",
   "AstkCmsn": "0.890000",
   "WonAmt6": 1188,
   "AstkTaxAmt": "0.000000",
   "WonAmt7": 0,
   "AstkPrdayCmpPrc": "-0.550000",
   "WonAmt8": -734,
   "AstkUpdnRat": "-0.590000",
   "HldWeght": "10.430000",
   "TpVal": "현금",
   "DueDt": "00000000",
   "LoanDt": "00000000",
   "LoanAmt": 0,
   "AstkBuyCmsn": "0.520000",
   "WonAmt9": 694,
   "AstkSellCmsn": "0.370000",
   "WonAmt0": 494,
   "AstkCmsnRat": "0.08000000000",
   "AstkCmsnRat0": "0.00080000000",
   "ThdayRlzPnlAmt": 0,
   "AstkSellExecQty": "0.000000",
   "AstkSellExecAmt": "0.000000",
   "AstkExecBaseBuyCost": "0.000000",
   "AstkCmsnRat1": "0.00000000000"
  },
  {
   "AstkIsuNo": "TMF.US",
   "AstkHanglIsuNm": "DIREXION DAILY 20Y TREASURY BULL 3X ETF",
   "SymCode": "TMF",
   "OwnSeCode": "AS",
   "AstkSeNm": "아멕스",
   "ShtnCntrySymCode": "US",
   "CntryNm": "미국",
   "CrcyCode": "USD",
   "AstkMktCode": "US",
   "AstkMktNm": "미국",
   "AstkExecBaseQty": "10.000000",
   "AstkSettBaseQty": "10.000000",
   "AstkRopSetupQty": "0.000000",
   "AstkOrdAbleQty": "10.000000",
   "AstkAvrPchsPrc": "8.140000",
   "WonAmt1": 10875,
   "AstkNowPrc": "53.600000",
   "WonAmt2": 71609,
   "AstkBuyAmt": "81.400000",
   "WonAmt3": 108750,
   "AstkEvalAmt": "536.000000",
   "WonAmt4": 716096,
   "AstkEvalPnlAmt": "454.110000",
   "WonAmt5": 606692,
   "EvalPnlRat": "557.87",
   "EvalPnlRat0": "557.87",
   "AstkCmsn": "0.490000",
   "WonAmt6": 654,
   "AstkTaxAmt": "0.000000",
   "WonAmt7": 0,
   "AstkPrdayCmpPrc": "-0.940000",
   "WonAmt8": -1255,
   "AstkUpdnRat": "-1.720000",
   "HldWeght": "11.980000",
   "TpVal": "현금",
   "DueDt": "00000000",
   "LoanDt": "00000000",
   "LoanAmt": 0,
   "AstkBuyCmsn": "0.070000",
   "WonAmt9": 93,
   "AstkSellCmsn": "0.420000",
   "WonAmt0": 561,
   "AstkCmsnRat": "0.08000000000",
   "AstkCmsnRat0": "0.00080000000",
   "ThdayRlzPnlAmt": 0,
   "AstkSellExecQty": "0.000000",
   "AstkSellExecAmt": "0.000000",
   "AstkExecBaseBuyCost": "0.000000",
   "AstkCmsnRat1": "0.00000000000"
  }
 ],
 "Out3": [],
 "rsp_cd": "00000",
 "rsp_msg": "조회가 완료되었습니다."
}
```

# 해외주식 매매내역 조회
- API 명: 해외주식 매매내역 조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/trading/overseas-stock/inquiry/trading-history
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 매매내역을 조회하는 API입니다.<br><br> <br> ※ 모의투자 계좌로 사용가능한 API입니다. <br>※ 원화환산은 가장 최근 최초고시환율을 기준으로 계산합니다.<br>※ 매매내역이 전부 표시되지 않는경우 연속키 조회를 통해 확인하실 수 있습니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|QrySrtDt|조회시작일자|string|Y|8|YYYYMMDD EX.20240101|
|QryEndDt|조회종료일자|string|Y|8|YYYYMMDD EX.20240105|
|AstkIsuNo|해외주식종목번호|string|Y|20|미국주식/ETF: "종목코드" (EX. AAPL, SOXL, QQQ)|
|AstkBnsTpCode|해외주식매매구분코드|string|Y|1|0:전체	1:매도	2:매수|
|StnlnTpCode|정렬구분코드|string|Y|1|0:역순	1:정순|
|QryTpCode|조회구분코드|string|Y|1|0:합산별	1:건별|
|WonFcurrTpCode|원화외화구분코드|string|Y|1|1:원화	2:외화|
|BaseDdTpCode|기준일구분코드|string|Y|1|1:매매일	2:결제일|
|DpntBalTpCode|소수점잔고구분코드|string|Y|1|0:전체	1:일반	2:소수점|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|array|Y|||
|OrdDt|주문일자|string|Y|8||
|OrdNo|주문번호|number|Y|10||
|AstkMktCode|해외주식시장코드|string|Y|2||
|AstkMktNm|해외주식시장명|string|Y|100||
|AstkIsuNo|해외주식종목번호|string|Y|20||
|AstkHanglIsuNm|해외주식한글종목명|string|Y|200||
|SymCode|심볼코드|string|Y|12||
|OwnSeCode|자체증권거래소코드|string|Y|2||
|AstkSeNm|해외주식증권거래소명|string|Y|40||
|ShtnCntrySymCode|단축국가심볼코드|string|Y|2||
|AstkBnsTpCode|해외주식매매구분코드|string|Y|1||
|BnsTpNm|매매구분명|string|Y|4||
|AstkExecQty|해외주식체결수량|number|Y|28||
|AstkExecPrc|해외주식체결가격|number|Y|28||
|AstkExecAmt|해외주식체결금액|number|Y|28||
|AstkExecCmsn|해외주식체결수수료|number|Y|28||
|AstkExecLclCmsn|해외주식체결현지수수료|number|Y|28||
|AstkSettAmt|해외주식결제금액|number|Y|28||
|CrcyCode|통화코드|string|Y|3||
|SettDt|결제일자|string|Y|8|매도시 유가증권결제일자 매수시 현금결제일자|
|MnySettDt|현금결제일자|string|Y|8||
|SecSettDt|유가증권결제일자|string|Y|8||
|RsvOrdDt|예약주문일자|string|Y|8||
|RsvOrdNo|예약주문번호|number|Y|10||
|AstkRsvOrdPtnCode|해외주식예약주문유형코드|string|Y|1||
|OtptItemNm|출력항목명|string|Y|20||
|LoanDt|대출일자|string|Y|8||
|DpntBalTpNm|소수점잔고구분명|string|Y|100||
|Out1|Out1|object|Y|||
|AdjstAmt|정산금액|number|Y|16||
|ExecAmt|체결금액|number|Y|16||
|Cmsn|수수료|number|Y|16||
|TaxAmt|세금금액|number|Y|16||

# 해외주식 거래내역 조회
- API 명: 해외주식 거래내역 조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/trading/overseas-stock/inquiry/trade-history
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 계좌의 거래내역을 조회하는 API입니다.<br><br> <br> ※ 모의투자 계좌로 사용가능한 API입니다. <br>※ 거래내역이 전부 표시되지 않는경우 연속키 조회를 통해 확인하실 수 있습니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|QrySrtDt|조회시작일자|string|Y|8|YYYYMMDD EX.20240101|
|QryEndDt|조회종료일자|string|Y|8|YYYYMMDD EX.20240110|
|StnlnTpCode|정렬구분코드|string|Y|1|0:역순	1:정순|
|AstkIsuNo|해외주식종목번호|string|Y|20|미국주식/ETF: "종목코드" (EX. AAPL, SOXL, QQQ)|
|QryTpCode|조회구분코드|string|Y|1|0:전체	1:입출금	2:입출고	3:매매|
|DpntBalTpCode|소수점잔고구분코드|string|Y|1|0: 전체	1: 일반	2: 소수점|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|array|Y|||
|TrdDt|거래일자|string|Y|8||
|TrdTime|거래시각|string|Y|9||
|SmryNm|적요명|string|Y|40||
|AstkIsuNo|해외주식종목번호|string|Y|20||
|AstkEngIsuNm|해외주식영문종목명|string|Y|200||
|AstkHanglIsuNm|해외주식한글종목명|string|Y|200||
|AstkMktCode|해외주식시장코드|string|Y|2||
|SymCode|심볼코드|string|Y|12||
|OwnSeCode|자체증권거래소코드|string|Y|2||
|AstkSeNm|해외주식증권거래소명|string|Y|40||
|CrcyCode|통화코드|string|Y|3||
|AstkTrdQty|해외주식거래수량|number|Y|28||
|AstkBnsAmt|해외주식매매금액|number|Y|28||
|AstkTrdUprc|해외주식거래단가|number|Y|28||
|AstkCmsn|해외주식수수료|number|Y|28||
|AstkTaxAmt|해외주식세금금액|number|Y|28||
|AstkCrbalQty|해외주식금잔수량|number|Y|28||
|AstkAppXchrat|해외주식적용환율|number|Y|30||
|FcurrTrdAmt|외화거래금액|number|Y|24||
|FcurrDpsCrbalAmt|외화예수금금잔금액|number|Y|21||
|TrdAmt|거래금액|number|Y|16||
|DpsCrbalAmt|예수금금잔금액|number|Y|16||
|FcurrRcvblRepayAmt|외화미수변제금액|number|Y|21||
|FcurrRcvblOcrAmt|외화미수발생금액|number|Y|21||
|FcurrRcvblAmt|외화미수금액|number|Y|21||
|FcurrOdpnt|외화연체료|number|Y|21||
|OppAcntNo|상대계좌번호|string|Y|20||
|OppAcntNm|상대계좌명|string|Y|40||
|OthbkpOppCoNo|타사대체상대회사번호|string|Y|10||
|BkeepIttNm|대체기관명|string|Y|40||

# 해외주식 주문가능금액조회
- API 명: 해외주식 주문가능금액조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/trading/overseas-stock/inquiry/able-orderqty
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 주문가능금액조회 API입니다. <br> ※ 모의투자 계좌로 사용가능한 API입니다. <br>
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|TrxTpCode|처리구분코드|string|Y|1|1:매도	2:매수|
|AstkIsuNo|해외주식종목번호|string|Y|20|미국주식/ETF: "종목코드" (EX. AAPL, SOXL, QQQ)|
|AstkOrdPrc|해외주식주문가격|number|Y|28||
|WonFcurrTpCode|원화외화구분코드|string|Y|1|1: 원화	2: 외화|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|object|Y|||
|AstkOrdAbleAmt|해외주식주문가능금액|number|Y|28|해당통화 기준|
|AstkOrdAbleQty|해외주식주문가능수량|number|Y|28|해당통화 기준|
|AstkOrdAbleAmt0|해외주식주문가능금액0|number|Y|28|통함증거금 기준|
|AstkOrdAbleQty0|해외주식주문가능수량0|number|Y|28|통함증거금 기준|
|AstkOrdAbleAmt1|해외주식주문가능금액1|number|Y|28|미수금포함 기준|
|AstkOrdAbleQty1|해외주식주문가능수량1|number|Y|28|미수금포함 기준|
|CsldtMgnUseYn|통합증거금사용여부|string|Y|1|(Y/N)|
|OtptItemNm0|출력항목명0|string|Y|20|통합증거금사용여부 (Y/N)|
|Mgnrt|증거금율|number|Y|26|종목증거금률|
|OtptItemNm1|출력항목명1|string|Y|20|계좌증거금률|
|Mgnrt0|증거금율0|number|Y|26|적용증거금률|

# 해외주식 실현손익 조회
- API 명: 해외주식 실현손익 조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/trading/overseas-stock/inquiry/day-rlzpnl
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 실현손익을 조회하는 API입니다.<br><br> <br> ※ 모의투자 계좌로 사용가능한 API입니다. <br>※ 원화환산은 가장 최근 최초고시환율을 기준으로 계산합니다.<br>※ 실현손익내역이 전부 표시되지 않는경우 연속키 조회를 통해 확인하실 수 있습니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|TrxTpCode|처리구분코드|string|Y|1|1:국가별 및 일별국가별 실현손익	2:일별국가별종목별 실현손익|
|QrySrtDt|조회시작일자|string|Y|8|YYYYMMDD EX.20240101|
|QryEndDt|조회종료일자|string|Y|8|YYYYMMDD EX.20240105|
|AstkIsuNo|해외주식종목번호|string|Y|20|미국주식/ETF: "종목코드" (EX. AAPL, SOXL, QQQ)|
|WonFcurrTpCode|원화외화구분코드|string|Y|1|1:원화	2:외화|
|EvrprcYn|제비용여부|string|Y|1|Y:예(기본)	N:아니요|
|DpntBalTpCode|소수점잔고구분코드|string|Y|1|0: 전체	1: 일반	2: 소수점|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out2|Out2|array|Y|||
|AstkMktCode|해외주식시장코드|string|Y|2||
|AstkMktNm|해외주식시장명|string|Y|100||
|OrdDt|주문일자|string|Y|8||
|CrcyCode|통화코드|string|Y|3||
|AstkIsuNo|해외주식종목번호|string|Y|20||
|AstkHanglIsuNm|해외주식한글종목명|string|Y|200||
|SymCode|심볼코드|string|Y|12||
|OwnSeCode|자체증권거래소코드|string|Y|2||
|AstkSeNm|해외주식증권거래소명|string|Y|40||
|ShtnCntrySymCode|단축국가심볼코드|string|Y|2||
|AstkBuyExecQty|해외주식매수체결수량|number|Y|28|사용안함|
|AstkBuyExecAmt|해외주식매수체결금액|number|Y|28|사용안함|
|AstkAvrPchsPrc|해외주식평균매입가|number|Y|28||
|AstkBuyExecCmsn|해외주식매수체결수수료|number|Y|28|사용안함|
|AstkSellExecQty|해외주식매도체결수량|number|Y|28||
|AstkSellExecAmt|해외주식매도체결금액|number|Y|28||
|AstkSellAvrPrc|해외주식매도평균가|number|Y|28||
|AstkSellExecCmsn|해외주식매도체결수수료|number|Y|28||
|AstkExecBaseBuyAmt|해외주식체결기준매수금액|number|Y|28||
|AstkExecBaseBuyCost|해외주식체결기준매수비용|number|Y|28||
|AstkTaxAmt|해외주식세금금액|number|Y|28||
|AstkCmsn|해외주식수수료|number|Y|28||
|AstkCost|해외주식비용|number|Y|28||
|AstkBnsplAmt|해외주식매매손익금액|number|Y|28||
|PnlRat|손익율|number|Y|18||
|BnsplWonAmt|매매손익원화금액|number|Y|16||
|EvalXchrat|평가환율|number|Y|15|매도체결 시점의 적용환율|
|LoanDt|대출일자|string|Y|8||
|Out3|Out3|object|Y|||
|BnsplAmt|매매손익금액|number|Y|16||
|BnsAmt|매매금액|number|Y|16||
|SellAmt|매도금액|number|Y|16||
|BuyAmt|매수금액|number|Y|16||
|CostAmt|비용금액|number|Y|16||
|Cmsn|수수료|number|Y|16||
|TaxAmt|세금금액|number|Y|16||
|PnlRat|손익율|number|Y|18||

# 해외주식 예수금상세
- API 명: 해외주식 예수금상세
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/trading/overseas-stock/inquiry/deposit-detail
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 상세 예수금을 조회하는 API입니다.<br><br> <br> ※ 모의투자 계좌로 사용가능한 API입니다. <br>※ 원화환산은 가장 최근 최초고시환율을 기준으로 계산합니다.<br>※ 예수금, 주문가능, 출금가능, 평가자산총액 등의 금액은 추정치로 실제와 차이가 있을 수 있습니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|||||||
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|object|Y|||
|OtptItemNm|출력항목명|string|Y|20||
|DpsBaseDt0|예수금기준일자0|string|Y|8|당일기준|
|DpsBaseDt1|예수금기준일자1|string|Y|8|D+1|
|DpsBaseDt2|예수금기준일자2|string|Y|8|D+2|
|DpsBaseDt3|예수금기준일자3|string|Y|8|D+3|
|DpsBaseDt4|예수금기준일자4|string|Y|8|D+4|
|Out1|Out1|array|Y|||
|CrcyCode|통화코드|string|Y|3||
|CrcyCodeNm|통화코드명|string|Y|50||
|OtptItemNm0|출력항목명0|string|Y|20||
|OtptItemNm1|출력항목명1|string|Y|20||
|OtptItemNm2|출력항목명2|string|Y|20||
|OtptItemNm3|출력항목명3|string|Y|20||
|AstkDps0|해외주식예수금0|number|Y|28||
|AstkBnsAmt0|해외주식매매금액0|number|Y|28||
|AstkUnsttBuyAmt0|해외주식미결제매수금액0|number|Y|28||
|AstkUnsttSellAmt0|해외주식미결제매도금액0|number|Y|28||
|AstkDps1|해외주식예수금1|number|Y|28||
|AstkBnsAmt1|해외주식매매금액1|number|Y|28||
|AstkUnsttBuyAmt1|해외주식미결제매수금액1|number|Y|28||
|AstkUnsttSellAmt1|해외주식미결제매도금액1|number|Y|28||
|AstkDps2|해외주식예수금2|number|Y|28||
|AstkBnsAmt2|해외주식매매금액2|number|Y|28||
|AstkUnsttBuyAmt2|해외주식미결제매수금액2|number|Y|28||
|AstkUnsttSellAmt2|해외주식미결제매도금액2|number|Y|28||
|AstkDps3|해외주식예수금3|number|Y|28||
|AstkBnsAmt3|해외주식매매금액3|number|Y|28||
|AstkUnsttBuyAmt3|해외주식미결제매수금액3|number|Y|28||
|AstkUnsttSellAmt3|해외주식미결제매도금액3|number|Y|28||
|AstkDps4|해외주식예수금4|number|Y|28||
|AstkBnsAmt4|해외주식매매금액4|number|Y|28||
|AstkUnsttBuyAmt4|해외주식미결제매수금액4|number|Y|28||
|AstkUnsttSellAmt4|해외주식미결제매도금액4|number|Y|28||
## Example
### Request Example
```json
{
  "In": {
  }
}
```
### Response Example
```json
{
 "Out": {
  "OtptItemNm": "결제일자",
  "DpsBaseDt0": "20240124",
  "DpsBaseDt1": "20240125",
  "DpsBaseDt2": "20240126",
  "DpsBaseDt3": "20240129",
  "DpsBaseDt4": "20240130"
 },
 "Out1": [
  {
   "CrcyCode": "KRW",
   "CrcyCodeNm": "원화",
   "OtptItemNm0": "예수금",
   "OtptItemNm1": "매매결제",
   "OtptItemNm2": "매수결제",
   "OtptItemNm3": "매도결제",
   "AstkDps0": "348555423.000000",
   "AstkBnsAmt0": "0.000000",
   "AstkUnsttBuyAmt0": "0.000000",
   "AstkUnsttSellAmt0": "0.000000",
   "AstkDps1": "345649473.000000",
   "AstkBnsAmt1": "-2905950.000000",
   "AstkUnsttBuyAmt1": "2905950.000000",
   "AstkUnsttSellAmt1": "0.000000",
   "AstkDps2": "345649473.000000",
   "AstkBnsAmt2": "0.000000",
   "AstkUnsttBuyAmt2": "0.000000",
   "AstkUnsttSellAmt2": "0.000000",
   "AstkDps3": "0.000000",
   "AstkBnsAmt3": "0.000000",
   "AstkUnsttBuyAmt3": "0.000000",
   "AstkUnsttSellAmt3": "0.000000",
   "AstkDps4": "0.000000",
   "AstkBnsAmt4": "0.000000",
   "AstkUnsttBuyAmt4": "0.000000",
   "AstkUnsttSellAmt4": "0.000000"
  },
  {
   "CrcyCode": "USD",
   "CrcyCodeNm": "미국 달러",
   "OtptItemNm0": "예수금",
   "OtptItemNm1": "매매결제",
   "OtptItemNm2": "매수결제",
   "OtptItemNm3": "매도결제",
   "AstkDps0": "605080.300000",
   "AstkBnsAmt0": "0.000000",
   "AstkUnsttBuyAmt0": "0.000000",
   "AstkUnsttSellAmt0": "0.000000",
   "AstkDps1": "605080.300000",
   "AstkBnsAmt1": "0.000000",
   "AstkUnsttBuyAmt1": "0.000000",
   "AstkUnsttSellAmt1": "0.000000",
   "AstkDps2": "605080.300000",
   "AstkBnsAmt2": "0.000000",
   "AstkUnsttBuyAmt2": "0.000000",
   "AstkUnsttSellAmt2": "0.000000",
   "AstkDps3": "605080.300000",
   "AstkBnsAmt3": "0.000000",
   "AstkUnsttBuyAmt3": "0.000000",
   "AstkUnsttSellAmt3": "0.000000",
   "AstkDps4": "605080.300000",
   "AstkBnsAmt4": "0.000000",
   "AstkUnsttBuyAmt4": "0.000000",
   "AstkUnsttSellAmt4": "0.000000"
  }
 ],
 "rsp_cd": "00000",
 "rsp_msg": "조회가 완료되었습니다."
}
```

# 해외주식 평균매입단가 조회
- API 명: 해외주식 평균매입단가 조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/trading/overseas-stock/inquiry/avg-pur-price
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
보유중인 해외주식 종목의 평균 매입단가를 조회할 수 있는 API입니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|AstkIsuNo|해외주식종목번호|string|Y|20|미국주식/ETF: "종목코드" (EX. AAPL, SOXL, QQQ)|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|object|Y|||
|AstkAvrPchsPrc|해외주식평균매입가|number|Y|28||
|AstkMktCode|해외주식시장코드|string|Y|2||
|OwnSeCode|자체증권거래소코드|string|Y|2||
