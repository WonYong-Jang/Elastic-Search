#ELK 

## Logstash

**실시간 파이프라인 기능을 가진 오픈소스 데이터 수집엔진(DataFlow Engine)**
**파이프라인 : 데이터 처리를 위한 logstash 설정(input, filter, output)**

- 서비스에 어떤 데이터가 필요한지에 따라 어떤 플러그인을 선택하여 필터할것인가를 고민하는게 파이프라인 설계 핵심!

- input(with codec) -> filter -> output(with codec) 을 통해 틀정 형식을 만족하는 로그를 정형화된 형식의 output으로 생성하기
위한 필터 역할  

- input 플러그인 : consume data from a source
=> file, redis, beats  
- filter 플러그인 : modify the data as you specify
=> grok, mutate, drop, geoip  
- output 플러그인 : write the data to a destination 
=> elasticsearch, file, kafka  
- codec : input 과 output에서 데이터를 인코딩하거나 디코딩 하기 위한 코덱 지원
=> json, multiline  

$ logstash -e 'input { stdin {}} output {stdout {} }'  
// -e 옵션을 이용하면 설정 파일을 읽지 않고, 그 다음에 오는 명령을 설정으로 인식  
// 위의 명령어는 stdin -> filter 없음 -> stdout으로 동작하는 로그 스태쉬를 실행  
// 명령어를 입력 후 hello world를 입력하면 timestamp, host 정보를 붙여서 hello world가 필터 되는것을 확인!  

### 설정파일
 /config 디렉토리에 위치!
- logstash.yml : Logstash 실행과 관련된 설정이 들어있음 
- pipeline.yml : 단일 Logstash 인스턴스에서 여러개의 파이프라인을 실핼할 때 사용 
- jvm.option : JVM 설정이 들어가 있음, 이설정파일을 통해 힙사이즈를 조절하거나 GC 관련 옵션 등을 설정 가능 
- log4j2.properties : log4j 관련 설정 파일 

$ logstash -f {설정파일 명}
// 설정파일을 읽어서 로그스태쉬를 실행시킴 
// 설정 파일은 input, filter, output 으로 구성 

==> Logstash를 파라미터 없이 시작하는 경우, pipeline.yml 파일을 읽어서 해당 파일에 정의된 모든 파이프라인을 시작   
==> 반면 Logstash를 -e , -f 를 사용하는 경우 pipeline.yml를 사용하지 않음 

**1. Input 설정**

ex) input 설정 예제
```
input {
    file{
        path => "/path/access.log"
        start_position => "beginning"
        ignore_older => 0
    }
}
```

- path => 입력 파일이 위치한 곳
- start_position => beginning : 로그 스태쉬는 기본적으로 unix시스템 tail -f 커맨드와 동일하게 동작
즉, access.log 파일의 증가분에 대해서만 input으로 인식 
따라서, 증가분이 아니라 항상 파일의 처음부터 input으로 인식하게 하기 위해 start_position=>beginning으로 추가
- ignore_older => 0 : 로그 스태쉬는 기본적으로 파일이 하루 이상 오래된 경우 input으로 인식하지 않는다. 이러한 동작을 멈추기 위해 사용

**2, Filter 설정**

- filter 작업은 Work Queue 에서 이루어지므로 Worker 를 늘려주면 빠르게 작업할 수 있음 => logstash Tunning 시에 중요 포인트가 됨  


- Filter Plugin: Grok
=> 로그 스태쉬에서 기본적으로 사용할수 있는 필터 플러그인이다.  
=> 불규칙적인 데이터를 정형화된 데이터로 만들어줌  
=> 자주 쓰이는 정규표현식 패턴이 사전정의되어 있어 패턴을 골라서 사용가능  

- mutate
=> 데이터 필드 단위로 변형할수 있음 / 필드를 타입 변경, join, 이름 변경 가능  
=> elastic search 에서 indexing된 data를 변경하는 것은 상대적으로 어려움. indexing 전에 필드를 변경해서 넘기는 것이 속도도 빠르고
불필요한 데이터 양도 줄일수 있음 
- data 
=> String을 Data타입으로 변환함. 날짜를 string 타입으로 elastic search에서 indexing하면 query나 aggregation할때 문제 발생 가능  

- json : input 의 json 데이터 구조 유지
- kv : key-value 형




참고 : http://blog.naver.com/PostView.nhn?blogId=kbh3983&logNo=221063092376












