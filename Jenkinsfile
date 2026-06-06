// CI de Nível 01 - Disparo Manual através de clique

pipeline {
    agent any

    stages {
        stage('Instalando Yarn') {
            steps {
                bat 'npm install -g yarn'
            }
        }

        stage('Instalando dependências') {
            steps {
                bat 'yarn'
            }
        }

        stage('Instalando Browsers do Playwright') {
            steps {
                bat 'yarn playwright install'
            }
        }

        stage('Executando testes e2e') {
            steps {
                bat 'yarn run e2e'
            }
        }
    }
}
