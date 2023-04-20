## **ECS Anywhere**

1. Definir la región:

		$ REGION=<la que se desee>
	
2. Lanzar la plantilla *ecs-anywhere.yaml*:

		$ aws cloudformation deploy --template-file ecs-anywhere.yaml --stack-name ecs-anywhere --region $REGION --capabilities CAPABILITY_IAM
	
3. Se obtiene el rol creado y el clúster. Anotar los valores:

		$ ROL=$(aws cloudformation describe-stacks --stack-name ecs-anywhere --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`Rol`].OutputValue' --output text)
		$ CLUSTER=$(aws cloudformation describe-stacks --stack-name ecs-anywhere --region $REGION --query 'Stacks[0].Outputs[?OutputKey==`Cluster`].OutputValue' --output text)
	
4. Crear mediante AWS SSM una activación híbrida para 4 instancias:

		$ aws ssm create-activation --default-instance-name ecs-anywhere --registration-limit 4 --iam-role $ROL --region $REGION > activacion.json
	
5. Obtener los datos de la activación y anotarlos:

		$ ID_ACTIVACION=$(jq -r .ActivationId activacion.json)
		$ CODIGO_ACTIVACION=$(jq -r .ActivationCode activacion.json)
	
6. Ejecutar las siguientes órdenes y anotar los valores:

		$ echo $ID_ACTIVACION
		$ echo $CODIGO_ACTIVACION
	
7. Aprovisionar VMs con SO Ubuntu 20 (en Proxmox deben ser LXC Privilegiados y habilitar el Nesting)

8. En cada instancia on-premises, descargar el script:

		$ curl -o "ecs-anywhere.sh" "https://amazon-ecs-agent.s3.amazonaws.com/ecs-anywhere-install-latest.sh"
		$ chmod +x ecs-anywhere.sh
	
El script anterior instala los agentes SSM y ECS en la VM, instala Docker y registra la VM en el clúster de ECS. Sustituir en la siguiente instruccion los valores anotados:
	
		$ sudo ./ecs-anywhere.sh --cluster $CLUSTER --activation-id $ID_ACTIVACION --activation-code $CODIGO_ACTIVACION --region $REGION

Puede necesitarse deshabilitar apparmor si no se conecta el agente ECS al cluster

		$ sudo systemctl disable apparmor
		$ sudo apt remove apparmor
		$ sudo reboot

9. Crear un servicio con la tarea anterior

		$ aws ecs create-service --cluster $CLUSTER --service-name test-service --task-definition test-family --desired-count 4 --launch-type EXTERNAL --region $REGION

O bien, lanzar tareas independientemente:

		$ aws ecs run-task --cluster $CLUSTER --launch-type EXTERNAL --count 4 --task-definition test-task --region $REGION
	