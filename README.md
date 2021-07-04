# このリポジトリ
[コンテナ利用者に捧げる AWS Lambda の新しい開発方式 !](https://aws.amazon.com/jp/builders-flash/202103/new-lambda-container-development/?awsf.filter-name=*all)
に従い、コンテナ Lambda を試したもの。

# ローカル確認
```
docker-compose up -d

curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{}'
```

# ECRにhello_worldをデプロイ
## imageビルド
```
docker-compose build
```

### ECRリポジトリ作成(なければ)
```
aws ecr create-repository --repository-name hello_world 
```

### リポジトリの URI を含めたタグを付与
```
docker tag hello_world:latest ${ACCOUNTID}.dkr.ecr.ap-northeast-1.amazonaws.com/hello_world:latest
```

### ECR ログイン
```
aws ecr get-login-password | docker login --username AWS --password-stdin ${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com
```

### リポジトリに push 
```
docker push ${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com/hello_world:latest
```
push が完了すると、ダイジェストが発行されるので、これを DIGEST 環境変数に入れる
```
DIGEST=$(aws ecr list-images --repository-name hello_world --out text --query 'imageIds[?imageTag==`latest`].imageDigest')
```

### ラムダロール作成(なければ)
```
aws iam create-role --role-name lambda-ex \
--assume-role-policy-document '{"Version": "2012-10-17","Statement": [{ "Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]}'
```

### ラムダ関数作成
```
ROLE_ARN=arn:aws:iam::${ACCOUNTID}:role/lambda-ex

aws lambda create-function \
     --function-name hello_world  \
     --package-type Image \
     --code ImageUri=${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com/hello_world@${DIGEST} \
     --role ${ROLE_ARN}
```