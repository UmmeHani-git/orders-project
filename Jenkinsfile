pipeline {
  agent any

  environment {
    REGION = "us-east-1"
  }

  stages {

      stage('Checkout Code') {
      steps {
        // Pull latest code from GitHub repo
        git branch: 'main', url: 'https://github.com/UmmeHani-git/orders-project.git'
        
      }
    }


    stage('Version') {
      steps {
        script {
          env.VERSION = sh(
            script: "bash scripts/version.sh",
            returnStdout: true
          ).trim()
        }
      }
    }

    stage('Zip Lambdas') {
      steps {
        sh '''
        chmod +x scripts/zip.sh
        ./scripts/zip.sh
        '''
      }
    }

    stage('Terraform') {
  steps {
    sh '''
    cd terraform
    terraform init
    terraform plan
    '''
  }
}

    stage('Update Lambdas') {
      steps {
        sh '''
        aws lambda update-function-code --function-name orders-producer --zip-file fileb://producer.zip
        aws lambda update-function-code --function-name orders-consumer --zip-file fileb://consumer.zip
        aws lambda update-function-code --function-name orders-status --zip-file fileb://status.zip
        '''
      }
    }

    stage('Build & Push Frontend') {
      steps {
        sh '''
        ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
        ECR="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/orders-frontend"

        aws ecr get-login-password --region $REGION \
        | docker login --username AWS --password-stdin $ECR

        docker build -t orders-frontend ./frontend

        docker tag orders-frontend:latest $ECR:latest
        docker tag orders-frontend:latest $ECR:$VERSION

        docker push $ECR:latest
        docker push $ECR:$VERSION
        '''
      }
    }

  stage('Run Frontend') {
  steps {
    sh '''
    echo "Stopping any container using port 80..."

    CONTAINER=$(docker ps -q --filter "publish=80")

    if [ ! -z "$CONTAINER" ]; then
      docker stop $CONTAINER
      docker rm $CONTAINER
    fi

    echo "Removing old frontend container if exists..."

    docker rm -f frontend 2>/dev/null || true

    echo "Starting new frontend container..."

    docker run -d -p 80:80 --name frontend orders-frontend

    echo "Frontend deployed successfully 🚀"
    '''
  }
}

  }
}
