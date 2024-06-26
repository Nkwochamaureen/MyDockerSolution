# MyDockerSolutions
# Dockerized Solutions Challenge

This is a solution to the Dockerized Solutions challenge on [Hackattic](https://hackattic.com/challenges/dockerized_solutions).

I first created an SSL signed certificate using a domain name. Then, I mapped out my VM to a domain name using my IP address (of the VM) as an A record and installed Docker.

## Solution Steps

### Step 1: Create and Configure SSL Signed Certificate

1. **Create necessary directories:**
    ```bash
    mkdir -p store/auth store/certs
    ```

2. **Move the created certs to the VM:**
    ```bash
    gcloud compute scp PATH_TO_FOLDER_CONTAINING_CERTS_AND_PRIVATE_KEY/* VM_NAME:HOMEDIR/store/certs/ --zone=$ZONE
    ```

3. **Set up Docker with TLS (rename the certs):**
    ```bash
    sudo mkdir -p /etc/docker/certs.d/$DOMAIN_NAME:443
    sudo cp HOMEDIR/store/certs/domain.crt /etc/docker/certs.d/$DOMAIN_NAME:443/
    sudo cp HOMEDIR/store/certs/domain.cert /etc/docker/certs.d/$DOMAIN_NAME:443/
    sudo cp HOMEDIR/store/certs/domain.key /etc/docker/certs.d/$DOMAIN_NAME:443/
    sudo cp HOMEDIR/store/certs/SubCA.crt /etc/docker/certs.d/$DOMAIN_NAME:443/
    sudo cp HOMEDIR/store/certs/Root_RSA_CA.crt /etc/docker/certs.d/$DOMAIN_NAME:443/
    sudo systemctl restart docker
    ```

### Step 2: Create `credentials_json.sh` File

1. **Create `credentials_json.sh` file:**
    ```bash
    nano credentials_json.sh
    ```

2. **Add the following script:**
    ```bash
    #!/bin/bash

    # Define the URL to fetch the JSON from
    URL="https://hackattic.com/challenges/dockerized_solutions/problem?access_token=$ACCESS_TOKEN"

    # Use curl to fetch the JSON data and store it in a variable
    JSON=$(curl -s "$URL")

    # Parse the JSON data using jq and extract the variables
    USER=$(echo "$JSON" | jq -r '.credentials.user')
    PASSWORD=$(echo "$JSON" | jq -r '.credentials.password')
    IGNITION_KEY=$(echo "$JSON" | jq -r '.ignition_key')
    TOKEN=$(echo "$JSON" | jq -r '.trigger_token')

    # Print the export commands
    echo "export USERNAME='$USER'"
    echo "export PASSWORD='$PASSWORD'"
    echo "export IGNITION_KEY='$IGNITION_KEY'"
    echo "export TOKEN='$TOKEN'"
    ```

3. **Execute the script:**
    ```bash
    bash credentials_json.sh > credentials_exports.sh
    source credentials_exports.sh
    ```

4. **Verify the accessibility of variables:**
    ```bash
    echo $USERNAME $PASSWORD $IGNITION_KEY $TOKEN
    ```

### Step 3: Configure Authentication

1. **Configure authentication:**
    ```bash
    htpasswd -Bbn $USERNAME $PASSWORD > auth/htpasswd
    ```

2. **Grant Docker command access without sudo:**
    ```bash
    sudo usermod -aG docker $USER
    ```

### Step 4: Create and Configure the Docker Registry

1. **Create and configure the Docker registry:**
    ```bash
    docker run -d -p 443:443 --name=local-registry --restart=always \
      -v /HOMEDIR/store/certs:/certs \
      -v /HOMEDIR/store/auth:/auth \
      -e "REGISTRY_AUTH=htpasswd" \
      -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
      -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
      -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
      -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.cert \
      -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
      registry:2
    ```

### Step 5: Verification

1. **Authenticate to the Docker registry:**
    ```bash
    docker login $DOMAIN_NAME -u $USERNAME -p $PASSWORD
    ```

2. **Ensure the registry is reachable:**
    ```bash
    curl https://$DOMAIN_NAME:443
    ```

3. **Stop the registry:**
    ```bash
    docker stop $CONTAINER_ID
    ```

4. **Restart the registry:**
    ```bash
    docker start $CONTAINER_ID
    ```

### Step 6: Trigger the Push

1. **Trigger the push:**
    ```bash
    curl -X POST https://hackattic.com/_/push/$TOKEN -d '{"registry_host": "$DOMAIN_NAME"}'
    ```

### Step 7: Retrieve List of Repositories

1. **Retrieve list of repositories:**
    ```bash
    curl -u $USERNAME:$PASSWORD -X GET https://$IP/v2/_catalog
    ```

2. **Retrieve list of tags in the repository:**
    ```bash
    curl -u $USERNAME:$PASSWORD -X GET https://$IP/v2/hack/tags/list
    ```

### Step 8: Pull the Image from the Registry

1. **Pull the image:**
    ```bash
    docker pull $DOMAIN_NAME/IMAGE:TAG
    ```

2. **Run the container:**
    ```bash
    docker run -e IGNITION_KEY=$IGNITION_KEY --name YOUR_CHOSEN_NAME $DOMAIN_NAME/IMAGE:TAG
    ```

### Step 9: Submit the Solution

1. **Create a submission script file:**
    ```bash
    nano submit_file.sh
    ```

2. **Add the following script:**
    ```bash
    #!/bin/bash

    # Extracted secret key from container logs
    SECRET_KEY="SECRET"
    # Endpoint URL
    URL="https://hackattic.com/challenges/dockerized_solutions/solve?access_token=$ACCESS_TOKEN"
    PAYLOAD="{\"secret\":\"$SECRET_KEY\"}"

    # Make the POST request
    curl -X POST -H "Content-Type: application/json" -d "$PAYLOAD" "$URL"
    ```

3. **Execute the script and check if it was successful:**
    ```bash
    bash submit_file.sh > submit_exports.sh
    source submit_exports.sh
    ```

### Additional Steps

1. **Push your work to GitHub:**
    ```bash
    git add .
    git commit -m "Initial commit of dockerized solution script"
    git push -u origin main
    ```
    
    If you encounter any branch issues, follow these steps:
    ```bash
    git branch -m master main
    git push -u origin main
    ```
