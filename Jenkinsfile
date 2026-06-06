// CI de Nível 01 - Disparo Manual através de clique

pipeline {
    agent any

    stages {
        stage('Instalando Yarn') {
            steps {
                sh 'npm install -g yarn'
            }
        }

        stage('Instalando dependências') {
            steps {
                sh 'yarn'
            }
        }

        stage('Instalando Browsers do Playwright') {
            steps {
                sh 'yarn playwright install'
            }
        }

        stage('Executando testes e2e') {
            steps {
                sh 'yarn run e2e'
            }
        }
    }
}
