pipeline {
    agent any

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timeout(time: 25, unit: 'MINUTES')
    }

    environment {
        // Azure credentials
        AZURE_CLIENT_ID = credentials('azure-client-id')
        AZURE_CLIENT_SECRET = credentials('azure-client-secret')
        AZURE_TENANT_ID = credentials('azure-tenant-id')
        AZURE_SUBSCRIPTION_ID = credentials('azure-subscription-id')

        // ACR credentials
        ACR_USERNAME = credentials('acr-username')
        ACR_PASSWORD = credentials('acr-password')

        // API keys
        OPENAI_API_KEY = credentials('OPENAI_API_KEY')
        GOOGLE_API_KEY = credentials('GOOGLE_API_KEY')
        GROQ_API_KEY   = credentials('GROQ_API_KEY')
        TAVILY_API_KEY = credentials('TAVILY_API_KEY')
        LLM_PROVIDER   = credentials('LLM_PROVIDER')

        // App config
        APP_RESOURCE_GROUP = 'research-report-app-rg'
        APP_NAME = 'research-report-app'
        ACR_NAME = 'researchreportacr'
        IMAGE_NAME = 'research-report-app'
        CONTAINER_ENV = 'research-report-env'
    }

    stages {

        stage('Checkout') {
            steps {
                cleanWs()
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: 'https://github.com/BharathKA13/multi-agent-report-generation.git']]
                ])
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 --version
                    pip3 install --upgrade pip --break-system-packages
                    pip3 install -r requirements.txt --break-system-packages
                '''
            }
        }

        stage('Quick Sanity Test') {
            steps {
                sh '''
                    python3 -c "from research_and_analyst.api.main import app; print('✅ App import OK')"
                '''
            }
        }

        stage('Login to Azure') {
            steps {
                sh '''
                    az login --service-principal \
                      -u $AZURE_CLIENT_ID \
                      -p $AZURE_CLIENT_SECRET \
                      --tenant $AZURE_TENANT_ID

                    az account set --subscription $AZURE_SUBSCRIPTION_ID
                '''
            }
        }

        stage('Resolve Image Tag') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: """
                            az acr repository show-tags \
                              --name $ACR_NAME \
                              --repository $IMAGE_NAME \
                              --orderby time_desc \
                              --output tsv | head -n 1
                        """,
                        returnStdout: true
                    ).trim()

                    if (!env.IMAGE_TAG) {
                        error "❌ No Docker image found in ACR"
                    }

                    echo "Using image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Azure Container Apps') {
            steps {
                sh '''
                    echo "🚀 Deploying container app..."

                    if az containerapp show --name $APP_NAME --resource-group $APP_RESOURCE_GROUP > /dev/null 2>&1; then
                        echo "Updating existing app..."
                        az containerapp update \
                          --name $APP_NAME \
                          --resource-group $APP_RESOURCE_GROUP \
                          --image ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG} \
                          --verbose
                    else
                        echo "Creating new app..."
                        az containerapp create \
                          --name $APP_NAME \
                          --resource-group $APP_RESOURCE_GROUP \
                          --environment $CONTAINER_ENV \
                          --image ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${IMAGE_TAG} \
                          --registry-server ${ACR_NAME}.azurecr.io \
                          --registry-username $ACR_USERNAME \
                          --registry-password $ACR_PASSWORD \
                          --target-port 8000 \
                          --ingress external \
                          --min-replicas 1 \
                          --max-replicas 3 \
                          --cpu 1.0 \
                          --memory 2.0Gi \
                          --env-vars LLM_PROVIDER=$LLM_PROVIDER \
                          --verbose
                    fi

                    echo "🔐 Setting secrets..."
                    az containerapp secret set \
                      --name $APP_NAME \
                      --resource-group $APP_RESOURCE_GROUP \
                      --secrets "openai-api-key=$OPENAI_API_KEY google-api-key=$GOOGLE_API_KEY groq-api-key=$GROQ_API_KEY tavily-api-key=$TAVILY_API_KEY" \
                      --verbose

                    echo "🔄 Updating env vars..."
                    az containerapp update \
                      --name $APP_NAME \
                      --resource-group $APP_RESOURCE_GROUP \
                      --set-env-vars \
                        OPENAI_API_KEY=secretref:openai-api-key \
                        GOOGLE_API_KEY=secretref:google-api-key \
                        GROQ_API_KEY=secretref:groq-api-key \
                        TAVILY_API_KEY=secretref:tavily-api-key \
                        LLM_PROVIDER=$LLM_PROVIDER \
                      --verbose
                '''
            }
        }

        stage('Print App URL (Non-Blocking)') {
            steps {
                sh '''
                    APP_URL=$(az containerapp show \
                      --name $APP_NAME \
                      --resource-group $APP_RESOURCE_GROUP \
                      --query properties.configuration.ingress.fqdn -o tsv)

                    echo "===================================="
                    echo "✅ DEPLOYMENT SUCCESSFUL"
                    echo "🌐 Application URL:"
                    echo "https://$APP_URL"
                    echo "ℹ️ App may take a few minutes to warm up"
                    echo "===================================="
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
