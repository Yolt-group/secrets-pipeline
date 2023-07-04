# secrets-pipeline

Project for generating and importing application secrets.

## FAQ

- Is Kustomize supported as well?

  Yes, the secrets pipeline looks at the project to determine how to generate the secrets.

- I did not get my secret for environment X on namespace Y??

  Did you add the correct `namespaces` to the definition. You can also check [environments.txt](https://git.yolt.io/security/secrets-pipeline-docker/-/blob/master/src/main/resources/environments.txt) to see which namespaces are supported for a particular environment.

  Furthermore, the target environment must have a `kustomization.yaml` on its `master` branch in the appropriate `k8s/env` folder.
    
- Is it necessary to add all secrets for all environments at once?
  
  No, you can first generate secrets for one environment and then later add definitions for the other environments. It is important to notice that your code should be resilient to this (so secret will only be available at a specific environment)


- What is the field rotate been used for?

  This field should be set to `true` if you want to replace a secret on a specific environment in the future. If set to `false` you will not be able to overwrite it. If you imported a secret from a third party `rotate` should always be set to `true`.


- Can we rotate a generated secret every now and then?

  Sure, just make a MR with the `rotate` flag set to `true`, make a MR merge it and change the setting back again.
  

- We want to encrypt files with the secrets-pipeline is that possible?

  Yes, with some manual work you can encrypt the file locally with Vault and then import it:
  1. Encrypt your values locally with <https://git.yolt.io/security/secrets-pipeline#import-existing-secrets>
  2. Create an MR in the secrets pipeline, with the outcome of step 1 (importedSecret)
  3. Secrets pipeline will create an MR in the specified project (note: this can be any project as it is just an MR you can close the MR once you collected the values)
  4. Update the K8s deployment file with the following annotations:

    ```hcl
    vault.hashicorp.com/agent-inject-template-secret-file: |
      {{- with secret "transit/git/decrypt/team1-<<namespace>>-<<service_name>>" "ciphertext=vault:v1:<<paste here>>" "context=eW9sdC1naXQtc3RvcmFnZQo=" -}}
      {{ .Data.plaintext }}
      {{- end -}}
    ```

    You need to fill in the correct environment secret from `/secrets/yolt_secrets.yaml`
  
  Start a deployment and vault-agent will through the Vault transit secret engine put the file in `/vault/secrets` of the pod.
  
- Troubleshooting imported secrets
  
  If for some reason the imported secrets does not get picked up during the deployment, you can check the deployment with the following command:
  
  ```shell
  kubectl edit deployment <<deployment>>
  ```
  
  Look for the key and the template, for example:
  
  ```hcl
  vault.hashicorp.com/agent-inject-template-pgp-key: |
    {{- with secret "transit/git/decrypt/team8-default-reconciliation" "ciphertext=vault:v1....
    type: GPG_PAIR
    {{ .Data.plaintext }}  <---- private key
    ----------
    LS0tLS1....            <---- public key
  ```
  
  Both should be base64 encoded ONCE. So when you encrypt the secret: do not double encode, as the vault-helper already adds a layer of base-64 encoding.
  If you log in to the pod with `kubectl exec ...` you can also check out the decrypted secret and if you base-64 decode it, it should be the same as the imported secret.

## How to use it

You can either import an existing secret, or let the pipeline generate a secret. The sections below show how to use it:

### Import existing secrets

When you want to import an existing secret, use the following steps:

1. Encrypt the secret with vault-helper. You need at least version v0.11.14 of [Vault-helper](https://git.yolt.io/infra/vault-helper). Secrets require to be base-64 encoded, so you either let vault-helper do the job (-b64 flag) or you encode it yourself.

    - for production, consider that for sensitive secrets the security team should do this for you: login with the first command and then execute the second one to encrypt the secret:

        ```sh
            ./vault-helper login
        ```

        ```sh
            ./vault-helper encrypt -key prd-provisioner -b64 -- "$(cat <file-containing-secret>)"
        ```

    - for DTA:

         ```sh
            ./vault-helper encrypt -key dta-provisioner -b64 -- "$(cat <file-containing-secret>)"
        ```

    The `-b64 --` is to stop the flag parser otherwise if the secrets starts with `--` it will be seen as a flag.

    Note on the alternative encodings:  `./vault-helper encrypt -key dta-provisioner -b64 $(cat <file-containing-secret>)` is the same as ` echo -n secretvalue | base64 | ./vault-helper encrypt -key dta-provisioner `.

2. Follow the process below and add in step 2 a json field with the name `imported-secret` with the result of the command above. Note in case of a keypair: only encrypt the private key and just copy the public key as the `public-key-or-cert` field in step 2. Last, make sure that you fill in the command on how you obtained the secret in the `metaData` next to the reason of why you need it in order to make it traceable how you got the secret. Please make sure that you do _*NOT*_ share the secret itself when you show the command.

### Generate new secrets

1. Create a json file in the folder for which you want to generate/import your secrets (So /stubs/< yourfilename >.json if you want to update the stubs service, or `/<name-of-thepod>/<your-secrets-file>.json` for any other arbitrary service).
2. Add your secret definition metadata to the json file (note, use an array):

    ```json
    [
        {
        "name": "Stubs example key",
        "secretType": "KEY_128",
        "description": "Example key for stubs",
        "gitlabRepoNamespace": "backend",
        "environments": [
            "team5",
            "yfb-acc"
        ],
        "rotate": false,
        "jsonVersion": "v1",
        },
        ...
    ]

    ```

    Note: a full list of fields can be found [here](https://git.yolt.io/security/secrets-pipeline-docker/-/blob/master/src/main/java/com/yolt/secretspipeline/secrets/SecretDefinition.java) and a full list of secretTypes can be found [here](https://git.yolt.io/security/secrets-pipeline-docker/-/blob/master/src/main/java/com/yolt/secretspipeline/secrets/SecretType.java)

    Note 2: for PGP, you do need to fill in the pgpTemplate, check [this class](https://git.yolt.io/security/secrets-pipeline-docker/-/blob/master/src/main/java/com/yolt/secretspipeline/secrets/templates/PGPTemplate.java).

    Note 3: for JWKS, you do need to fill in the jwksTemplate, check [this class](https://git.yolt.io/security/secrets-pipeline-docker/-/blob/master/src/main/java/com/yolt/secretspipeline/secrets/templates/JWKSTemplate.java).

    Note 4: for CSR, you do need to fill in the csrTemplate, check [this class](https://git.yolt.io/security/secrets-pipeline-docker/-/blob/master/src/main/java/com/yolt/secretspipeline/secrets/templates/CSRTemplate.java). Please make sure the CN and SAN fields also include information regarding the target environment; we want to be able to discriminate dta and prd certificates.
 
    Note 5: for imported secrets, please define in `metaData` how you got them (e.g. command how you created the imported secret without leaking the value of the secret ;-))

    Note 6: for DTA and PRD you need to add separate secret definitions. For some secrets one definition would have worked but for imported and shared secrets this is not possible. To make the rule more simpler we opted for specifying two separate definitions. 

3. Create a merge-request and inform security at [#yolt-security](https://lovebirdteam.slack.com/archives/CKLQ6C7BN).
4. The security team will review the secret definitions with you and approve & merge when agreed.
5. The pipeline will create MRs for the services for which you defined your secrets in step 1 and 2.
6. Merge the MRs provided by the secrets-pipeline at your targetted service or adapt them further to your needs.

Note: please always check the output of the pipeline at step 3, as you might have json issues, or no secret defined or other side-effects.

Note-2: if the pipeline crashes during step 3 as it does [here](https://git.yolt.io/security/secrets-pipeline/-/jobs/3644309), rerun the pipeline, because apparently Gitlab is unaware of the MR while the pipeline started.

## How to consume the secret

The secrets are consumed by the following steps:

1. The Yolt Deployment tools pick up the secret-definitions in the `/secrets/yolt_secrets.yml` file and extends the K8s deployment file with extra annotations. The secrets will be provided in `/vault/secrets`.
2. In your project use `YoltSecretsAutoConfiguration` for automatically providing the secrets.


## Limitations

Currently, there is no support for multi-project secrets: if you need multiple projects to consume the same secret, then generate the secret offline and feed it to the pipeline as a secret that needs to be imported.

## Instructions for reviewers

If you are reviewing a secret request for the first time (as SRE/Security), make sure that:

1. Only json is added: the gitlab-ci.yml should not be changed. If it is, find out the actual deltas and make sure there is no security downgrade anywhere.
2. The same secret is not imported for both DTA and PRD related environments: separation between test and production is required. Ask the importer whether they used the correct key with vault-helper (dta-provisioner or prd-provisioner)
3. The secret is only imported if:

    - It comes from an external source;
    - It is used to communicate with an external source (e.g. the external source needs it now) in case of a password. In case of a certificate; the key generation can be done by the pipeline, the public key required for the CSR is in the build-log and can then be used to override the certificate until <https://yolt.atlassian.net/browse/STY-401> is implemented.
    - It requires to be the same for multiple different services (so not just different environments, but actually different services) until <https://yolt.atlassian.net/browse/STY-390> is implemented.

5. The name is describing, but does not contain special-chars that are not allowed on a linux FS (note spaces get converted to hyphens).
6. In case of an imported RSA keypair, PGP keypair or a cert+private key, make sure that the public field is filled with b64 encoded content.
7. The secret type is correctly chosen. Make sure that they understand that this has consequences for how the secret can be consumed in their service.
8. In case a password is imported, make sure that you check for `skipPasswordLengthValidation` being true in case the password is too short. Before approving ALLWAYS ask whether they can obtain a longer password before continuing.

Note: make sure that you merge as well: Devops is not allowed to do that.

## Top reasons for pipeline failure

1. It cannot find the branch or MR for which the secrets should be generated: restart the pipeline, as it got started too soon.
2. You are regenerating a key for which the _`rotate`_ boolean is not set to true, therefore: evict it from the json file or set rotate to true and swap the secret.
3. You have errors in your json. Please read the pipeline result carefully.

## Future work

See [Jira](https://yolt.atlassian.net/browse/STY-114) for more information.
