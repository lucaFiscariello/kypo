## Integrazione Kypo-OpenStack
La seguente guida permette di allocare un'istanza di kypo e connetterla a un'istanza di OpenStack, sia essa multinodo o singolo nodo. 

### Step 0
Installa tutte le librerie necessarie.

```
sudo apt install -y qemu qemu-kvm libvirt-daemon libvirt-clients bridge-utils virt-manager vagrant vagrant-libvirt
sudo snap install core snapd
sudo snap install terraform --classic
sudo snap install kubectl --classic
sudo snap install helm --classic
sudo apt update
sudo apt install -y python3-dev libffi-dev gcc libssl-dev python3-venv python3-docker pipenv jq
sudo apt install --reinstall ca-certificates
apt install python3-openstackclient
```


### Step 1 
Accediamo alla Dashboard grafica di openstack per scaricare un file di configurazione denominato admin-openrc.sh. Questo file contiene i dati necessari per impartire comandi ad Openstack tramite le sue API. 
Questo file contiene informazioni come l'ID del progetto, il nome del progetto, il nome della regione .. etc.

![2024-04-16_10-52](https://github.com/lucaFiscariello/kypo-kali/assets/80633764/618e2f9c-4ded-4c2a-b390-5a470f9f9ac8)

### Step 2
Clona il repository di kypo lite e accedi alla cartella root.

```
git clone https://gitlab.ics.muni.cz/muni-kypo-crp/devops/kypo-lite
cd kypo-lite
```

Copia il file admin-openrc.sh in kypo-lite.

![2024-04-16_10-59](https://github.com/lucaFiscariello/kypo-kali/assets/80633764/76e39f50-6ac5-411e-8deb-39346ad72eba)

Esequi il seguente comando. Ti verrà chiesta di inserire una password. Inserisci la stessa password che usi per il login su Openstack. Questo script salva tutte le informazioni che servono per connettersi ad OpenStack in delle varibili di ambiente, che verranno usate successivamente da altri script.

```
. admin-openrc.sh
```

### Step 3
Crea le credenziali openstack attrverso cui kypo potrà interagire con Openstack.
```
if openstack application credential show demo > /dev/null 2> /dev/null; then
  export OS_APPLICATION_CREDENTIAL_ID=$(openstack application credential show demo -f json | jq '.id' | tr -d '"')
else
  export OS_APPLICATION_CREDENTIAL_ID=$(openstack application credential create --unrestricted --role admin --secret password demo -f json | jq '.id' | tr -d '"')
fi
export OS_AUTH_TYPE=v3applicationcredential
export OS_INTERFACE=public
export OS_APPLICATION_CREDENTIAL_SECRET=password
unset OS_AUTH_PLUGIN OS_ENDPOINT_TYPE OS_PASSWORD OS_PROJECT_DOMAIN_NAME\
 OS_PROJECT_NAME OS_TENANT_NAME OS_USERNAME OS_USER_DOMAIN_NAME
```

### Step 4
Cloniamo il repository contenente il codice che instanzia i server Kypo.

```
git clone -b v3.3.0 https://gitlab.ics.muni.cz/muni-kypo-crp/devops/kypo-crp-tf-deployment.git
cd kypo-crp-tf-deployment/tf-openstack-base
```

Ora modifichiamo il file provider.tf. Questo file di configurazione Terraform descrive il provider, ovvero l'entità predisposta ad allocare le macchine virtuali su cui installare i server kypo. Con il seguente comando stiamo dicendo che il provider è la nostra istanza Openstack ,già  esistente, a cui vogliamo agganciarci. 
Sostituisci al "your-password" la password Openstack che usi per fare il login, e a "your-ip", l'ip che usi per accedere alla dashboard openstack. Il resto del comando non toccarlo, dovrebbe essere corretto con i parametri di default.

```
sed -i 's/provider "openstack" {/provider "openstack" {\n  user_name   = "admin"\n  tenant_name = "admin"\n  password    = "your-password"\n  auth_url    = "http:\/\/your-ip:5000\/v3\/"\n  region      = "RegionOne"\n/g' provider.tf
```

Eseguiamo gli script terraform che creano le macchine virtuali. Tali script creeranno due macchine virtuali, il kypo-proxy-jump e il default-kubernetes-cluster e una rete openstack denominata kypo-base-net.

```
terraform init
export TF_VAR_external_network_name=public
export TF_VAR_dns_nameservers='["1.1.1.1","1.0.0.1"]'
export TF_VAR_standard_small_disk="10"
export TF_VAR_standard_medium_disk="16"
sudo terraform apply -auto-approve -var-file tfvars/vars-all.tfvars
```

Di seguito la schermata della dashboard Openstack della network topology.

![2024-04-16_11-50](https://github.com/lucaFiscariello/kypo-kali/assets/80633764/39d7721d-dfc9-4cff-a4cb-f309f3e23ee4)

### Step 5
Configuriamo l'accesso al cluster kubernetes istanziato all'interno della macchina virtuale "default-kubernetes-cluster". Per farlo basta copiare un file di configurazione prodotto dagli script terraform in una cartella che andremo a creare.

```
cd kypo-lite # vai nella cartella kypo-lite
mkdir -p .kube
sudo cp /kypo-crp-tf-deployment/tf-openstack-base/config .kube/config
```

### Step 6
Prepariamo l'ambiente per installare tutti i micro servizi nel cluster kubernets. 

```
export TF_VAR_acme_contact="demo@kypo.cz"
export TF_VAR_application_credential_id=$OS_APPLICATION_CREDENTIAL_ID
export TF_VAR_application_credential_secret=$OS_APPLICATION_CREDENTIAL_SECRET
export TF_VAR_enable_monitoring=true
export TF_VAR_gen_user_count=10
export TF_VAR_guacamole_admin_password=password
export TF_VAR_guacamole_user_password=password
export TF_VAR_head_host=$(terraform output -raw cluster_ip)
export TF_VAR_head_ip=$(terraform output -raw cluster_ip)
export TF_VAR_kubernetes_host=$(terraform output -raw cluster_ip)
export TF_VAR_kubernetes_client_certificate=$(terraform output -raw kubernetes_client_certificate)
export TF_VAR_kubernetes_client_key=$(terraform output -raw kubernetes_client_key)
export TF_VAR_kypo_crp_head_version="3.1.11"
export TF_VAR_kypo_gen_users_version="2.0.1"
export TF_VAR_kypo_postgres_version="2.1.0"
export TF_VAR_man_flavor="standard.medium"
export TF_VAR_man_image="debian-11-man-preinstalled"
export TF_VAR_openid_configuration_insecure=true
export TF_VAR_os_auth_url=$OS_AUTH_URL
export TF_VAR_os_region="RegionOne"
export TF_VAR_proxy_host=$(terraform output -raw proxy_host)
export TF_VAR_proxy_key=$(terraform output -raw proxy_key)
export TF_VAR_git_config='{type="INTERNAL",server="git-internal.kypo",sshPort="22",restServerUrl="http://git-internal.kypo:5000/",user="git",privateKey="",accessToken="no-gitlab-token",ansibleNetworkingUrl="git@git-internal.kypo:/repos/backend-python/ansible-networking-stage/kypo-ansible-stage-one.git",ansibleNetworkingRev="v1.0.14"}'
export TF_VAR_users='{"kypo-admin"={iss="https://'$TF_VAR_head_host'/keycloak/realms/KYPO",keycloakUsername="kypo-admin",keycloakPassword="password",email="kypo-admin@example.com",fullName="Demo Admin",givenName="Demo",familyName="Admin",admin=true}}'
cd /home/kypo/kypo-lite/kypo-crp-tf-deployment/tf-head-services
sudo sed -i -e "s/1.1.1.1/1.1.1.1/" -e "s/1.0.0.1/1.0.0.1/" values.yaml
```

Nel seguente comando controlla che "--kubeconfig" punti correttamente al file .kube/config creato in precedenza. Inizializziamo cluster.

```
iterations=0
while true; do
    ((iterations++))
    echo "Iteration: $iterations"
    result=$(sudo kubectl get crd --kubeconfig="../.kube/config" | grep middlewares.traefik.containo.us )
    if [ $? -eq 0 ]; then
        echo "K3s cluster ready:"
        echo "$result"
        break
    else
        echo "K3s cluster initializing. Retrying..."
    fi

    sleep 1
done
```

Installiamo tutti i microservizi.

```
sudo terraform init
terraform apply -auto-approve
```

Controlla che l'installazione è andata a buon fine.

```
echo "ALL DONE. Open https://$TF_VAR_head_host/"
echo "Login: kypo-admin"
echo "Password: password"
echo "Monitoring admin password: $(terraform output -raw monitoring_admin_password)"
echo "Keycloak admin password: $(terraform output -raw keycloak_password)"
echo "Import demo-training with URL git@git-internal.kypo:/repos/prototypes-and-examples/sandbox-definitions/kypo-library-demo-training.git"
echo "Import demo-training-adaptive with URL git@git-internal.kypo:/repos/prototypes-and-examples/sandbox-definitions/kypo-library-demo-training-adaptive.git"
echo "Import junior-hacker with URL git@git-internal.kypo:/repos/prototypes-and-examples/sandbox-definitions/kypo-library-junior-hacker.git"
echo "Import junior-hacker-adaptive with URL git@git-internal.kypo:/repos/prototypes-and-examples/sandbox-definitions/kypo-library-junior-hacker-adaptive.git"
echo "Import locust-3302 with URL git@git-internal.kypo:/repos/prototypes-and-examples/sandbox-definitions/kypo-library-locust-3302.git"
echo "Import secret-laboratory with URL git@git-internal.kypo:/repos/prototypes-and-examples/sandbox-definitions/kypo-library-secret-laboratory.git"

```


