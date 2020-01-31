# AWS Elastic Beanstalk

Elastic Beanstalk은 AWS에 어플리케이션을 배포하는 가장 간편하고 빠른 방법으로 Java, .NET, PHP, Node.js, Python, Ruby, Go, Docker를 사용하여 Apache, Nginx, IIS와 같은 서버에서 구동되는 웹서비스들을 지원합니다. 사용자를 대신해서 인프라를 프로비저닝해주고 업데이트 및 패치도 간단하게 적용할수 있습니다. 인프라 운영 오버헤드등을 줄여서 개발에 더 집중할수 있도록 도와주는 PaaS 입니다.

## 시작하기전에

1. 본 Hands-on lab에서 사용할 Application 예제는 MDN (the Mozilla Developer Network) 에서 만든 [튜토리얼](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Django/Tutorial_local_library_website)의 예제입니다.
2. 본 Hands-on lab은 AWS Seoul region 기준으로 작성되었습니다. Region을 Seoul (ap-northeast-2)로 변경 후 진행 부탁드립니다.
3. [AWS Credit 추가하기](https://aws.amazon.com/ko/premiumsupport/knowledge-center/add-aws-promotional-code/)
4. [Lab 환경 구축](https://ap-northeast-2.console.aws.amazon.com/cloudformation/home?region=ap-northeast-2#/stacks/quickcreate?templateURL=https://saltware-aws-lab.s3.ap-northeast-2.amazonaws.com/eb/eb.yml&stackName=eb-lab)

## Elastic Beanstalk 환경 구성

1. AWS Management Console에서 좌측 상단에 있는 [Services] 를 선택하고 검색창에서 Elastic Beanstalk을 검색하거나 [Compute] 밑에 있는 [Elastic Beanstalk] 를 선택

2. **[Create New Application]** &rightarrow; **Application Name** = local-library &rightarrow; **[Create]**

3. Elastic Beanstalk Application Dashboard 에서 **[Actions]** &rightarrow; **[Create environment]** &rightarrow; **Web server environment** :radio_button: &rightarrow; **[Select]**

4. **Environment name** = local-library-<YOUR_INITIAL>,\
**Domain** = local-library-<YOUR_INITIAL>,\
**Platform** = :radio_button: Preconfigured platform - Python,\
**Application code** = :radio_button: Sample application,\
&rightarrow; **[Configure more options]**

5. **Configuration presets** = :radio_button: High availability

6. **[Capacity]** &rightarrow; **[Modify]** &rightarrow; **Instance type** = m5.large, **Metric** = CPUUtilization, **Unit** = Percent, **Upper threshold** = 80, **Lower threshold** = 30 &rightarrow; **[Save]**

7. **[Security]** &rightarrow; **[Modify]** &rightarrow; **EC2 key pair** = EC2 인스턴스에 접속할 Key pair 지정, **IAM instance profile** = eb-lab-InstanceProfile-XXXXX &rightarrow; **[Save]**

8. **[Network]** &rightarrow; **[Modify]** &rightarrow; **VPC** = eb-vpc, **Load balancer visibility** = Public, **Load balancer subnets** = 모든 public subnet 선택, **Instance subnets** = 모든 private subnet 선택 &rightarrow; **[Save]**

9. **[Create environment]**

## Application 배포

1. AWS Management Console에서 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 Cloud9를 검색하거나 **[Developer Tools]** 밑에 있는 **[Cloud9]** 를 선택 &rightarrow; **[Open IDE]**

2. 해당 [Git Repository](https://github.com/aws/aws-elastic-beanstalk-cli-setup)를 참고해서 EB CLI 설치

3. 해당 [Git Repository](https://github.com/mdn/django-locallibrary-tutorial)를 Fork (GitHub 계정 필수)

4. Forking한 Repository를 Cloud9으로 환경으로 Clone

   ```bash
   git clone http://<REPOSITORY_URL>
   ```

5. 위에서 Clone한 로컬 Git Repository의 Root 디렉토리로 이동 후 EB CLI 설정

   ```bash
   eb init
   ```

   ```bash
   Select a default region: 10) ap-northeast-2 : Asia Pacific (Seoul)
   Select an application to use: 1) local-library
   Do you wish to continue with CodeCommit? (y/N): N
   ```

6. EB CLI로 Application 배포

   ```bash
   eb deploy
   ```

7. Elastic Beanstalk 도메인 주소로 접속해서 어플리케이션이 정상적으로 작동하는지 확인. 에러발생시 아래의 커맨드로 Log를 확인

   ```bash
   eb logs -a
   ```

## Configuration & Customization

### WSGI 설정

1. EB Dashboard에서 **[Configuration]** &rightarrow; Category가 **Software** 인 탭에서  **[Modify]** 클릭 &rightarrow; **WSGIPath** = locallibrary/wsgi.py &rightarrow; **[Apply]**

### ALLOWED_HOSTS 설정

1. `locallibrary/settings.py` 파일을 열고 아래와 같이 수정

   ```python
   ALLOWED_HOSTS = ['.elasticbeanstalk.com']
   ```

2. EB CLI로 Application 배포 후 변경사항이 적용됬는지 확인

3. 수정한 코드를 Commit & Push

   ```bash
   git add .
   git commit -m "fixed DisallowedHost error"
   git push
   ```

4. EB CLI로 Application 배포 후 변경사항이 적용됬는지 확인

### Custom script 실행

1. ebextensions 폴더 생성

   ```bash
   mkdir .ebextensions
   ```

2. configuration 파일 셍성

   ```bash
   touch .ebextensions/05_django.config
   ```

3. `.ebextensions/05_django.config` 파일을 열고 아래의 코드블록 붙여넣기

    ```yaml
    container_commands:
      01_collect_static:
        command: python manage.py collectstatic
      02_migrate:
        command: python manage.py migrate --noinput
        leader_only: true
    ```

### Database 연결

1. PostgreSQL client 설치

   ```bash
   sudo yum install postgresql -y
   ```

2. RDS 인스턴스에 접속 (RDS Endpoint는 RDS Console에서 확인 가능)

   ```bash
   psql -h eb-postgres.xxxxxx.ap-northeast-2.rds.amazonaws.com -U master postgres
   ```

3. 접속이 안될 경우 RDS 인스턴스에 붙어있는 Security Group에 Cloud9 인스턴스에서 PostgreSQL 포트로 접속 가능한 Inbound rule을 설정

4. 아래와 같이 비밀번호 입력창이 나올경우, `asdf1234`를 입력

   ```bash
   Password for user master:
   ```

5. Database 생성

    ```sql
    CREATE DATABASE local_library;
    ```

6. Database User 생성

    ```sql
    CREATE USER local_library WITH PASSWORD 'asdf1234';
    ```

7. Database User에 권한 부여

    ```sql
    GRANT ALL PRIVILEGES ON DATABASE local_library TO local_library;
    ```

8. `locallibrary/settings.py` 파일을 열고 아래와 같이 수정

    ```python
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': 'local_library',
            'USER': 'local_library',
            'PASSWORD': 'asdf1234',
            'HOST': '<RDS_ENDPOINT>',
            'PORT': '5432',
        }
    }
    ```

9. `locallibrary/settings.py` 파일을 열고 아래와 같은 코드블록을 삭제

    ```python
    # Heroku: Update database configuration from $DATABASE_URL.
    import dj_database_url
    db_from_env = dj_database_url.config(conn_max_age=500)
    DATABASES['default'].update(db_from_env)
    ```

10. 수정한 코드를 Commit & Push하고 EB CLI를 통해서 Application 배포

## CodePipeline 구축

1. AWS Management Console에서 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 CodePipeline를 검색하거나 **[Developer Tools]** 밑에 있는 **[CodePipeline]** 를 선택

2. **[Create pipeline]** &rightarrow; **Pipeline name** = eb, **Service role** = New service role &rightarrow; **[Next]** &rightarrow; **Source provider** = GitHub &rightarrow; **[Connect to GitHub]** &rightarrow; **Repository** = 랩 시작할때 Forking한 Repository, **Branch** = master &rightarrow; **[Skip build stage]**

3. **Deploy provider** = AWS Elastic Beanstalk, **Application name**과 **Environment name**에 위에서 생성한 리소스들을 넣고 **[Next]** &rightarrow; **[Create pipeline]**

## Autoscaling 설정

### ELB Health check 설정

1. `locallibrary/settings.py` 파일을 열고 아래와 같은 코드블록을 붙여넣기

    ```python
    # Get the private IP address of the instance to stop Sentry errors from
    # Load Balancer health checks.
    import requests

    PRIVATE_IP_URL = 'http://169.254.169.254/latest/meta-data/local-ipv4'
    PRIVATE_IP_REQUEST_DATA = requests.get(PRIVATE_IP_URL)
    if PRIVATE_IP_REQUEST_DATA:
        IP_ADDRESS = PRIVATE_IP_REQUEST_DATA.text

        ALLOWED_HOSTS.append(IP_ADDRESS)
    ```

2. `requirements.txt`파일을 열고 아래의 라인을 추가

    ```python
    requests==2.22.0
    ```

3. 수정한 코드를 Commit & Push

4. Load balancer 설정에서 `Health check path`를 `/catalog/`로 변경하고 적용

5. Health Status가 :white_check_mark: `OK` 로 변했는지 확인

### Autoscaling 테스트

1. AWS Management Console 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 ssm를 검색하고 **[Systems Manager]** 를 선택
2. Systems Manager Dashboard 왼쪽 패널 Instances & Nodes 섹션 아래에 있는 **[Session Manager]** 선택
3. **[Start Session]** &rightarrow; Instance Name: **local-library-xxx** 선택 &rightarrow; **[Start Session]** 클릭

4. Root 환경으로 전환

    ```bash
    sudo -i
    ```

5. Stress utility 설치

    ```bash
    yum install stress -y
    ```

6. CPU load 생성

    ```bash
    stress --cpu 4 --timeout 600
    ```

7. 신규 인스턴스가 생성되는지 확인

### Auto-healing 테스트

1. **Session Manager**를 통해서 EC2 인스턴스에 접속

2. Apache HTTP Server 정지

    ```bash
    sudo service httpd stop
    ```

3. 인스턴스가 자동으로 복구되는지 확인 후 Cloud9 IDE 실행

4. configuration 파일 셍성

   ```bash
   touch .ebextensions/03_autoscaling.config
   ```

5. `.ebextensions/03_autoscaling.config` 파일을 열고 아래의 코드블록 붙여넣기

    ```yaml
    Resources:
      AWSEBAutoScalingGroup:
        Type: "AWS::AutoScaling::AutoScalingGroup"
        Properties:
          HealthCheckType: ELB
          HealthCheckGracePeriod: 300
    ```

6. 수정한 코드를 Commit & Push

7. 다시 EC2 인스턴스에 접속해서 Apache HTTP Server 정지하고 기존 인스턴스가 새로운 인스턴스로 교체되는지 확인

## SSM Parameter Store 설정

### KMS 암호화 키 생성

1. AWS Management Console 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 kms를 검색하고 **[Key Management Service]** 를 선택

2. **[Customer managed keys]** &rightarrow; **[Create key]** &rightarrow; **Key type** = Symmetric &rightarrow; **[Next]** &rightarrow; **Alias** = eb &rightarrow; **[Next]** &rightarrow; **Key administrators** = 해당 암호화키에 관리자 권한을 줄 IAM 유저 선택 &rightarrow; **[Next]** &rightarrow; **This account** 탭에서 *eb-lab-IAMRole-xxxx*를 선택 &rightarrow; **[Next]** &rightarrow; **[Finish]**

### SSM Parameter 생성

1. AWS Management Console 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 ssm를 검색하고 **[Systems Manager]** 를 선택

2. Systems Manager Dashboard 왼쪽 패널 Application Management 섹션 아래에 있는 **[Parameter Store]** 선택

3. **[Create parameter]** &rightarrow; **Name** = /LOCAL_LIBRARY/DB_PASSWORD, **Tier** = Standard, **Type** = SecureString, **KMS Key ID** = alias/eb, **Value** = asdf1234 &rightarrow; **[Create parameter]**

### IAM Role 권한 설정

1. AWS Management Console에서 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 IAM를 검색하거나 **[Security, Identity, & Compliance]** 바로 밑에 있는 **[IAM]** 를 선택

2. **[Roles]** &rightarrow; *eb-lab-IAMRole-xxxx*를 선택 

3. **[Add inline policy]** 선택 후, **Service** = Systems Manager, **Actions** = GetParameter, **Resources** = :white_check_mark: Specific &rightarrow; **[Add ARN]**, **Region** = ap-northeast-2, **Fully qualified parameter name** = LOCAL_LIBRARY/DB_PASSWORD &rightarrow; **[Add]**, **[Review policy]** 클릭, **Name** = ssm_get_param, **[Create policy]** 클릭

### Application 설정 파일 수정

1. `locallibrary/settings.py` 파일을 열고 `DATABASES` 블록 위에 아래의 코드블록을 붙여넣기

    ```python
    # SSM
    import boto3
    ## Create the SSM Client
    ssm = boto3.client(
        'ssm',
        region_name='ap-northeast-2'
    )

    ## Get the requested parameter
    response = ssm.get_parameter(
        Name='/LOCAL_LIBRARY/DB_PASSWORD',
        WithDecryption=True
    )

    DB_PASSWORD = response['Parameter']['Value']
    ```

2. `locallibrary/settings.py` 파일을 열고 `DATABASES` 블록을 아래와 같이 수정

    ```python
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql_psycopg2',
            'NAME': 'local_library',
            'USER': 'local_library',
            'PASSWORD': DB_PASSWORD,
            'HOST': '<RDS_ENDPOINT>',
            'PORT': '5432',
        }
    }
    ```

3. `requirements.txt`파일을 열고 아래의 라인을 추가

    ```python
    boto3==1.11.9
    ```

4. 수정한 코드를 Commit & Push