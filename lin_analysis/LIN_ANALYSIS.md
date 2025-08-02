# BMW Motorrad 바이크의 LIN 신호 분석
### 분석내용은 S1000XR(2016식) 기준입니다.
### 다른 바이크 오너분들의 추가적인 분석을 기다립니다.

## 작성방법
바이크에서 발생하는 LIN 신호는 한번에 7 byte 씩 전송됩니다.

### Byte 표현방법
|byte 번호|byte 내용|
|---|---|
|0|Message ID|
|1|Data-0|
|2|Data-1|
|3|Data-2|
|4|Data-3|
|5|Data-4|
|6|Data-5|
|7|Data-6|

몇몇 정보들은 1Byte를 4bit 단위로 나눠서 사용하기도 합니다. 따라서 Bit 단위의 기록이 필요할 수도 있습니다.
### Bit 표현방법
#### 0xA1 일 경우
|7th|6th|5th|4th|3rd|2nd|1st|0th|
|---|---|---|---|---|---|---|---|
|1|0|1|0|0|0|0|1|

#### 7th ~ 4th 비트 배열을 MSB / 3th ~ 0th 비트 배열을 LSB로 표현합니다.


발생 신호를 사용하는 방법은 다음 프로젝트를 참조해주세요
* [클러스터 애플리케이션 개발](https://github.com/kendrickkim/bmw-motorrad-cluster-application)
* [ESP32 펌웨어 개발](https://github.com/kendrickkim/bmw-motorrad-cluster-esp32)

<br/>
<br/>
<br/>

# Message ID 별 분석 내용
## ID 0x14
---
### DATA-0
* 분석되지 않음

### DATA-1
* Left Handle Controller 1
* S1000XR / ...
<table border="1">
  <tr>
    <th>7</th>
    <th>6</th>
    <th>5</th>
    <th>4</th>
    <th>3</th>
    <th>2</th>
    <th>1</th>
    <th>0</th>
  </tr>
  <tr>
    <td>Horn 버튼</td>
    <td>방향 지시등 취소</td>
    <td>방향지시등 오른쪽</td>
    <td>방향지시등 왼쪽</td>
    <td>크루즈 버튼</td>
    <td>경보 버튼</td>
    <td>ABS 버튼</td>
    <td>ESA 버튼</td>
  </tr>
</table>

### DATA-2
* Left Handle Controller 2
* S1000XR / ...
<table border="1">
  <tr>
    <th>7</th>
    <th>6</th>
    <th>5</th>
    <th>4</th>
    <th>3</th>
    <th>2</th>
    <th>1</th>
    <th>0</th>
  </tr>
  <tr>
    <td>?</td>
    <td>?</td>
    <td>Trip</td>
    <td>Info</td>
    <td>크루즈 Set</td>
    <td>크루즈 Reset</td>
    <td>하이빔</td>
    <td>?</td>
  </tr>
</table>

### DATA-3
* Wonder Wheel Control in out
* S1000XR / ...
  
|값|설명|
|-|-|
|0xFD|Out|
|0xFE|In|

#### DATA-3
* Wonder Wheel Control - wheel value
* S1000XR / ...
  
0x00 ~ 0xFF 까지 휠 회전에 따라 값이 변합니다.

## ID 0x20
---
### DATA-0
* 분석되지 않음

### DATA-1
* 0x7F 일 경우 IGNITION ON

### DATA-2
* 후륜 RPM 의 하위 8비트

### DATA-3
* LBS 4bit 는 후륜 RPM 의 상위 4비트
<br/>
<br/>

#### 차량 속도 계산 방법
- 후륜 RPM = ((data[3] & 0xF0) << 8) + (data[2] & 0xFF)
- 속도 = 후륜 RPM x 0.121
- (후륜 타이어 : 190/55ZR17) 후륜 지름 : 642mm - 1회전시 2017mm 이동
- 속도 = (후륜 RPM x 60 x 2017) / 1000 / 1000

## ID 0x2b
---
* 분석되지 않음

## ID 0x2e
---
* 분석되지 않음

## ID 0xe2
---
### DATA-0
* RPM / 5 의 하위 8비트

### DATA-1
* LBS 4bit 는 RPM / 5 의 상위 4비트
* RPM = (((data[1] & 0xF0) << 8) + (data[0] & 0xFF)) * 5
* MSB 4bit 는 현재 기어 정보
  
|값|설명|
|-|-|
|0x10|1단|
|0x20|N|
|0x40|2단|
|0x70|3단|
|0x80|4단|
|0xB0|5단|
|0xD0|6단|
|0xF0|기어 단수 사이에 변경 위치|

## ID 0xe2
---
### DATA-5
* 배터리 전압 정보
* 배터리 전압(V) = (data[5] & 0xFF) * 0.1