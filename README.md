# Petstore Kusk Gateway CICD Example

This repo shows a simple example of how you can automate the kusk gateway configuration using Swaggerhub and pipelines.

This example uses GKE for Kubernetes, Github Action for CI, and SmartBear's Petstore service as the application to be routed to. You can swap any of these out for any alternatives and the concepts will work exactly the same.

## Prerequisites
### SwaggerHub
You need to have a SwaggerHub org and have set up the Github sync integration to your repo.
Our SwaggerHub org is already setup to sync to this repo.

For instructions on how to set this up, check out the docs [here](https://support.smartbear.com/swaggerhub/docs/integrations/github-sync.html)

### Kubernetes
Bring your own cluster.
In this example, I have a pre-provisioned GKE cluster running to which I authenticate in my action.

In the cluster, you need to have Kusk Gateway installed. This example runs using the default configuration.

`helm install kusk-gateway kubeshop/kusk-gateway -n kusk-system --create-namespace`

`helm install kusk-gateway-envoyfleet kubeshop/kusk-gateway-envoyfleet -n kusk-system`

### Application
You need to install an application which you want to use Kusk Gateway to route to.
Here we use the Swagger Petstore application.

`kubectl apply -f app/petstore.yaml`

### CICD Lifecyle
Editing of the API document will take place in SwaggerHub.

Once everyone is happy with the changes, press sync.

Because we have sync integration with Github set up, it will sync the changes with our chosen branch (`swaggerhub`).

Open a Pull Request from `swaggerhub` ➡ `main`. Here you review the changes are normal.

Once the Pull Request is merged, the Github Action will be triggered and does the following:
- Authenticates to the cluster
- Fetches the kgw binary which embeds openapi specs inside our API CRD
- Applies the API CRD to the cluster which in turn will configure the gateway.

### Demonstration Commands
#### Get Gateway External IP
```
export EXTERNAL_IP=$(kubectl get svc kgw-envoy-kusk-gateway-envoyfleet -n kusk-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

#### Curl Petstore API findByStatus endpoint
We configure the petstore api to be available at path `/petstore` acting as a prefix which will then be stripped off before the request is passed to the service.

```
x-kusk:
	upstream:
		...
		rewrite:
			pattern: "^/petstore"
			substitution: ""
	path:
		prefix: /petstore/api/v3
```

```
curl -X 'GET' \
"http://${EXTERNAL_IP}/petstore/api/v3/pet/findByStatus?status=available" \
-H 'accept: application/json'
```

Show the affect of the SwaggerHub sync update by disabling this endpoint, merging the PR and trying to curl it again. Show the API is still live by curling another endpoint.

#### Curl Petstore API endpoint
```
curl -X 'GET' \
"http://${EXTERNAL_IP}/petstore/api/v3/pet/1" \
-H 'accept: application/json'
```