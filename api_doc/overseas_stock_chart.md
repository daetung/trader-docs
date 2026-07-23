# API 목록

|순번|서비스 명|API 명|HTTP Method|URL|Domain|포맷|TPS|
|---|---|---|---|---|---|---|---|
|1|해외주식시세|해외주식종목 조회|POST|/api/v1/quote/overseas-stock/inquiry/stock-ticker|https://openapi.dbsec.co.kr:8443|JSON|2|
|2|해외주식시세|해외주식 멀티현재가조회|POST|/api/v1/quote/overseas-stock/inquiry/multiprice|https://openapi.dbsec.co.kr:8443|JSON|2|
|3|해외주식시세|해외주식현재가조회|POST|/api/v1/quote/overseas-stock/inquiry/price|https://openapi.dbsec.co.kr:8443|JSON|2|
|4|해외주식시세|해외주식호가조회|POST|/api/v1/quote/overseas-stock/inquiry/orderbook|https://openapi.dbsec.co.kr:8443|JSON|2|
|5|해외주식시세|해외주식시간대별체결조회|POST|/api/v1/quote/overseas-stock/inquiry/hour-price|https://openapi.dbsec.co.kr:8443|JSON|2|
|6|해외주식시세|해외주식 틱차트조회|POST|/api/v1/quote/overseas-stock/chart/tick|https://openapi.dbsec.co.kr:8443|JSON|4|
|7|해외주식시세|해외주식 분차트조회|POST|/api/v1/quote/overseas-stock/chart/min|https://openapi.dbsec.co.kr:8443|JSON|4|
|8|해외주식시세|해외주식 일차트조회|POST|/api/v1/quote/overseas-stock/chart/day|https://openapi.dbsec.co.kr:8443|JSON|4|
|9|해외주식시세|해외주식 주차트조회|POST|/api/v1/quote/overseas-stock/chart/week|https://openapi.dbsec.co.kr:8443|JSON|4|
|10|해외주식시세|해외주식 월차트조회|POST|/api/v1/quote/overseas-stock/chart/month|https://openapi.dbsec.co.kr:8443|JSON|4|
|11|해외주식시세|해외주식 상승하락조회|POST|/api/v1/quote/overseas-stock/inquiry/rank-list|https://openapi.dbsec.co.kr:8443|JSON|2|

# 해외주식종목 조회
- API 명: 해외주식종목 조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/inquiry/stock-ticker
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 종목 조회 API 입니다.
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
|InputDataCode|입력해외증시구분코드|string|Y|2|NY: 뉴욕	NA: 나스닥	AM: 아멕스|
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
|Iscd|종목코드|string|Y|9||
|KorIsnm|한글종목명|string|Y|40||
|BstpLargName|업종대분류명|string|Y|40||
|ExchClsCode2|거래소코드2|string|Y|4||
|SelnVolUnit|매도량단위|string|Y|9||
|ShnuVolUnit|매수량단위|string|Y|9||
## Example
### Request Example
```json
{
	"In": {
		"InputDataCode": "NA"
	}
}
```
### Response Example
```json
{
	"Out": [
		{
			"Iscd": "AACG",
			"KorIsnm": "ATA 크리에티비티 글로벌(ADR)",
			"BstpLargName": "경기 소비재",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AACI",
			"KorIsnm": "아마다 애퀴지션",
			"BstpLargName": "금융",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AACIU",
			"KorIsnm": "아마다 애퀴지션 유닛",
			"BstpLargName": "금융",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AADI",
			"KorIsnm": "AADI 바이오사이언스",
			"BstpLargName": "헬스케어",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AADR",
			"KorIsnm": "ADVISORSHARES TRUST ADVISORSHARES DORSEY WRIGHT ADR ETF",
			"BstpLargName": "",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAGR",
			"KorIsnm": "아프리칸 애그리컬처 홀딩스",
			"BstpLargName": "필수 소비재",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAL",
			"KorIsnm": "아메리칸 에어라인스 그룹",
			"BstpLargName": "산업재",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAME",
			"KorIsnm": "애틀랜틱 아메리칸",
			"BstpLargName": "금융",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAOI",
			"KorIsnm": "어플라이드 옵토일렉트로닉스",
			"BstpLargName": "IT",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAON",
			"KorIsnm": "에이에이온",
			"BstpLargName": "산업재",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAPB",
			"KorIsnm": "GRANITESHARES ETF TRUST 2X LONG AAPL DAILY ETF",
			"BstpLargName": "",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAPD",
			"KorIsnm": "DIREXION SHARES ETF TRUST DAILY AAPL BEAR 1X SHS",
			"BstpLargName": "",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAPL",
			"KorIsnm": "애플",
			"BstpLargName": "IT",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAPU",
			"KorIsnm": "DIREXION SHARES ETF TRUST DAILY AAPL BULL 1.5X SHS",
			"BstpLargName": "",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "AAXJ",
			"KorIsnm": "ISHARES TRUST MSCI ALL COUNTRY ASIA EX JAPAN ETF",
			"BstpLargName": "",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "ABAT",
			"KorIsnm": "아메리칸 배터리 테크놀로지",
			"BstpLargName": "소재",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "ABCB",
			"KorIsnm": "아메리스 뱅코프",
			"BstpLargName": "금융",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "ABCL",
			"KorIsnm": "앱셀레라 바이오로직스",
			"BstpLargName": "헬스케어",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "ABCS",
			"KorIsnm": "EA SERIES TRUST ALPHA BLUE CAP US SM-MID CAP DYNAMIC ETF",
			"BstpLargName": "",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		},
		{
			"Iscd": "ABEO",
			"KorIsnm": "아베오나 테라퓨틱스",
			"BstpLargName": "헬스케어",
			"ExchClsCode2": "FN",
			"SelnVolUnit": "1",
			"ShnuVolUnit": "1"
		}
	],
	"rsp_cd": "00000",
	"rsp_msg": "정상 처리 되었습니다."
}
```


# 해외주식 멀티현재가조회
- API 명: 해외주식 멀티현재가조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/inquiry/multiprice
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식시세 멀티 현재가 조회 API입니다. <br><br> ※ 1회 호출에 최대 50종목의 시세를 확인 가능합니다.  <br> ※ "dataCnt"  필드에 요청할 데이터의 개수를 입력하여 호출이 가능 합니다. (1~50)  <br> ※ "dataCnt" 필드의 값과 입력 데이터의 개수가 일치하지 않으면 호출이 불가합니다.  <br>※ 아래와 같이시장구분필드와 종목코드가 1:1 쌍을 이뤄야 호출이 정상적으로 이뤄집니다.    <br> - InputIscd1:J (시장구분필드),  <br> - InputCondMrktDivCode1:005930 (종목코드) <br>※ [InputIscd1 ~ InputCondMrktDivCode1] & [InputCondMrktDivCode50 ~ InputCondMrktDivCode50]과 같이 최대 50건 호출이 가능합니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB증권 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|dataCnt|호출건수|string|Y|2|1~50사이의 값 입력|
|InputCondMrktDivCode1|입력조건시장분류코드1|string|Y|2|FY:뉴욕	FN:나스닥	FA:아멕스|
|InputIscd1|입력종목코드1|string|Y|12|해외주식 종목코드 (ex. TQQQ)|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB증권 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|array|Y|||
|Iscd|종목코드|string|Y|16||
|KorIsnm|한글종목명|string|Y|16||
|Sdpr|기준가|string|Y|16||
|Prpr|현재가|string|Y|16||
|Mxpr|상한가|string|Y|16||
|Llam|하한가|string|Y|16||
|Oprc|시가|string|Y|16||
|SdprVrssMrktRate|기준가대비시가비율|string|Y|16||
|PrprVrssOprcRate|현재가대비시가비율|string|Y|16||
|Hprc|고가|string|Y|16||
|SdprVrssHgprRate|기준가대비고가비율|string|Y|16||
|PrprVrssHgprRate|현재가대비고가비율|string|Y|16||
|Lprc|저가|string|Y|16||
|SdprVrssLwprRate|기준가대비저가비율|string|Y|16||
|PrprVrssLwprRate|현재가대비저가비율|string|Y|16||
|PrdyVrss|전일대비|string|Y|16||
|PrdyCtrt|전일대비율|string|Y|16||
## Example
### Request Example
```json
{
    "In": {
    "dataCnt": 5,
    "InputCondMrktDivCode1": "FN",
    "InputIscd1": "TSLA",
    "InputCondMrktDivCode2": "FN",
    "InputIscd2": "AAPL",
    "InputCondMrktDivCode3": "FN",
    "InputIscd3": "GOOG",
    "InputCondMrktDivCode4": "FN",
    "InputIscd4": "NVDA",
    "InputCondMrktDivCode5": "FN",
    "InputIscd5": "META"
    }
}
```
### Response Example
```json
{
	"Out": [
		{
			"Iscd": "TSLA",
			"KorIsnm": "테슬라",
			"Sdpr": "300.7100",
			"Prpr": "304.0600",
			"Mxpr": "0.0000",
			"Llam": "0.0000",
			"Oprc": "303.0700",
			"SdprVrssMrktRate": "0.78",
			"PrprVrssOprcRate": "",
			"Hprc": "304.6000",
			"SdprVrssHgprRate": "1.29",
			"PrprVrssHgprRate": "",
			"Lprc": "302.4800",
			"SdprVrssLwprRate": "0.59",
			"PrprVrssLwprRate": "",
			"PrdyVrss": "3.3500",
			"PrdyCtrt": "1.11"
		},
		{
			"Iscd": "AAPL",
			"KorIsnm": "애플",
			"Sdpr": "207.8200",
			"Prpr": "209.4900",
			"Mxpr": "0.0000",
			"Llam": "0.0000",
			"Oprc": "209.3800",
			"SdprVrssMrktRate": "0.75",
			"PrprVrssOprcRate": "",
			"Hprc": "209.6400",
			"SdprVrssHgprRate": "0.88",
			"PrprVrssHgprRate": "",
			"Lprc": "209.0000",
			"SdprVrssLwprRate": "0.57",
			"PrprVrssLwprRate": "",
			"PrdyVrss": "1.6700",
			"PrdyCtrt": "0.80"
		},
		{
			"Iscd": "GOOG",
			"KorIsnm": "알파벳 C",
			"Sdpr": "176.9100",
			"Prpr": "176.7500",
			"Mxpr": "0.0000",
			"Llam": "0.0000",
			"Oprc": "177.0000",
			"SdprVrssMrktRate": "0.05",
			"PrprVrssOprcRate": "",
			"Hprc": "177.0400",
			"SdprVrssHgprRate": "0.07",
			"PrprVrssHgprRate": "",
			"Lprc": "176.6500",
			"SdprVrssLwprRate": "-0.15",
			"PrprVrssLwprRate": "",
			"PrdyVrss": "-0.1600",
			"PrdyCtrt": "-0.09"
		},
		{
			"Iscd": "NVDA",
			"KorIsnm": "엔비디아",
			"Sdpr": "153.3000",
			"Prpr": "153.3600",
			"Mxpr": "0.0000",
			"Llam": "0.0000",
			"Oprc": "153.4400",
			"SdprVrssMrktRate": "0.09",
			"PrprVrssOprcRate": "",
			"Hprc": "153.6200",
			"SdprVrssHgprRate": "0.21",
			"PrprVrssHgprRate": "",
			"Lprc": "153.2200",
			"SdprVrssLwprRate": "-0.05",
			"PrprVrssLwprRate": "",
			"PrdyVrss": "0.0600",
			"PrdyCtrt": "0.04"
		},
		{
			"Iscd": "META",
			"KorIsnm": "메타 플랫폼스(페이스북)",
			"Sdpr": "719.2200",
			"Prpr": "720.0000",
			"Mxpr": "0.0000",
			"Llam": "0.0000",
			"Oprc": "720.0100",
			"SdprVrssMrktRate": "0.11",
			"PrprVrssOprcRate": "",
			"Hprc": "720.7400",
			"SdprVrssHgprRate": "0.21",
			"PrprVrssHgprRate": "",
			"Lprc": "719.7500",
			"SdprVrssLwprRate": "0.07",
			"PrprVrssLwprRate": "",
			"PrdyVrss": "0.7800",
			"PrdyCtrt": "0.11"
		}
	],
	"rsp_cd": "00000",
	"rsp_msg": "정상 처리 되었습니다."
}
```

# 해외주식현재가조회
- API 명: 해외주식현재가조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/inquiry/price
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 현재가 조회 API 입니다.
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
|InputCondMrktDivCode|입력조건시장분류코드|string|Y|2|FY:뉴욕	FN:나스닥	FA:아멕스|
|InputIscd1|입력종목코드1|string|Y|12|해외주식 종목코드 (ex. TQQQ)|
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
|Sdpr|기준가|string|Y|16||
|prdyVol|전일거래량|string|Y|16||
|AcmlVol|거래량|string|Y|16||
|AcmlTrPbmn|거래대금|string|Y|32||
|Per|PER|string|Y|16||
|PrdyCtrt|전일대비율|string|Y|16||
|PrdyVrss|전일대비|string|Y|16||
|askp1|매도호가|string|Y|16||
|bidp1|매수호가|string|Y|16||
|Prpr|현재가|string|Y|16||
|Mxpr|상한가|string|Y|16||
|Llam|하한가|string|Y|16||
|Oprc|시가|string|Y|16||
|SdprVrssMrktRate|기준가대비시가비율|string|Y|16||
|PrprVrssOprcRate|현재가대비시가비율|string|Y|16||
|Hprc|고가|string|Y|16||
|SdprVrssHgprRate|기준가대비고가비율|string|Y|16||
|PrprVrssHgprRate|현재가대비고가비율|string|Y|16||
|Lprc|저가|string|Y|16||
|SdprVrssLwprRate|기준가대비저가비율|string|Y|16||
|PrprVrssLwprRate|현재가대비저가비율|string|Y|16||
## Example
### Request Example
```json
{
  "In": {
    "InputIscd1": "TSLA",
    "InputCondMrktDivCode": "FN"
  }
}
```
### Response Example
```json
{
	"Out": {
		"Sdpr": "207.8200",
		"Prpr": "207.8200",
		"Mxpr": "0.0000",
		"Llam": "0.0000",
		"Oprc": "207.8200",
		"SdprVrssMrktRate": "0.00",
		"PrprVrssOprcRate": "",
		"Hprc": "207.8200",
		"SdprVrssHgprRate": "0.00",
		"PrprVrssHgprRate": "",
		"Lprc": "207.8200",
		"SdprVrssLwprRate": "0.00",
		"PrprVrssLwprRate": "",
		"PrdyVrss": "0.0000",
		"PrdyCtrt": "0.00",
		"Per": "32.430",
		"AcmlTrPbmn": "0",
		"AcmlVol": "0",
		"prdyVol": "78788867",
		"bidp1": "0.0000",
		"askp1": "0.0000"
	},
	"rsp_cd": "00000",
	"rsp_msg": "정상 처리 되었습니다."
}
```

# 해외주식호가조회
- API 명: 해외주식호가조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/inquiry/orderbook
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 호가 조회 API 입니다.
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
|InputCondMrktDivCode|입력조건시장분류코드|string|Y|2|FY:뉴욕	FN:나스닥	FA:아멕스|
|InputIscd1|입력종목코드1|string|Y|12|해외주식 종목코드 (ex. TQQQ)|
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
|Askp1|매도호가1|string|Y|16||
|Askp2|매도호가2|string|Y|16||
|Askp3|매도호가3|string|Y|16||
|Askp4|매도호가4|string|Y|16||
|Askp5|매도호가5|string|Y|16||
|Bidp1|매수호가1|string|Y|16||
|Bidp2|매수호가2|string|Y|16||
|Bidp3|매수호가3|string|Y|16||
|Bidp4|매수호가4|string|Y|16||
|Bidp5|매수호가5|string|Y|16||
|AskpRsqn1|매도호가잔량1|string|Y|16||
|AskpRsqn2|매도호가잔량2|string|Y|16||
|AskpRsqn3|매도호가잔량3|string|Y|16||
|AskpRsqn4|매도호가잔량4|string|Y|16||
|AskpRsqn5|매도호가잔량5|string|Y|16||
|BidpRsqn1|매수호가잔량1|string|Y|16||
|BidpRsqn2|매수호가잔량2|string|Y|16||
|BidpRsqn3|매수호가잔량3|string|Y|16||
|BidpRsqn4|매수호가잔량4|string|Y|16||
|BidpRsqn5|매수호가잔량5|string|Y|16||
|AskpRsqnIcdc1|매도호가잔량증감1|string|Y|16||
|AskpRsqnIcdc2|매도호가잔량증감2|string|Y|16||
|AskpRsqnIcdc3|매도호가잔량증감3|string|Y|16||
|AskpRsqnIcdc4|매도호가잔량증감4|string|Y|16||
|AskpRsqnIcdc5|매도호가잔량증감5|string|Y|16||
|BidpRsqnIcdc1|매수호가잔량증감1|string|Y|16||
|BidpRsqnIcdc2|매수호가잔량증감2|string|Y|16||
|BidpRsqnIcdc3|매수호가잔량증감3|string|Y|16||
|BidpRsqnIcdc4|매수호가잔량증감4|string|Y|16||
|BidpRsqnIcdc5|매수호가잔량증감5|string|Y|16||
## Example
### Request Example
```json
{
  "In": {
    "InputIscd1": "TSLA",
  "InputCondMrktDivCode": "FN"
 }
}
```
### Response Example
```json
{
 "Out": {
  "Askp1": "189.9000",
  "Askp2": "189.9100",
  "Askp3": "189.9200",
  "Askp4": "189.9300",
  "Askp5": "189.9400",
  "Bidp1": "189.8500",
  "Bidp2": "189.8400",
  "Bidp3": "189.8300",
  "Bidp4": "189.8200",
  "Bidp5": "189.8100",
  "AskpRsqn1": "25",
  "AskpRsqn2": "0",
  "AskpRsqn3": "0",
  "AskpRsqn4": "0",
  "AskpRsqn5": "0",
  "BidpRsqn1": "5",
  "BidpRsqn2": "0",
  "BidpRsqn3": "0",
  "BidpRsqn4": "0",
  "BidpRsqn5": "0",
  "AskpRsqnIcdc1": "-20",
  "AskpRsqnIcdc2": "0",
  "AskpRsqnIcdc3": "0",
  "AskpRsqnIcdc4": "0",
  "AskpRsqnIcdc5": "0",
  "BidpRsqnIcdc1": "0",
  "BidpRsqnIcdc2": "0",
  "BidpRsqnIcdc3": "0",
  "BidpRsqnIcdc4": "0",
  "BidpRsqnIcdc5": "0"
 },
 "rsp_cd": "00000",
 "rsp_msg": "정상 처리 되었습니다."
}
```

# 해외주식시간대별체결조회
- API 명: 해외주식시간대별체결조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/inquiry/hour-price
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 시간대별 체결조회 API입니다.
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
|InputCondMrktDivCode|입력조건시장분류코드|string|Y|2|FY:뉴욕	FN:나스닥	FA:아멕스|
|InputIscd1|입력종목코드1|string|Y|12|해외주식 종목코드 입력 (ex. TQQQ)|
|InputHourClsCode|입력시간구분코드|string|Y|9|0: 전체	1: 장전	2: 장중 	3: 장후	4: 장전+장중	5: 장전+장후|
|InputDivXtick|입력X틱분틱일별구분코드|string|Y|9|30: 30초	60: 1분	600: 10분	3600: 60분	[60*N: N분]|
|InputSctnClsCode|입력구간구분코드|string|Y|9|0:default 	1:당일 	2:전일|
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
|Hour|시간|string|Y|16||
|Prpr|현재가|string|Y|16||
|PrdyVrssSign|전일대비부호|string|Y|1||
|PrdyCtrt|전일대비율|string|Y|16||
|CntgVol|체결거래량|string|Y|16||

# 해외주식 틱차트조회
- API 명: 해외주식 틱차트조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/chart/tick
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 틱차트 조회 API 입니다.
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
|InputPwDataIncuYn|기간지정여부코드|string|Y|1|"Y": 기간지정	"N":기간미지정 (InputDate2 부터 이전날짜 계속조회)|
|InputDate1|입력날짜1|string|Y|16|조회 마감일을 YYYYMMDD 형식으로 입력 ex. 20241204|
|InputOrgAdjPrc|수정주가사용여부|string|Y|1|0:수정주가 미사용	1: 수정주가 사용|
|dataCnt|호출건수|string|Y|4|입력범위: "1" ~ "2000" 	""(공백입력) 또는 "0" 입력시 기본개수(400개)조회|
|InputHourClsCode|입력시간구분코드|string|Y|9|"0" 입력|
|InputCondMrktDivCode|입력조건시장분류코드|string|Y|2|FY:뉴욕	FN:나스닥	FA:아멕스|
|InputIscd1|입력종목코드1|string|Y|12|해외주식 종목코드 (ex. TQQQ)|
|InputDate2|입력날짜2|string|Y|8|조회 시작일을 YYYYMMDD 형식으로 입력 ex. 20241204	조회 시작일 부터 이전데이터를 조회합니다.|
|InputDivXtick|틱분틱일별구분코드|string|Y|9|틱건수 (기본값: 0)|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB금융투자 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|N|1|연속거래 여부|
|cont_key|연속키 값|string|N|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|array|Y|||
|Hour|시간|string|Y|6||
|Date|일자|string|Y|8||
|Prpr|현재가|string|Y|16||
|Oprc|시가|string|Y|16||
|Hprc|고가|string|Y|16||
|Lprc|저가|string|Y|16||
|CntgVol|체결거래량|string|Y|16||
## Example
### Request Example
```json
{
  "In": {
		"InputCondMrktDivCode": "FN",
		"InputIscd1": "TSLA",
		"InputDate1": "20250717",
		"InputDate2": "20250721",
		"InputHourClsCode": "0",
		"InputDivXtick": "600",
		"InputPwDataIncuYn": "Y",
		"dataCnt": "100"
  }
}
```
### Response Example
```json
{
	"Out": [
		{
			"Hour": "195957",
			"Date": "20250721",
			"Prpr": "327.7900",
			"Oprc": "327.6600",
			"Hprc": "327.8200",
			"Lprc": "327.5200",
			"CntgVol": "18818"
		},
		{
			"Hour": "195402",
			"Date": "20250721",
			"Prpr": "327.6501",
			"Oprc": "328.2300",
			"Hprc": "328.2400",
			"Lprc": "327.6500",
			"CntgVol": "39160"
		},
		{
			"Hour": "194230",
			"Date": "20250721",
			"Prpr": "328.2200",
			"Oprc": "327.9000",
			"Hprc": "328.2700",
			"Lprc": "327.8700",
			"CntgVol": "37295"
		}
	],
	"rsp_cd": "00000",
	"rsp_msg": "정상 처리 되었습니다."
}
```

# 해외주식 분차트조회
- API 명: 해외주식 분차트조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/chart/min
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 분차트 조회 API 입니다.
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
|InputPwDataIncuYn|기간지정여부코드|string|Y|1|"Y": 기간지정	"N":기간미지정 (InputDate2 부터 이전날짜 계속조회)|
|InputOrgAdjPrc|수정주가사용여부|string|Y|1|0:수정주가 미사용	1: 수정주가 사용|
|dataCnt|호출건수|string|Y|4|입력범위: "1" ~ "2000" 	""(공백입력) 또는 "0" 입력시 기본개수(400개)조회|
|InputHourClsCode|입력시간구분코드|string|Y|9|"0" 입력|
|InputCondMrktDivCode|입력조건시장분류코드|string|Y|2|FY:뉴욕	FN:나스닥	FA:아멕스|
|InputIscd1|입력종목코드1|string|Y|12|해외주식 종목코드 (ex. TQQQ)|
|InputDate1|입력날짜1|string|Y|8|YYYYMMDD|
|InputDate2|입력날짜2|string|Y|8|YYYYMMDD|
|InputDivXtick|분일별구분코드|string|Y|9|30: 30초	60: 1분	600: 10분	3600: 60분	[60*N: N분]|
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
|Hour|시간|string|Y|6||
|Date|일자|string|Y|8||
|Prpr|현재가|string|Y|16||
|Oprc|시가|string|Y|16||
|Hprc|고가|string|Y|16||
|Lprc|저가|string|Y|16||
|CntgVol|체결거래량|string|Y|16||
## Example
### Request Example
```json
{
    "In": {
        "InputCondMrktDivCode": "FN",
        "InputIscd1": "AAPL",
        "InputDate1": "20240201",
        "InputDate2": "20240205",
        "InputHourClsCode": "0",
        "InputDivXtick": "600",
        "InputPwDataIncuYn": "Y",
        "InputOrgAdjPrc": "1" ,
        "dataCnt": ""   
    }
}
```
### Response Example
```json
{
 "Out": [
  {
   "Hour": "163000",
   "Date": "20240205",
   "Prpr": "187.5700",
   "Oprc": "187.8300",
   "Hprc": "187.8500",
   "Lprc": "187.5150",
   "CntgVol": "14162"
  },
  {
   "Hour": "162000",
   "Date": "20240205",
   "Prpr": "187.8300",
   "Oprc": "187.9700",
   "Hprc": "188.0300",
   "Lprc": "187.6800",
   "CntgVol": "26443"
  },
  {
   "Hour": "161000",
   "Date": "20240205",
   "Prpr": "187.8700",
   "Oprc": "187.8300",
   "Hprc": "188.0500",
   "Lprc": "187.6800",
   "CntgVol": "47724"
  }
 ],
 "rsp_cd": "00000",
 "rsp_msg": "정상 처리 되었습니다."
}
```

# 해외주식 일차트조회
- API 명: 해외주식 일차트조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/chart/day
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 일차트 조회 API 입니다.
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
|InputCondMrktDivCode|입력조건시장분류코드|string|Y|2|FY:뉴욕	FN:나스닥	FA:아멕스|
|InputOrgAdjPrc|수정주가사용여부|string|Y|1|0:수정주가 미사용	1: 수정주가 사용|
|InputIscd1|입력종목코드1|string|Y|12|해외주식 종목코드 (ex. TQQQ)|
|InputDate1|입력날짜1|string|Y|8|YYYYMMDD|
|InputDate2|입력날짜2|string|Y|8|YYYYMMDD|
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
|Hour|시간|string|Y|6||
|Date|일자|string|Y|8||
|Prpr|현재가|string|Y|16||
|Oprc|시가|string|Y|16||
|Hprc|고가|string|Y|16||
|Lprc|저가|string|Y|16||
|AcmlVol|체결거래량|string|Y|16||

# 해외주식 주차트조회
- API 명: 해외주식 주차트조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/chart/week
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 주차트 조회 API 입니다.
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
|InputCondMrktDivCode|입력조건시장분류코드|string|Y|2|FY:뉴욕	FN:나스닥	FA:아멕스|
|InputOrgAdjPrc|수정주가사용여부|string|Y|1|0:수정주가 미사용	1: 수정주가 사용|
|InputIscd1|입력종목코드1|string|Y|12|해외주식 종목코드 (ex. TQQQ)|
|InputDate1|입력날짜1|string|Y|8|YYYYMMDD|
|InputDate2|입력날짜2|string|Y|8|YYYYMMDD|
|InputPeriodDivCode|입력일/주/월/년|string|Y|1|W|
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
|Hour|시간|string|Y|6||
|Date|일자|string|Y|8||
|Prpr|현재가|string|Y|16||
|Oprc|시가|string|Y|16||
|Hprc|고가|string|Y|16||
|Lprc|저가|string|Y|16||
|AcmlVol|체결거래량|string|Y|16||

# 해외주식 월차트조회
- API 명: 해외주식 월차트조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/chart/month
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 월차트 조회 API 입니다.
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
|InputOrgAdjPrc|수정주가사용여부|string|Y|1|0:수정주가 미사용	1: 수정주가 사용|
|InputCondMrktDivCode|입력조건시장분류코드|string|Y|2|FY:뉴욕	FN:나스닥	FA:아멕스|
|InputIscd1|입력종목코드1|string|Y|12|해외주식 종목코드 (ex. TQQQ)|
|InputDate1|입력날짜1|string|Y|8|YYYYMMDD|
|InputDate2|입력날짜2|string|Y|8|8YYYYMMDD|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100||
|cont_yn|연속 거래 여부|string|N|1||
|cont_key|연속키 값|string|N|18||
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|array|Y|||
|Hour|시간|string|Y|6||
|Date|일자|string|Y|8||
|Prpr|현재가|string|Y|16||
|Oprc|시가|string|Y|16||
|Hprc|고가|string|Y|16||
|Lprc|저가|string|Y|16||
|AcmlVol|누적체결거래량|string|Y|16||

# 해외주식 상승하락조회
- API 명: 해외주식 상승하락조회
- HTTP Method: POST
- Domain: https://openapi.dbsec.co.kr:8443
- URL: /api/v1/quote/overseas-stock/inquiry/rank-list
- 포맷: JSON
- 콘텐츠 타입: application/json;charset=utf-8
## 개요
해외주식 조건상승하락 조회 API입니다.  ※ 해외주식 상승/하락률 조건에 맞는 종목을 제공합니다.
## Request Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB증권 제공 API를 호출하기 위한 Request Body 데이터 포맷으로 "application/json; charset=utf-8" 설정|
|authorization|접근토큰|string|Y|1000|OAuth 토큰이 필요한 API 경우 발급한 Access Token을 설정하기 위한 Request Heaeder Parameter/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부(Y:연속거래 사용 N:연속거래 사용안함)|
|cont_key|연속키 값|string|N|70|연속일 경우 그전에 내려온 연속키 값 올림|
|mac_address|MAC 주소|string|N|12|법인인 경우 필수 세팅|
## Request Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|In|In|object|Y|||
|InputRealDelayClsCode|지연실시간구분코드|string|Y|2|0: 지연 	1: 실시간|
|InputDataCode|입력해외증시구분코드|string|Y|9|NY: 뉴욕 	NA: 나스닥 	AM: 아멕스 	US:미국|
|InputDateClsCode|입력일자구분코드|string|Y|9|0: 당일 	1: 전일|
|InputVolClsCode|입력거래량구분코드|string|Y|2|7: 10만주이상 	8: 50만주이상 	9: 100만주이상 	10:500만주이상 	11:1000만주이상 	12:5000만주이상|
|InputDprice1|가격최소값|string|Y|16||
|InputDprice2|가격최대값|string|Y|16||
|InputTrPbmn1|거래대금최소값|string|Y|16||
|InputRankSortClsCode1|입력순위정렬구분코드1|string|Y|9|249: 상승율 	250: 하락율|
## Response Header
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|content-type|컨텐츠타입|string|Y|100|DB증권 제공 API를 호출한 후 Client로 응답하는 Response Header Parameter로 "application/json; charset=utf-8" 설정|
|cont_yn|연속 거래 여부|string|Y|1|연속거래 여부|
|cont_key|연속키 값|string|Y|18|연속일 경우 그전에 내려온 연속키 값 올림|
## Response Body
|요소|한글명|타입|필수여부|길이|설명|
|---|---|---|---|---|---|
|Out|Out|object|Y|||
|MrktClsName|시장구분명|string|Y|16||
|AcmlTrPbmn|거래대금|string|Y|32||
|AcmlVol|거래량|string|Y|16||
|PrdyClpr|전일종가|string|Y|16||
|PrdyCtrt|전일대비율|string|Y|16||
|PrdyVrss|전일대비|string|Y|16||
|PrdyVrssSign|전일대비부호|string|Y|1||
|Prpr|현재가|string|Y|16||
|KorIsnm|한글종목명|string|Y|40||
|Iscd|종목코드|string|Y|16||
## Example
### Request Example
```json
{
  "In": {
    "InputRealDelayClsCode": "1",
		"InputDataCode": "US",
		"InputDateClsCode": "0",
		"InputRankSortClsCode1": "249",
		"InputVolClsCode" : "9",
		"InputTrPbmn1" : "",
		"InputDprice1" : "",
		"InputDprice2" : ""
	}
}
```
### Response Example
```json
{
	"Out": [
		{
			"MrktClsName": "나스닥",
			"Iscd": "HUBC",
			"KorIsnm": "허브 사이버 시큐리티",
			"Prpr": "0.1422",
			"PrdyVrssSign": "2",
			"PrdyVrss": "0.0362",
			"PrdyCtrt": "34.15",
			"PrdyClpr": "0.1060",
			"AcmlVol": "1281537",
			"AcmlTrPbmn": "180005"
		},
		{
			"MrktClsName": "아멕스",
			"Iscd": "SOXS",
			"KorIsnm": "DIREXION SEMICONDUCTOR DAILY -3X",
			"Prpr": "6.6900",
			"PrdyVrssSign": "2",
			"PrdyVrss": "0.1600",
			"PrdyCtrt": "2.45",
			"PrdyClpr": "6.5300",
			"AcmlVol": "1597113",
			"AcmlTrPbmn": "10693388"
		}
	],
	"rsp_cd": "00000",
	"rsp_msg": "정상 처리 되었습니다."
}
```
