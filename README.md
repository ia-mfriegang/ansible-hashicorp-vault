# Ansible Hashicorp Vault Role
### Requirements

- Minimum 3 Ubuntu 20.04/22.04 servers

- Valid certificates for each Vault node, this should have the node's IP address as a subject alternative name, as well as the DNS name for the load balanced IP on each cert.

- Familiarity with Linux and Ansible.

- DNS records for each Vault node, this dns name should match the common name on the certs. 

 - OPTIONAL - AWS KMS key for automatic unsealing. This isn't required, but is preferable for production environment. This needs an associated IAM policy and IAM user. 
 
 ---
### Preparation
1. Prepare a variables file for Ansible(this is under roles/defaults/main.yml) Adjust values as necessary.
2. Create a host vars file for each node with the following content
3. Create your AWS KMS Key
    a. Log into the AWS console, and head to the KMS menu
    b. Create a new key, following the default options. You shouldn’t need to attach any policies to it yet.
    c. Copy down the arn number for later.
4. Create an IAM policy for the key
 ``` {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt",
                "kms:Encrypt",
                "kms:DescribeKey"
            ],
            "Resource": "<key arn>"
        }
    ]
}
```
5. Create a user with the policy you created in the last step attached, and the permission boundary set to the same policy as well.
    a. Copy down the secret key and access key somewhere secure such as a password vault. Encrypt them with ansible-vault, and add them to the variables file from step 1.
6. Launch the Ansible Playbook on all 3 of the Vault nodes.

--- 

### Steps after running the playbook
The Ansible playbook did most of the heavy lifting, but there are a few more manual things we need to do in order to get the cluster ready to go.
1. SSH into each Vault node, its easiest to have a separate shell open side by side for each node
2. Copy the cert and private key onto each server, they should be put under `/opt/vault/tls/tls.key` and `/opt/vault/tls/tls.key`, if you're using your own Root CA, add that as well `/opt/vault/tls/ca.pem`
3. Start the vault service on each of the nodes with `sudo systemctl start vault`
4. Check the status of the vault
    a. From any of the nodes, set the vault address variable `export VAULT_ADDR=https://<vault node's DNS name>:8200` The vault shgould be in a sealed and uninitialized state.
    b. Use the command `vault operator init` to initialize the vault. This will output the Vault's recovery keys, save these somewhere secure. NOTE: For production systems, its more secure to do this step with PGP encryption, see [this article](https://developer.hashicorp.com/vault/docs/concepts/pgp-gpg-keybase)
5. Enable apparmor from each node with `sudo aa-enforce /usr/bin/vault`
6. Make sure the keepalived service is running one each node `sudo systemctl status vault`
7. Assuming everything was successful, your load balanced IP/DNS name should now be working.

---
### Next Steps
- Disable root token and enable an alternative auth method and admin policy
- Make sure to follow the [Vault Hardening Guide](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening), this is especially important on production systems, but any Vault with sensitive credentials should be fully hardened. This will require making some changes beyond the scope of this guide, but you should now have a good foundation to build on.
- Configure audit logging
