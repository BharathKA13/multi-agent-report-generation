pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        AZURE_CLIENT_ID       = credentials('azure-client-id')
        AZURE_CLIENT_SECRET   = credentials('azure-client-secret')
        AZURE_TENANT_ID       = credentials('azure-tenant-id')
        AZURE_SUBSCRIPTION_ID = credentials('azure-subscription-id')

        ACR_USERNAME = credentials('acr-username')
        ACR_PASSWORD = credentials('acr-password')

        OPENAI_API_KEY = credentials('OPENAI_API_KEY')
        GOOGLE_API_KEY = credentials('GOOGLE_API_KEY')
        GROQ_API_KEY   = credentials('GROQ_API_KEY')
        TAVILY_API_KEY = credentials('TAVILY_API_KEY')
        LLM_PROVIDER   = credentials('LLM_PROVIDER')

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
                checkout scm
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

        stage('Deploy Application') {
            steps {
                sh '''
                    echo "🚀 Starting deployment..."

                    # --- KEEPALIVE (prevents Jenkins stream timeout) ---
                    while true; do
                      echo "⏳ Deployment in progress..."
                      sleep 15
                    done &
                    KEEPALIVE_PID=$!

                    # --- Resolve latest image ---
                    IMAGE_TAG=$(az acr repository show-tags \
                      --name $ACR_NAME \
                      --repository $IMAGE_NAME \
                      --orderby time_desc \
                      --output tsv | head -n 1)

                    if [ -z "$IMAGE_TAG" ]; then
                      echo "❌ No image found in ACR"
                      kill $KEEPALIVE_PID
                      exit 1
                    fi

                    echo "Using image tag: $IMAGE_TAG"

                    # --- Create or Update Container App ---
                    if az containerapp show --name $APP_NAME --resource-group $APP_RESOURCE_GROUP > /dev/null 2>&1; then
                        az containerapp update \
                          --name $APP_NAME \
                          --resource-group $APP_RESOURCE_GROUP \
                          --image ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:$IMAGE_TAG
                    else
                        az containerapp create \
                          --name $APP_NAME \
                          --resource-group $APP_RESOURCE_GROUP \
                          --environment $CONTAINER_ENV \
                          --image ${ACR_NAME}.azurecr.io/${IMAGE_NAME}:$IMAGE_TAG \
                          --registry-server ${ACR_NAME}.azurecr.io \
                          --registry-username $ACR_USERNAME \
                          --registry-password $ACR_PASSWORD \
                          --target-port 8000 \
                          --ingress external \
                          --min-replicas 1 \
                          --max-replicas 3 \
                          --cpu 1.0 \
                          --memory 2.0Gi \
                          --env-vars LLM_PROVIDER=$LLM_PROVIDER
                    fi

                    # --- Set Secrets ---
                    az containerapp secret set \
                      --name $APP_NAME \
                      --resource-group $APP_RESOURCE_GROUP \
                      --secrets "openai-api-key=$OPENAI_API_KEY google-api-key=$GOOGLE_API_KEY groq-api-key=$GROQ_API_KEY tavily-api-key=$TAVILY_API_KEY"

                    # --- Bind Secrets as Env Vars ---
                    az containerapp update \
                      --name $APP_NAME \
                      --resource-group $APP_RESOURCE_GROUP \
                      --set-env-vars \
                        OPENAI_API_KEY=secretref:openai-api-key \
                        GOOGLE_API_KEY=secretref:google-api-key \
                        GROQ_API_KEY=secretref:groq-api-key \
                        TAVILY_API_KEY=secretref:tavily-api-key \
                        LLM_PROVIDER=$LLM_PROVIDER

                    # --- Stop keepalive ---
                    kill $KEEPALIVE_PID

                    # --- Print App URL ---
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
