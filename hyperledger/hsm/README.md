# hyperledger fabric support hsm
I worked with a repository of fabric-ca and fabric cloned.
- https://github.com/hyperledger/fabric-ca (release-1.4)
- https://github.com/hyperledger/fabric (release-1.4)
ca, baseos :: fabric-ca
peer, orderer, cli :: fabric

# Troubleshooting
## softhsm2
```
<action>
softhsm 다운로드 후 tar 파일 압축 해제 후 "./configure command 실행

<error>
config.status: error: Something went wrong bootstrapping makefile fragments for automatic dependency tracking.  Try re-running configure with the 
'--disable-dependency-tracking' option to at least be able to build the package (albeit without support for automatic dependency tracking).
See `config.log' for more details

<cause>
본인 환경 OpenSSL version : OpenSSL 1.1.0g  2 Nov 2017 (Library: OpenSSL 1.1.1  11 Sep 2018)
OpenSSL 1.1.0 and later no longer include the GOST engine.

<solution>
./configure --disable-gost
```
```
<action>
softhsm 다운로드 후 tar 파일 압축 해제 후 "./configure --disable-gost" command 실행

<error>
1. configure: error: Can't find OpenSSL headers
2. configure: error: in `/root/softhsm-2.5.0':
   configure: error: no acceptable C compiler found in $PATH
   See `config.log' for more details

<cause>
1. libssl-dev 패키지가 설치되있지 않을 경우 발생
2. gcc 패키지 설치되있지 않을 경우 발생

<solution>
1. apt install libssl-dev
2. apt install gcc
```

## fabric-ca
```
<action>
1. docker-compose -f docker-compose-ca.yaml up -d

<error>
1. Error: Failed to initialize BCCSP Factories: Failed initializing PKCS11.BCCSP %!s(<nil>): Could not initialize BCCSP PKCS11 [Failed initializing PKCS11 library /etc/hyperledger/fabric/libsofthsm2.so ForFabric: Could not find token with label ForFabric]
2. Error: Failed to find private key for certificate in '/etc/hyperledger/fabric-ca-server/ca-cert.pem': Could not find matching private key for SKI: Failed getting key for SKI [[154 216 194 32 150 244 187 27 214 47 171 67 55 50 10 217 86 196 215 84 159 52 167 20 213 191 44 106 54 12 84 215]]: Key with SKI 9ad8c22096f4bb1bd62fab4337320ad956c4d7549f34a714d5bf2c6a360c54d7 not found in msp/keystore

<cause>
1. fabric-ca-server 시작 시 fabric-ca-server-config.yaml에 설정한 bccsp값에 해당하는 값을 softhsm에서 찾을 수 없다. fabric-compose-ca.yaml에서 tokens 폴더 volume을 잘못 설정하면 해당 에러 발생
2. fabric-ca-server를 띄우는데 잘못 된 msp가 설정되있다. ca를 띄우는 데 FABRIC_CA_HOME에 msp가 없으면 새로 enroll 받아 진행하지만 msp가 있을 경우 해당 msp로 인증을 받으려한다. 그러나 msp가 잘못되있는 경우 에러가 발생한다.

<solution>
1. fabric-compose-ca.yaml에서 volume 설정을 다시 잡아준다.
2. 경로 /etc/hyperledger/fabric-ca-server/ 에 있는 msp 관련 파일을 전부 삭제 후 재실행
```

## chaincode install
```
<action>
peer chaincode install -n fabcar -p github.com/chaincode/fabcar -v 1.0

<error>
ERRO 014 Cannot run peer because error when setting up MSP of type bccsp from directory /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/peerorg1/users/1zxcvpeeradmin/msp: could not initialize BCCSP Factories: %!s(<nil>)
Could not find default `PKCS11` BCCSP

<cause>
cli container에 pkcs11이 설치 되있지 않음.
fabric 1.4.5 기준으로 make 하여 fabric 이미지 생성하는데 Makefile 및 cli Dockerfile에 pkcs11 세팅하는 부분이 없음.

<Solution>
.build/image/tools/Dockerfile에서 11번째 줄 변경
AS-IS :: RUN make configtxgen configtxlator cryptogen peer discover idemixgen
TO-BE :: RUN make GO_TAGS=pkcs11 configtxgen configtxlator cryptogen peer discover idemixgen
```

## chaincode instantiate
```
<action>
peer chaincode instantiate -o orderer.example.com:7050 -C yongchannel -n fabcar -v 1.0 -c '{"Args":[]}'

<error>
Error: could not assemble transaction, err proposal response was not successful, error code 500, msg error starting container: error starting container: API error (404): network _basic not found

<cause>
peer container 띄울 때 docker-compose.yaml에 설정값(CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE)이 잘못되있었음.

<Solution>
docker-compose.yaml 변경
as is :: CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_basic
to be :: 삭제
```

```
<action>
peer chaincode instantiate -o orderer.example.com:7050 -C yongchannel -n fabcar -v 1.0 -c '{"Args":[]}'

<error>
chaincode log...
fatal error: unexpected signal during runtime execution
[signal SIGSEGV: segmentation violation code=0x1 addr=0x63 pc=0x7f235a424360]

<cause>
해당 에러는 cgo를 사용할 때 나는 에러이다.
chaincode 패키징 중 tar.FileInfoHeader 호출할 때 간접적으로 c 라이브러리를 사용한다.
순수 go를 사용하도록 환경변수 세팅해야한다.
참조 : https://www.alibabacloud.com/blog/hyperledger--fabric--deployment--on--alibaba--cloud--environment--%E2%80%93--sigsegv--problem--analysis--and--solutions_593970?spm=a2c4.11999857.0.0

<solution>
cli docker-compose 파일 환경변수 설정
-> GODEBUG=netdns=go

cli에서 체인코드 띄우는 부분 환경변수 설정 시 argument가 없음 ... 하드코딩 되어있음 ㅠㅠ
-> release-1.4 (1.4.5) : container_runtime.go, chaincode_support.go   
-> master (2.0) : dockercontroller.go getenv 메소드에서 하드코딩 되어있음

.build/image/ccenv/Dockerfile 수정
ENV GODEBUG netdns=go 추가
```