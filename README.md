# ELK ( 'Learning Elastic Stack 6.0' 교재 )

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
        sincedb_path => "NULL"
        ignore_older => 0
    }
}
```

- path => 입력 파일이 위치한 곳
- start_position => Logstash를 시작할 때 file의 어디서부터 읽을지를 결정하는 설정  
=> beginning 과 end(default) 설정을 지원함  
_주의 : 이 설정은 file을 최초 읽을 때만 적용된다는 점. start_position => "beginning" 인 경우 logstash를 여러번 restart 하더라도 동일한 내용이 출력될 것이라 생각하는데 그렇지 않음! _

- sincedb : logstash가 최초 시작될 때는 file을 읽을 위치가 start_position에 의해 결정되지만, 이후 실행시에는 sincedb에 저장된 offset부터 읽기 시작함!

```
$ cat ~/.sincedb_a8ae6517118b64cf101eba5a72c366d4
22253332 1 3 14323
```

=> 위의 예에서 22253332번 파일은 현재 14323번 offset까지 처리가 되었다는 걸 의미   
=> logstash를 재실행하는 경우 start_position값과 상관없이 14323 offset부터 file 을 읽기 시작!  

_Logstash를 테스트 하는 동안에는 이 sincedb 때문에 테스트가 좀 번거로움 이경우 NULL 로 path 설정!_

- ignore_older => 0 : 로그 스태쉬는 기본적으로 파일이 하루 이상 오래된 경우 input으로 인식하지 않는다. 이러한 동작을 멈추기 위해 사용

**2 Filter 설정**

- filter 작업은 Work Queue 에서 이루어지므로 Worker 를 늘려주면 빠르게 작업할 수 있음 => logstash Tunning 시에 중요 포인트가 됨  


- Filter Plugin: Grok
=> 로그 스태쉬에서 기본적으로 사용할수 있는 필터 플러그인이다.  
=> 불규칙적인 데이터를 정형화된 데이터로 만들어줌  
=> 자주 쓰이는 정규표현식 패턴이 사전정의되어 있어 패턴을 골라서 사용가능  

- mutate
=> 데이터 필드 단위로 변형할수 있음 / 필드를 타입 변경, join, 이름 변경 가능  
=> elastic search 에서 indexing된 data를 변경하는 것은 상대적으로 어려움. indexing 전에 필드를 변경해서 넘기는 것이 속도도 빠르고
불필요한 데이터 양도 줄일수 있음 
- date  
=> String을 Data타입으로 변환함. 날짜를 string 타입으로 elastic search에서 indexing하면 query나 aggregation할때 문제 발생 가능  

- json : input 의 json 데이터 구조 유지
- kv : key-value 형

### Logstash 실습1 ( csv file 분석하기 )

- 샘플 데이터 : stock-data.csv

```
input {
   file {
     path => ""
     start_position => "beginning"
     sincedb_path => "/dev/null"
    }
}
 
filter {
   csv {
     separator => ","
     columns => ["Date","Open","High","Low","Close","Volume","Adj Close"]
  }
 mutate {
   convert => {
      "Open" => "float"
      "High" => "float"
      "Low" => "float"
      "Close" => "float"
      "Volume" => "float"
      "Adj Close" => "float"
    }
  }
}
 
output {
   elasticsearch {
     hosts => "127.0.0.1:9200"
     index => "stock-%{+YYYY.MM.dd}"
   }
 
 stdout {
  }
}
```
- csv 컬럼이 String 이기 때문에 컬럼 타입을 Float로 변경 ( mutate 플러그인 )

### Logstash 실습 2 (Apache Access Log 분석)

샘플 데이터 : sample-access.log

```
input {
   file {
     path => ""
     start_position => "beginning"
     sincedb_path => "/dev/null"
    }
}
 
filter {
 grok {
   match => { "message" => "%{COMMONAPACHELOG}" }
  }
 date {
   match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}
 
output {
 elasticsearch {
   hosts => "127.0.0.1:9200"
   index => "apache-log"
  }
 stdout {}
}
```

- date 필터를 이용하여 이벤트 timestamp로 만들때 사용 
- @timestamp 값이 현재시간이 아니라, 로그 메시지에 있는 시간 

### Log Rotate 사용 시, Log 유실이 없으려면 ? / 파일 복수 개 처리 방식

- 참고 : http://jason-heo.github.io/elasticsearch/2016/02/28/logstash-offset.html

참고 : http://blog.naver.com/PostView.nhn?blogId=kbh3983&logNo=221063092376

## beat

- 로그 수집하는 경량 로그 수집기
- 바이너리로 실행하며 JVM 처럼 런타임이 필요하지 않기 때문에 매우 가볍고 적은 자원을 소모
- 비트는 경량 에이전트로 더 적은 자원을 소비하기 때문에 운영 데이터를 수집하고 전달해야하는 엣지 서버에 설치해 사용해야함
- 즉, 비트는 이벤트 변환 및 분석 로그스태시에서 제공하는 강력한 기능은 없음



## Elastic Search
- 매우 풍부한 Rest API를 제공( Representational State Transfer )

ex) put /catalog/product/1  
==> put 새로운 도큐먼트를 색인 ( index - type - 색인 생성 후 할당되는 id)

- 일래스틱서치에서 타입은 테이블 / 도큐먼트는 테이블 레코드와 유사 (하지만 관계형 데이터 베이스는 거의 항상 다중 테이블을 포함하지만, 일래스틱 서치는 단일 인덱스와 단일 타입만 가질수 있음)


**인덱스**

- 일래스틱서치에서 단일 타입의 도큐먼트를 저장하고 관리하는 컨테이너
- 인덱스틑 타입을 포함한 논리적인 컨테이너
- 일래스틱서치 6.0 이전 버전에서는 단일 인덱스에 여러 타입을 포함 할수 있었지만 6.0 버전부터는 인덱스에는
단 하나의 타입만 가질수 있도록 변경, 하지만 6.0 버전 이전에 만든 다중 타입을 가진 인덱스가 있는 경우에는 일래스틱서치 6.0 버전으로 
업그레이드 할때 변경 없이 사용할수는 있음   
=> 관계형 데이터베이스에서 테이블은 서로 독립적, 특정 테이블의 컬럼은 다른 테이블에서 같은 이름을 가진 컬럼과 
관련이 없다. 하지만 일래스틱서치에서 같은 이름을 가진 매핑 필드는 내부적으로 같은 필드로 저장되기 때문에 단일 매핑 타입만 지원하는 것!

**타입**

- 관계형 데이터 베이스에서 테이블과 비슷

**도큐먼트**

- 레코드와 비슷 ( 키(필드) 와 값(필드값) 으로 이루어져있음)

**노드**
- 일래스틱 서치는 분산시스템이고, 네트워크에 위치한 각 시스템에서 실행되고 다른 프로세스와 통신하는 다중 프로세스
- 모든 노드는 시작할때 고유 id와 이름이 지정되고 일래스틱서치 config/elasticsearch.yml
- 노드는 가장 낮은 레벨에서 일래스틱서치 프로세스의 단일 인스턴스로 데이터 공유를 담당한다.

**클러스터**
- 클러스터는 단일 혹은 다중 인덱스를 호스팅
- 여러 노드로 구성될수 있으며, 각 노드는 공유 데이터를 저장하고 관리
- 모든 일래스틱서치 노드는 항상 클러스터의 부분 집합
- 기본적으로 모든 일래스틱 서치 노드는 elasticsearch 라는 이름으로 클러스터에 참여하려고 시도한다
==> config/elasticsearch.yml ㅍ파일의 cluster.name 속성을 변경하지 않고 같은 네트워크에서 여러 노드를 시작하면 클러스터가 자동으로 구성됨   
==> 같은 네트워크 환경에서 개발자 컴퓨터나 다른 시스템에서 노드를 실행 할때 문제 일으킬수 있기 때문에 설정 파일 변경 권장!

**샤드 및 복제본**
- 클러스터에서 인덱스를 분배하고 단일 인덱스의 도큐먼트를 여러 노드로 분할하는데 사용
- 샤드에 위치한 데이터를 분할하는 과정을 샤딩이라고 함
- 샤드에 위체한 데이터를 분할하는 과정을 샤딩!!
- 기본적으로 모든 인덱스는 일래스틱서치에서 5개의 샤드를 갖도록 구성
- 인덱스 생성 시점에 인덱스의 데이터를 나눌 샤드 개수를 지정가능하지만 인덱스 생성하고 나면 샤드 개수 변경 불가능

요약,
- 하나 이상의 노드가 모여 클러스터를 형성
- 클러스터는 다중 인덱스를 생성할수 있는 물리계층 서비스를 제공
- 인덱스는 타입을 포함하고 수백 또는 수십억 개의 도큐먼트를 포함
- 인덱스의 데이터는 개별 조각으로 분리돼 샤드로 나뉜다. 
- 샤드는 클러스터 노드 전반에 걸쳐 분산되고, 주 샤드의 복제본을 통해 고가용성과 장애 조치를 지원!

__관계형 데이터 베이스가 일반적인 crud 연산에 적합한 b트리를 만들고 관리하는 것과 달리 일래스틱 서치는 아파치 루씬 기능을 사용해 역색인 이라는 완전히 다른 데이터 구조로 데이터를 관리한다는점!!__

## Kibana

- 








