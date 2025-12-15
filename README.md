# uv-aws-lambda-example

An example project for using uv with [AWS Lambda](https://aws.amazon.com/lambda/), with a focus on
best practices for developing with the project mounted in the local image.

See the [uv AWS Lambda integration guide](https://docs.astral.sh/uv/guides/integration/aws-lambda/)
for more background.

## Project overview

### Dockerfile

The [`Dockerfile`](./Dockerfile) defines the image and includes:

- Installation of uv
- Installing the project dependencies and the project separately for optimal image build caching
- Bundling the project into the AWS Lambda target directory
- Defining the AWS Lambda entrypoint

### Dockerignore file

The [`.dockerignore`](./.dockerignore) file includes an entry for the `.venv` directory to ensure
the `.venv` is not included in image builds. Note that the `.dockerignore` file is not applied to
volume mounts during container runs.

### Application code

The Python application code for the project is at [`app/main.py`](./app/main.py).

### Project definition

The project at [`pyproject.toml`](./pyproject.toml) includes FastAPI as a dependency along with
Magnum, which manages the AWS Lambda integration.

## Useful commands

To run the application locally:

```console
$ uv lock
$ uv run fastapi dev
```

### Docker

To deploy as a Docker image, build the image and push it to AWS Elastic Container Registry (ECR).

To build the image for AWS Lambda:

```console
$ uv lock
$ docker build -t fastapi-app .
```

To push the image to AWS [Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/)

```console
$ aws ecr get-login-password --region region | docker login --username AWS --password-stdin aws_account_id.dkr.ecr.region.amazonaws.com
$ docker tag fastapi-app:latest aws_account_id.dkr.ecr.region.amazonaws.com/fastapi-app:latest
$ docker push aws_account_id.dkr.ecr.region.amazonaws.com/fastapi-app:latest
```

To deploy the image to AWS Lambda:

```console
$ aws lambda create-function \
   --function-name myFunction \
   --package-type Image \
   --code ImageUri=aws_account_id.dkr.ecr.region.amazonaws.com/fastapi-app:latest \
   --role arn:aws:iam::111122223333:role/my-lambda-role
```

Where the [execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html#permissions-executionrole-api)
is created via:

```console
$ aws iam create-role \
   --role-name my-lambda-role \
   --assume-role-policy-document '{"Version": "2012-10-17","Statement": [{ "Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]}'
```

To update the Lambda function:

```console
$ aws lambda update-function-code \
   --function-name myFunction \
   --image-uri aws_account_id.dkr.ecr.region.amazonaws.com/fastapi-app:latest \
   --publish
```

### Zip archive

To deploy as a zip archive, create a zip archive and upload it to AWS Lambda.

To create a zip archive:

```console
$ uv export --frozen --no-dev --no-editable -o requirements.txt
$ uv pip install \
   --no-installer-metadata \
   --no-compile-bytecode \
   --python-platform x86_64-manylinux2014 \
   --python 3.13 \
   --target packages \
   -r requirements.txt
$ cd packages
$ zip -r ../package.zip .
$ cd ..
$ zip -r package.zip app
```

To deploy the zip archive to AWS Lambda:

```console
$ aws lambda create-function \
   --function-name myFunction \
   --runtime python3.13 \
   --zip-file fileb://package.zip \
   --handler app.main.handler \
   --role arn:aws:iam::111122223333:role/service-role/my-lambda-role
```

Where the [execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html#permissions-executionrole-api)
is created via:

```console
$ aws iam create-role \
   --role-name my-lambda-role \
   --assume-role-policy-document '{"Version": "2012-10-17","Statement": [{ "Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]}'
```

To update the Lambda function:

```console
$ aws lambda update-function-code \
   --function-name myFunction \
   --zip-file fileb://package.zip
```
