## ArgoCD

### Important Points to keep in mind

- **Be sure to annotate your ConfigMap resources using the label app.kubernetes.io/part-of: argocd, otherwise Argo CD will not be able to use them.**
- For Application and AppProject resources, the name of the resource equals the name of the application or project within Argo CD. This also means that application and project names are unique within a given Argo CD installation - you cannot have the same application name for two different applications.
- For `Application` and `Project` the namespace must match the namespace of your Argo CD instance - typically this is argocd.
- When creating an application from a Helm repository, the chart attribute must be specified instead of the path attribute within spec.source.

    ```
    spec:
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo
    ```

- **By default, deleting an application will not perform a cascade delete, which would delete its resources. You must add the finalizer if you want this behaviour - which you may well not want.**
    
    ```
    metadata:
        finalizers:
          - resources-finalizer.argocd.argoproj.io
    ```

- Some Git hosters - notably GitLab and possibly on-premise GitLab instances as well - require you to specify the .git suffix in the repository URL, otherwise they will send a HTTP 301 redirect to the repository URL suffixed with .git. Argo CD will not follow these redirects, so you have to adjust your repository URL to be suffixed with .git.

- **Consider using bitnami-labs/sealed-secrets to store an encrypted secret definition as a Kubernetes manifest. When using bitnami-labs/sealed-secrets the labels will be removed and have to be readded as descibed here: https://github.com/bitnami-labs/sealed-secrets#sealedsecrets-as-templates-for-secrets**

#### App of Apps

You can create an app that creates other apps, which in turn can create other apps. This allows you to declaratively manage a group of apps that can be deployed and configured in concert.

See [cluster bootstrapping](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/).

#### Projects

Projects¶

The AppProject CRD is the Kubernetes resource object representing a logical grouping of applications. It is defined by the following key pieces of information:

  - sourceRepos reference to the repositories that applications within the project can pull manifests from.
  - destinations reference to clusters and namespaces that applications within the project can deploy into (don't use the name field, only the server field is matched).
  - roles list of entities with definitions of their access to resources within the project.

    > If a Project's destinations configuration allows deploying to the namespace in which Argo CD is installed, then Applications under that project have admin-level access. RBAC access to admin-level Projects should be carefully restricted, and push access to allowed sourceRepos should be limited to only admins.

#### Applications

The Application CRD is the Kubernetes resource object representing a deployed application instance in an environment. It is defined by two key pieces of information:

  - source reference to the desired state in Git (repository, revision, path, environment)
  - destination reference to the target cluster and namespace. For the cluster one of server or name can be used, but not both (which will result in an error). Under the hood when the server is missing, it is calculated based on the name and used for any operations.

See [application.yaml](https://argo-cd.readthedocs.io/en/stable/operator-manual/application.yaml) for additional fields.

#### Repositories

Repository details are stored in secrets. To configure a repo, create a secret which contains repository details.

Each repository must have a url field and, depending on whether you connect using HTTPS, SSH, or GitHub App, username and password (for HTTPS), sshPrivateKey (for SSH), or githubAppPrivateKey (for GitHub App).

Example for HTTPS:

```
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/argoproj/private-repo
  password: my-password
  username: my-username
```

Example for SSH:

```
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:argoproj/my-private-repository
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

Example for GitHub App:
    
```
    apiVersion: v1
    kind: Secret
    metadata:
    name: github-repo
    namespace: argocd
    labels:
        argocd.argoproj.io/secret-type: repository
    stringData:
    type: git
    url: https://github.com/argoproj/my-private-repository
    githubAppID: 1
    githubAppInstallationID: 2
    githubAppPrivateKey: |
        -----BEGIN OPENSSH PRIVATE KEY-----
        ...
        -----END OPENSSH PRIVATE KEY-----
    ---
    
    apiVersion: v1
    kind: Secret
    metadata:
    name: github-enterprise-repo
    namespace: argocd
    labels:
        argocd.argoproj.io/secret-type: repository
    stringData:
    type: git
    url: https://ghe.example.com/argoproj/my-private-repository
    githubAppID: 1
    githubAppInstallationID: 2
    githubAppEnterpriseBaseUrl: https://ghe.example.com/api/v3
    githubAppPrivateKey: |
        -----BEGIN OPENSSH PRIVATE KEY-----
        ...
        -----END OPENSSH PRIVATE KEY-----
```

#### Repository Credentials¶

If you want to use the same credentials for multiple repositories, you can configure credential templates. Credential templates can carry the same credentials information as repositories.