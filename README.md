# ELK 

## Logstash

- input -> filter -> output 을 통해 틀정 형식을 만족하는 로그를 정형화된 형식의 output으로 생성하기
위한 필터 역할

$ logstash -e 'input { stdin {}} output {stdout {} }' 
// -e 옵션을 이용하면 설정 파일을 읽지 않고, 그 다음에 오는 명령을 설정으로 인식
// 위의 명령어는 stdin -> filter 없음 -> stdout으로 동작하는 로그 스태쉬를 실행
// 명령어를 입력 후 hello world를 입력하면 timestamp, host 정보를 붙여서 hello world가 필터 되는것을 확인!

### 설정파일
$ logstash -f {설정파일 명}
// 설정파일을 읽어서 로그스태쉬를 실행시킴
// 설정 파일은 input, filter, output 으로 구성

**1. Input 설정**

ex) input 설정 예제
`
input {
    file{
        path => "/path/access.log"
        start_position => "beginning"
        ignore_older => 0
    }
}
`

- path => 입력 파일이 위치한 곳
- start_position => beginning : 로그 스태쉬는 기본적으로 unix시스템 tail -f 커맨드와 동일하게 동작
즉, access.log 파일의 증가분에 대해서만 input으로 인식 
따라서, 증가분이 아니라 항상 파일의 처음부터 input으로 인식하게 하기 위해 start_position=>beginning으로 추가
- ignore_older => 0 : 로그 스태쉬는 기본적으로 파일이 하루 이상 오래된 경우 input으로 인식하지 않는다. 이러한 동작을 멈추기 위해 사용

**2, Filter 설정**

- Filter Plugin: Grok
- 로그 스태쉬에서 기본적으로 사용할수 있는 필터 플러그인이다.
- 불규칙적인 데이터를 정형화된 데이터로 만들어줌
- 어떤 패턴의 로그 데이터가 처리 대상인지 패턴 정의를 통해 지정
- 해당 패턴이 만족되면 만족된 데이터를 정형화된 데이터로 변환



참고 : http://blog.naver.com/PostView.nhn?blogId=kbh3983&logNo=221063092376












