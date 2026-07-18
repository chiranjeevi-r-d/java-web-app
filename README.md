# java-web-app

A minimal Maven web application used for Jenkins pipeline testing.

## Build

```bash
mvn clean package
```

## Jenkins

The Jenkinsfile uses a shared library pipeline and expects the following Jenkins configuration:
- Jenkins URL: http://localhost:8080/
- Shared library named: payment-shared-lib
- Credentials: git-creds, aws-creds
