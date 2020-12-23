# 유리온 유저 웹 & 라즈베리파이 LCD 웹

## 유리온이란🤔?

### 1. 수익성 있는 병, 분리수거가 가능한 유리, 분리수거 불가능한 유리 세 가지를 분리합니다.

### 2. 기기에서 수익성이 있는 분리수거 용품을 분리하여 분리수거 이용자에게 포인트를 제공합니다.

### 3. 기기에서 분리수거 업자에게 일정량 이상의 쓰레기가 쌓이면 문자 알림이 가는 서비스를 제공합니다.



# 프로젝트 정보

## ◼ 프로젝트 기간

2020.11.18 ~ 2020.12.24

## ◼ 개발 플랫폼

▪ Windows 10

## ◼ 개발 툴

▪ Amazon Web Service(AWS)

▪ Pychram

▪ Visual Studio Code 

## ◼ 사용 언어

▪ HTML / CSS / JavaScript

▪ Python 3.7

## ◼ 사용 기술

▪ Amazon Web Service(AWS)

​	▫ API Gateway

​	▫  Lambda

​	▫  RDS

​	▫ SNS

​	▫  S3

​	▫ IoT Core

▪ Ajax

▪ PostgreSQL



# 아키텍처

## 💻 사용자 웹

![KakaoTalk_20201219_191615207](https://user-images.githubusercontent.com/59273807/102947410-9b25d400-4506-11eb-800a-a375086677e0.png)

▪ AWS Amplify를 이용하여 웹 호스팅 - Github과 연동되어 commit시 자동으로 배포

▪ AWS Lambda를 이용하여 Amazon RDS(PostgreSQL)와 통신 (via psycopg2 Layer)

▪ Ajax를 이용하여 API Gateway와 통신(JSON 형식)

▪ API Gateway는 AWS Lambda를 호출하여 Web↔Lambda의 중계기 역할 

| 리소스  | 메소드 | Lambda 함수           | 역할                                                         |
| ------- | ------ | --------------------- | ------------------------------------------------------------ |
| /index  | POST   | user-web-index-page   | 로그인 여부 파악하여 인덱스 페이지  콘텐츠를 DB로부터 가져오고 결과를 보냄 |
| /point  | POST   | PointListAPI          | 로그인된 사용자의 ID 값을  이용하여 DB로부터 포인트 적립 내역을 가져오고 결과를 보냄 |
| /login  | POST   | get-user-info-from-db | 로그인 - Ajax로 받은 로그인  폼 데이터를 DB에 검색하여 로그인 성/패 결정하고 결과를 보냄 |
| /signup | POST   | put-user-info-to-db   | 회원가입 - Ajax로 받은  회원가입 폼 데이터를 DB에 저장       |



## 💻 라즈베리 파이 LCD 웹

![LCD웹](https://user-images.githubusercontent.com/59273807/102949179-02de1e00-450b-11eb-9e50-14a72a8ffa1e.png)

#### 분리수거 시작

▪ 사용자가 RFID 리더기에 카드를 태그하면 AWS IoT로 ID값 전송(MQTT)

▪ AWS Lambda에서 전송받은 ID값을 이용해 DB에 조회하여 사용자 정보 return

▪ return한 사용자 정보를 LCD 웹으로 전송(via API Gateway)

#### 분리수거 종료

▪ 사용자가 LCD 웹의 '분리수거 완료'버튼을 누르면 Lambda로 신호 전송

▪ 신호를 받은 Lambda는 라즈베리 파이에 신호 전달

▪ 라즈베리 파이는 AI에 의해 인식된 유리병들의 이미지, 인식 결과를 S3버킷으로 전송(인식 결과 및 로그인한 사용자의 아이디, 인식 시간 등은 이미지의 파일명으로 저장)

▪ S3 버킷 업로드에 의해 트리거 된 Lambda는 S3 버킷의 이미지 파일명에서 데이터를 추출하여 DB에 저장하고 사용자의 보유 포인트 정보를 받아와 웹으로 전송

| 리소스            | 메소드 | Lambda 함수             | 역할                                                         |
| ----------------- | ------ | ----------------------- | ------------------------------------------------------------ |
| start-signal      | GET    | StartMQTT2              | 라즈베리 파이에 분리수거 시작 신호  보내기(for 기기 음성 출력) |
| tag-waiting       | POST   | rfid-data-to-view-table | RFID 리더기에 카드가 태그되면  유효한지 검증 후, 값을 임시 테이블에 저장 |
| login-progressing | POST   | start-get-rfid          | 빈 요청 보내고 임시 테이블의  RFID값 받아와서 세션 생성      |
| end               | POST   | end-get-user-info       | 분리수거 종료 버튼 클릭시 세션의  RFID값을 보내 닉네임, 적립된 포인트, 총 포인트 받아오기 |
| end-signal        | POST   | CloudtoDeviceTest       | 라즈베리 파이에 분리수거 종료 신호  보내기(for 기기 음성 출력) |



# 데이터베이스 구조

<img width="681" alt="ERD" src="https://user-images.githubusercontent.com/59273807/102948354-2dc77280-4509-11eb-83e5-5000863b77ad.png">



# 스토리보드

## 💻 사용자 웹

![그림1](https://user-images.githubusercontent.com/59273807/102953554-e810a700-4514-11eb-9c0b-b5aa232f653d.png)

## 💻 라즈베리 파이 LCD 웹

![그림2](https://user-images.githubusercontent.com/59273807/102954324-8cdfb400-4516-11eb-842e-a2994a68b518.png)



# 제공 서비스

▪ 로그인 / 회원가입

▪ 포인트 조회

▪ Raspberry Pi와 MQTT 통신

