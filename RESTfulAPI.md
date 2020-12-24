# REST API

- AWS Lambda + API gateway 사용



# amplify-user-web



## /index

### POST

- 매핑템플릿 - 통합요청

  Content-Type: application/json

  ```json
  {
    "rfid":$input.json('$.rfid')
  }
  ```

- 매핑템플릿 - 통합응답

  Content-Type: application/json

  ```json
  {
    "kings":$inputRoot('$.kings'),
    "data_amount":$input.json('$.data_amount'),
    "total_point":$input.json('$.total_point')
  }
  ```

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

- Lambda

```python
import json
import psycopg2


ENDPOINT = ""
PORT = "5432"
USR = ""
DBNAME = ""
PASSWORD = ""


# 사용자 닉네임과 토탈 포인트
QUERY = """
select nickname, total_point from recycle_ruser
order by total_point desc 
limit 3;
"""

# 페이지 헤더 - 분리수거된 유리병 개수
QUERY2 = """
select count(id)
from recycle_bottle
"""

QUERY3 = """
select total_point
from recycle_ruser 
where rfid = '%s'
"""

# QUERY2 = """
# select count(id)
# from recycle_bottle 
# where extract(month from datetime) = extract(month from now())
# """



def lambda_handler(event, context):
    
    try:
        conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
        cur = conn.cursor()
    
        cur.execute(QUERY)
        point_kings = cur.fetchall()
        
        cur.execute(QUERY2)
        data_amount = cur.fetchone()
    
        rfid = event['rfid']
        if rfid:
            cur.execute(QUERY3 % rfid)
            total_point = cur.fetchone()
            data = {"kings": point_kings, "data_amount": data_amount[0], "total_point": total_point[0]}

        else:
            data = {"kings": point_kings, "data_amount": data_amount[0]}

    
        return data
    
    except Exception as e:
        print("Database connection failed due to {}".format(e))
        print(e)
   
```





## /login

### POST

- 매핑템플릿 - 통합요청

  Content-Type: application/json

  ```json
  {
    "userid":$input.json('$.userid'),
    "password":$input.json('$.password')
  }
  ```

  

- 매핑템플릿 - 통합 응답

  Content-Type: application/json

  ```json
  {
      "userid":$input.json('$.userid'),
      "username":$input.json('$.username')
  }
  ```

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

- Lambda

```python
import psycopg2
from botocore.vendored import requests
import json

def lambda_handler(event, context):
    # DB 정보
    ENDPOINT = "recycledb.c8eedu3otduy.us-east-1.rds.amazonaws.com"
    PORT = "5432"
    USR = "team05"
    DBNAME = "recycledb"
    PASSWORD = "adminyes"
    
    # 쿼리
    QUERY = """
    select rfid, nickname from recycle_ruser where rfid = '%s' and pw = '%s';
    """
    END_QUERY = 'select * from recycle_ruser;'

    # DB 연결
    conn = None
    try:
        conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
        cur = conn.cursor()
        # print(conn)
    
        # ID, 비밀번호 검색
        userid = event['userid']
        password = event['password']
        cur.execute(QUERY % (userid, password))

        # 쿼리 실행 결과 dict로
        data = cur.fetchone()
        data = {"userid": data[0].rstrip(), "username": data[1].rstrip()}
    
        # 로그인 성공 실패 유무 출력
        if data:
            print("로그인 성공!!")
            print(data)
            
            # 웹으로 데이터 보내기
            # url = "https://taz84axy7a.execute-api.us-east-1.amazonaws.com/dev/login"
            # r = requests.post(url, data=json.dumps(data))
            # print(r.text)

        else:
            print("로그인 실패")
    
        return data
        
    except Exception as e:
        print("DB 연결 실패: {}".format(e))
        return e
        

```



## /point

### POST

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

- Lambda

```python
import json
import psycopg2


ENDPOINT = "recycledb.c8eedu3otduy.us-east-1.rds.amazonaws.com"
PORT = "5432"
USR = "team05"
DBNAME = "recycledb"
PASSWORD = "adminyes"


# rfid를 통해 얻은 날짜별 토탈 포인트
QUERY1 = """
select datetime, sum(recycle_points) from user_points
where rfid = '%s'
group by datetime
order by datetime desc;
"""

# 사용자 닉네임과 토탈 포인트
QUERY2 = """
select nickname, total_point from recycle_ruser
where rfid = '%s'
"""


def lambda_handler(event, context):
    print(event)
    
    try:
        conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
        cur = conn.cursor()
        
        # session에서 가져온 rfid 값
        rfid = event['rfid']

        cur.execute(QUERY1 % rfid)
        result = cur.fetchall()
    
        # 분리수거 내역 없을 경우 처리
        if result == None:
            result = ['', 0]
            
        else:
            result = [{'date': r[0].strftime("%Y-%m-%d   %H:%M"), 'point': int(r[1])} for r in result]
            result = dict(zip(range(1, len(result) + 1), result))
    
        cur.execute(QUERY2 % rfid)
        user = cur.fetchone()
    
        data = {
            "rfid": rfid,
            "list": result, 
            "nickname": user[0], 
            "total_point": user[1]
        }
        
        print(data)
        
        return data
        # {
        #     "rfid": rfid,
        #     "nickname": user[0],
        #     "total_point": user[1]
        # }
        
    except Exception as e:
        print("Database connection failed due to {}".format(e))
        print(e)

```







## /signup

### POST

- 매핑템플릿 - 통합요청

  Content-Type: application/json

  ```json
  {
    "userid":$input.json('$.userid'),
    "password":$input.json('$.password'),
    "username":$input.json('$.username')
  }
  ```

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

- Lambda

```python
import psycopg2


# DB 정보
ENDPOINT="recycledb.c8eedu3otduy.us-east-1.rds.amazonaws.com"
PORT="5432"
USR="team05"
DBNAME="recycledb"
PASSWORD="adminyes"

# 쿼리
QUERY="""
insert into recycle_ruser (rfid, pw, nickname, total_point) values(%s, %s, '%s', 0);
"""
END_QUERY = 'select * from recycle_ruser;'

def lambda_handler(event, context):
    # DB 연결
    conn = None
    try:
        conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
        cur = conn.cursor()
        print(conn)
        
    except Exception as e:
        print("DB 연결 실패: {}".format(e))  
        return e
        
    # 받아온 값 DB에 저장
    userid = event['userid']
    password = event['password']
    username = event['username']
    print("-------types", type(userid),type(password),type(username))
    cur.execute(QUERY%(userid, password, username))
    
    conn.commit()
    
```



# user-web-api



## /end

### POST

- 매핑템플릿 - 통합요청

  Content-Type: application/json

  ```json
  {
      "rfid":$input.json('$.rfid')
  }
  ```

  

- 매핑템플릿 - 통합 응답

  Content-Type: application/json

  ```json
  {
      "nickname":$input.json('$.nickname'),
      "total_point":$input.json('$.total_point'),
      "point":$input.json('$.point')
  }
  ```

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

- Lambda

```python
from botocore.vendored import requests

import psycopg2
import boto3

client = boto3.client('iot-data', region_name='us-east-1')

ENDPOINT = "recycledb.c8eedu3otduy.us-east-1.rds.amazonaws.com"
PORT = "5432"
USR = "team05"
DBNAME = "recycledb"
PASSWORD = "adminyes"

# rfid로 사용자 정보 찾기
QUERY1 = """
select nickname, total_point from recycle_ruser where rfid = '%s'
"""

# 적립된 포인트
QUERY2 = """
select sum(recycle_points) from user_points
where rfid = '%s'
group by datetime
order by datetime desc limit 1;
"""

# 새로 적립된 포인트 total_point에 더하기
QUERY3 = """
update recycle_ruser
set total_point = (select sum(recycle_points) from user_points
where rfid = '%s')
where rfid ='%s';
"""


def lambda_handler(event, context):
    
    # Change topic, qos and payload
    # response = client.publish(
    #       topic='user/end',
    #       qos=1,
    #       payload = "end"
    # )
    
    url = "https://gcjxk00ls1.execute-api.us-east-1.amazonaws.com/iot-user/end-signal"
    random_data = 'zzz'
    r = requests.post(url, random_data)
    
    rfid = event['rfid']
    # rfid = '1012714626'
    
    try:
        if rfid:
            conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
            cur = conn.cursor()
    
    
            cur.execute(QUERY2 % rfid)
            result2 = cur.fetchone()
            
            cur.execute(QUERY3 % (rfid, rfid))
            conn.commit()
            
            cur.execute(QUERY1 % rfid)
            result = cur.fetchone()
            
            
            data = {"nickname": result[0], "total_point": str(result[1]), "point": str(result2[0])}
            print(data)
            
        return data
        
    except Exception as e:
        print("Database connection failed due to {}".format(e))
        print(e)

```



## /end-signal

### POST

- Lambda

```python
import json
import boto3

client = boto3.client('iot-data', region_name='us-east-1')


def lambda_handler(event, context):

    # Change topic, qos and payload
    response = client.publish(
           topic='user/end',
           qos=1,
           payload = "end"
    )
    print(response)
    
    
```



## /login-progressing

### POST

- 매핑템플릿 - 통합 응답

  Content-Type: application/json

  ```json
  {
      "status":$input.json('$.status'),
      "rfid":$input.json('$.rfid')
  }
  ```

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

- Lambda

```python
import psycopg2

ENDPOINT = "recycledb.c8eedu3otduy.us-east-1.rds.amazonaws.com"
PORT = "5432"
USR = "team05"
DBNAME = "recycledb"
PASSWORD = "adminyes"

# 임시테이블의 rfid 가져오기
QUERY = """
select * from session_ruser limit 1;
"""

def lambda_handler(event, context):
    try:
        conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
        cur = conn.cursor()
        
        cur.execute(QUERY)
        result = cur.fetchone()
        # status => 0:오류 , 1: 성공
    
        if result == None:
            data = {"status": "0", "rfid": None}
            print("로그인 실패")
        else:
            data = {"status": "1", "rfid": result[0].rstrip()}
            print("로그인 성공")
            print(data)
            
        return data

        
    except Exception as e:
        print("Database connection failed due to {}".format(e))
        print(e)
```





## /result

### POST

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

- Lambda

```python
import json
import psycopg2


ENDPOINT = "recycledb.c8eedu3otduy.us-east-1.rds.amazonaws.com"
PORT = "5432"
USR = "team05"
DBNAME = "recycledb"
PASSWORD = "adminyes"

# rfid를 통해 얻은 날짜별 토탈 포인트
QUERY1 = """
select datetime, sum(recycle_points) from user_points
where rfid = '%s'
group by datetime
order by datetime desc
limit 1
"""

# 사용자 닉네임과 토탈 포인트
QUERY2 = """
select nickname, total_point from recycle_ruser
where rfid = '%s'
"""


def lambda_handler(event, context):
    
    try:
        conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
        cur = conn.cursor()
        
        # iot기기로부터 RFID값 받아오기
        rfid = event['rfid']

        cur.execute(QUERY1 % rfid)
        result = cur.fetchone()
        # 분리수거 내역 없을 경우 처리
        if result == None:
            result = ['', 0]

        cur.execute(QUERY2 % rfid)
        user = cur.fetchone()
        
        
        data = {"rfid": rfid, "date": str(result[0]), "daily_total_point": int(result[1]), "nickname": str(user[0]), "total_point": user[1]}
        print(data)
    
        return data

        # return cur_point, name, total_point
    
    except Exception as e:
        print("Database connection failed due to {}".format(e))
        print(e)

```



## /start

### GET

- Lambda

```python
import json
import boto3

client = boto3.client('iot-data', region_name='us-east-1')


def lambda_handler(event, context):

    # Change topic, qos and payload
    response = client.publish(
           topic='user/end',
           qos=1,
           payload = "start"
    )
    print(response)
```



## /tag-waiting

### POST

- 매핑템플릿 - 통합 응답

  Content-Type: application/json

  ```json
  {
      "status":$input.json('$status')
  }
  ```

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

- Lambda

```python
import psycopg2

ENDPOINT = "recycledb.c8eedu3otduy.us-east-1.rds.amazonaws.com"
PORT = "5432"
USR = "team05"
DBNAME = "recycledb"
PASSWORD = "adminyes"

# 테이블 비우기
QUERY1 = """
delete from session_ruser;
"""
# rfid 유효한지 검증
QUERY2 = """
select * from recycle_ruser where rfid = '%s'
"""
# 뷰 테이블에 넣기
QUERY3 = """
insert into session_ruser (rfid) values(%s);
"""


def lambda_handler(event, context):
    rfid = event['message']
    # rfid = '1012714626'
    
    try:
        conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
        cur = conn.cursor()
        
        cur.execute(QUERY1)
        conn.commit()
        
        cur.execute(QUERY2 % rfid)
        result = cur.fetchone()

        if result == None:
            print("유효한 RFID가 아님")
        else:
            print("테이블에 저장 완료")
            cur.execute(QUERY3 % rfid)
            conn.commit()

    except Exception as e:
        print("Database connection failed due to {}".format(e))
        print(e)

    
```



## S3 버킷에 추가된 객체에 대한 이벤트 Lambda

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

```
import json
import psycopg2
import sys
import boto3
from datetime import datetime
from urllib.parse import unquote_plus

ENDPOINT="recycledb.c8eedu3otduy.us-east-1.rds.amazonaws.com"
PORT="5432"
USR="team05"
DBNAME="recycledb"
PASSWORD="adminyes"
QUERY="""
insert into recycle_bottle(image, datetime, bottle_class, user_rfid, machine_id)
values ('%s', '%s', %d, '%s', 1);
"""


s3_url = "https://recycle-img.s3.amazonaws.com/"


s3_client = boto3.client('s3')


def lambda_handler(event, context):

    data = []
    # 1012714626_20201214_174932_15_3_1.jpg
    
    for record in event['Records']:
        img_name = unquote_plus(record['s3']['object']['key'])
        print('이미지 명: ', img_name)

        # t = datetime.strptime(now_time, '%y%m%dT%H%M').strftime("%Y-%m-%d %H:%M")
        user, now_date, now_time, bottle, result, num = img_name[:-4].split('_')
        date = datetime.strptime(now_date + now_time, '%Y%m%d%H%M%S')
        data.append([s3_url+img_name, date, int(bottle), user])


    
    try:
        conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
        cur = conn.cursor()
    
        cur.execute("SET session_replication_role = replica;")
    
        for d in data:
            cur.execute(QUERY % (d[0], d[1], d[2], d[3]))
        cur.execute("SET session_replication_role = DEFAULT;")
    
        # 실행한 쿼리문 결과 DB에 반영
        conn.commit()

    except Exception as e:
        print("Database 연동 실패: {}".format(e)) 
        return e


```



## 분리수거함 다 찼을 때 MQTT 통신

- Layer

  arn:aws:lambda:us-east-1:898466741470:layer:psycopg2-py37:3

```
from botocore.vendored import requests

import json
import boto3
import psycopg2
from datetime import datetime, timedelta


ENDPOINT="recycledb.c8eedu3otduy.us-east-1.rds.amazonaws.com"
PORT="5432"
USR="team05"
DBNAME="recycledb"
PASSWORD="adminyes"

QUERY="""
insert into recycle_collection (id, collected_date, class1_amount, class2_amount, class3_amount, machine_id, partner_id) 
values (default, '%s', %s, %s, %s, 1, 1);
"""

QUERY2 = """
update recycle_machine
set class{}_full_rate = 0
where id = 1;
"""

QUERY3 = """
select local1, local2 from recycle_machine
"""


def lambda_handler(event, context):

    full_num = int(event['message'])

    # 가득 찬 칸 = 1, 아닌 칸 = 0
    full_check = [0, 0, 0]
    full_check[full_num-1] = 1
    
    now = datetime.now() + timedelta(hours=9)
    print(now)
    
    location = ''
        
        
    try:
        conn = psycopg2.connect(host=ENDPOINT, port=PORT, database=DBNAME, user=USR, password=PASSWORD)
        cur = conn.cursor()
    
        # 쓰레기통 다 참 -> DB에 저장
        cur.execute(QUERY % (now, full_check[0], full_check[1], full_check[2]))
    
        # 쓰레기통 비운 후 리셋
        cur.execute(QUERY2.format(full_num))
    
        conn.commit()
    
        # 쓰레기통 위치 찾기
        cur.execute(QUERY3)
        result = cur.fetchone()
        location = "{} {}".format(result[0], result[1])
    
    except Exception as e:
        print("Database connection failed due to {}".format(e))
        print(e)
    
    data = {
        'num': full_num,
        'location': location,
        'address': "서울특별시 강남구 삼성동 160-21"
    }
    
    eventText = json.dumps(data)


    # 문자 전송 rest api 호출
    eventText = json.dumps(data)
    print(eventText)
    url = "https://923b5o8j40.execute-api.us-east-1.amazonaws.com/conn/sns"
    # r = requests.post(url, data=eventText)
    
    
```



- 문자 전송 Lambda 함수

```
import json
import boto3

def lambda_handler(event, context):

    print('연동!')
    print(event)
    n = event['num']
    location = event['location']
    address = event['address']
    
    # eventText = json.dumps()
        
    eventText = """{} 분리수거함 {}번칸을 비워주세요.
주소: {}""".format(location, n, address)
    
    sns = boto3.client('sns')
    
    response = sns.publish (
        TopicArn = 'arn:aws:sns:us-east-1:013223759423:full-mqtt-sns-topic',
        Message = eventText
    )
    
    print('이메일 확인')
    
   
```

