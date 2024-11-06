# Ecoguardian

## Informe del Laboratorio: Implementación de Infraestructura en AWS con CloudFormation

Este informe resume los pasos realizados para desplegar una infraestructura en AWS utilizando plantillas de CloudFormation. El objetivo fue crear y configurar diversos servicios de AWS, probar su funcionamiento e identificar y solucionar problemas encontrados durante el proceso.

## Índice
1. Configuración Inicial
2. Despliegue de la VPC
3. Despliegue de Instancias EC2
4. Despliegue del Bucket S3
5. Despliegue de DynamoDB
6. Despliegue de SQS
7. Despliegue de RDS
8. Despliegue de Lambda y SNS
9. Pruebas y Verificación
10. Conclusiones

## 1. Configuración Inicial

**Configuración de Credenciales AWS**: Se configuraron las credenciales de AWS en el archivo `~/.aws/credentials`.

**Verificación de la Identidad del Usuario**:
    ```bash
    aws sts get-caller-identity
    ```

**Configuración del Perfil AWS**: Se editó el archivo `~/.aws/config` para establecer la región y el formato de salida por defecto.

**Importación de la Clave SSH**:
    ```bash
    mv ~/Downloads/my-ec2-keypair.pem ~/.ssh/
    chmod 400 ~/.ssh/my-ec2-keypair.pem
    ```

## 2. Despliegue de la VPC
Se desplegó la VPC utilizando la plantilla `vpc.yaml`:
```bash
aws cloudformation deploy \
  --template-file vpc.yaml \
  --stack-name MyVPCStack \
  --parameter-overrides EnvironmentName=MyEnvironment
```
*Mensaje de confirmación*: `Successfully created/updated stack - MyVPCStack`

## 3. Despliegue de Instancias EC2
Se desplegaron instancias EC2 con la plantilla `ec2.yaml`:
```bash
aws cloudformation deploy \
  --template-file ec2.yaml \
  --stack-name MyEC2Stack \
  --parameter-overrides KeyName=my-ec2-keypair VPCId=[VPC_ID] PublicSubnetId=[PUBLIC_SUBNET_ID] EnvironmentName=MyEnvironment
```

## 4. Error y Solución en Despliegue del Bucket S3
Al desplegar el bucket S3 usando `s3.yaml`, inicialmente el despliegue falló debido a que el nombre contenía mayúsculas. Esto se corrigió usando solo minúsculas en `EnvironmentName`. 

```bash
aws cloudformation deploy \
  --template-file s3.yaml \
  --stack-name MyS3Stack \
  --parameter-overrides EnvironmentName=myenvironment
```

## 5. Despliegue de DynamoDB
Se creó la tabla DynamoDB `sensor_data` exitosamente:
```bash
aws cloudformation deploy \
  --template-file dynamodb.yaml \
  --stack-name MyDynamoDBStack \
  --parameter-overrides EnvironmentName=MyEnvironment
```

## 6. Despliegue de SQS
Se desplegó la cola SQS `MyEnvironment-DataQueue`:
```bash
aws cloudformation deploy \
  --template-file sqs.yaml \
  --stack-name MySQSStack \
  --parameter-overrides EnvironmentName=MyEnvironment
```

## 7. Desafíos con el Despliegue de RDS
Al desplegar RDS, hubo problemas con los parámetros y versiones de base de datos, que se solucionaron ajustando el `DBInstanceClass` y `EngineVersion`.

```bash
aws cloudformation deploy \
  --template-file rds.yaml \
  --stack-name MyRDSStack \
  --parameter-overrides \
    VPCId=VPC_ID_DEL_PASO_1 \
    PrivateSubnetIds=SUBNETS_PRIVADAS_IDS_DEL_PASO_1 \
    EnvironmentName=MyEnvironment \
    DBUsername=admin \
    DBPassword=TuContraseñaSegura
```

## 8. Despliegue de Lambda y SNS
Se desplegó Lambda y SNS con `lambda_sns.yaml` y solucionó errores de dependencias y permisos IAM:
```bash
aws cloudformation deploy \
  --template-file lambda_sns.yaml \
  --stack-name MyLambdaSNSStack \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides EnvironmentName=MyEnvironment SNSNotificationEmail=tu_email@example.com DataQueueArn=arn:aws:sqs:[REGION]:[ACCOUNT_ID]:MyEnvironment-DataQueue LambdaRoleArn=arn:aws:iam::[ACCOUNT_ID]:role/LabRole
```

## 9. Pruebas y Verificación

- **Mensajes a la Cola SQS**:
    ```bash
    aws sqs send-message \
      --queue-url "https://sqs.[REGION].amazonaws.com/[ACCOUNT_ID]/MyEnvironment-DataQueue" \
      --message-body '{"sensor_id": "sensor_1", "timestamp": "2024-10-12T16:00:00Z", "value": 150}'
    ```
- **Verificación en DynamoDB**: Se confirmó el almacenamiento de datos en la tabla `sensor_data`.
- **Notificaciones SNS**: SNS envió alertas por email cuando el valor del sensor superó el umbral definido.
- **Revisión de Logs en CloudWatch**: Los logs confirmaron que la función Lambda se ejecutó correctamente.

## 10. Conclusiones
Se desplegó y configuró exitosamente una infraestructura en AWS, integrando VPC, EC2, S3, DynamoDB, SQS, RDS, Lambda y SNS. 

### Recomendaciones:
1. **Gestión de Nombres de Recursos**: Evitar mayúsculas en nombres de recursos.
2. **Compatibilidad de Versiones**: Verificar compatibilidad de versiones.
3. **Permisos IAM**: Planificar políticas de IAM.
4. **Uso de Parámetros y Outputs**: Utilizar parámetros y outputs en CloudFormation para facilitar la configuración entre stacks.

*Nota*: Detalles sensibles, como identificadores de cuenta y direcciones de correo electrónico, han sido omitidos para proteger la confidencialidad de la información.


